---
layout: post
title: "Model FLOPs and Memory Estimation"
author: "Lingze Zeng"
image: dragon.png
tags: [Model Scaling]
---

*Cheetsheet for Model FLOPs and Memory estimation in **training** and **inference** mode.*

The single mental shift from **inference** to **training**: 

- inference frees activations (X) immediately; 
- training must keep activations (X) alive for the backward pass, and  additionally carry gradients and optimizer state in GPU memory.

---

## 1. Setup: a Transformer feed-forward block

Built around a Transformer-style feed-forward (FFN) block. A feed-forward block is two stacked linear layers (biases dropped for simplicity). The input and output both live in the model's residual width `D`, with a wider hidden dimension `F` in between — exactly the $W_\text{in}$ / $W_\text{out}$ pair from the scaling chapter:

$$H = X W_\text{in} \qquad\qquad Y = H W_\text{out}$$

| Tensor | Role | Shape |
|---|---|---|
| `X` | block input (activations) | `[B, D]` |
| `W_in` | up-projection | `[D, F]` |
| `H` | hidden activation (= layer-1 output = layer-2 input) | `[B, F]` |
| `W_out` | down-projection | `[F, D]` |
| `Y` | block output (same width as input) | `[B, D]` |

`B` is the batch (number of tokens), `D = d_model`, `F = d_ff` (typically `F ≈ 4D`). Because input and output share the width `D`, blocks like this stack cleanly into a deep residual network.

 This is the inference computation; training adds a **backward pass** and **optimizer state** on top.

> A real FFN has a nonlinearity between the two matmuls (`H = σ(X W_in)`). It's an elementwise op — cheap in FLOPs, and in the backward pass it just multiplies the incoming gradient by `σ'`. We omit it here so the matrix algebra stays clean.

---

## 2. Forward pass

Two matmuls, one per layer:

$$H_{bj} = \sum_i X_{bi} (W_\text{in})_{ij} \qquad Y_{bk} = \sum_j H_{bj} (W_\text{out})_{jk}$$

**FLOPs.** Each output element is a dot product (one multiply + one add ≈ `2` ops per term):

$$\text{Forward FLOPs} \approx \underbrace{2 \cdot B \cdot D \cdot F}_{\text{up-proj}} + \underbrace{2 \cdot B \cdot F \cdot D}_{\text{down-proj}} = 4BDF$$

The block has `P = DF + FD = 2DF` parameters, so forward `= 4BDF = 2BP` 

**<2 FLOPs per parameter per token>**. 

**The key fork — inference vs training.** 

- In pure inference you compute `Y`, pass it on, and **throw `X` and `H` away**. 
- In training you **cannot discard them**, because the backward pass needs them (see below).

---

## 3. Matrix-calculus background

The loss `L` is a scalar at the end. Backprop hands each layer the gradient of the loss w.r.t. its *output*, with the **same shape as that output**. Each layer must produce:

- **`dW = ∂L/∂W`** — to update that layer's weight.
- **`dX = ∂L/∂X`** — the gradient w.r.t. its *input*, to hand to the previous layer so the chain rule can continue.

Take a generic single matmul $Y = X W$. From the scalar chain rule, tracking indices ($Y_{bj} = Σ_i X_{bi} W_{ij}$):

$$\frac{\partial L}{\partial W_{ij}} = \sum_b \frac{\partial L}{\partial Y_{bj}} \frac{\partial Y_{bj}}{\partial W_{ij}} = \sum_b X_{bi}\, dY_{bj} = (X^\top dY)_{ij}$$

$$\frac{\partial L}{\partial X_{bi}} = \sum_j \frac{\partial L}{\partial Y_{bj}} \frac{\partial Y_{bj}}{\partial X_{bi}} = \sum_j dY_{bj}\, W_{ij} = (dY\, W^\top)_{bi}$$

So the two backward rules (can be used directly) for *any* linear layer are:

$$\boxed{dW = X^\top\, dY} \qquad\qquad \boxed{dX = dY\, W^\top}$$

**Two intuitions worth memorizing:**

1. **Shape-matching gives the answer.** `dW` must match `W`, and `dX` must match `X`. Given the tensors on hand (`X`, `W`, `dY`), there's essentially only one way to transpose-and-multiply them into the right shape.
2. **To get a weight's gradient, contract the two tensors around it.** `dW = Xᵀ dY` sums over the batch dimension `B`: every token contributes to every weight's gradient. That sum over `B` is exactly why, in data parallelism, gradients must be **AllReduced** across chips (each chip saw different tokens).

---

## 4. Backward pass (applying the rules to the block)

Start from `dY = ∂L/∂Y` and apply the two rules above to each layer, walking backward. 

**Down-projection** (inputs `H`, `W_out`; incoming grad `dY`):
$$dW_\text{out} = H^\top\, dY \qquad\qquad dH = dY\, W_\text{out}^\top$$

**Up-projection** (inputs `X`, `W_in`; incoming grad is `dH`):
$$dW_\text{in} = X^\top\, dH \qquad\qquad dX = dH\, W_\text{in}^\top$$

The chain is the whole point:

$$dY \;\xrightarrow{\text{down-proj}}\; dH \;\xrightarrow{\text{up-proj}}\; dX$$

Each layer uses the incoming gradient to compute *its own* weight gradient, and passes a gradient further back via `dX = dY Wᵀ`. Here `dH` is **consumed** (the bridge between the two layers); the final `dX` (shape `[B, D]`, same as `X`) is what flows into the previous block in a deep network.

**FLOPs.** Each layer's backward is two matmuls (`dW` and `dX`), each the same size as that layer's forward matmul:

| Quantity | Formula | FLOPs |
|---|---|---|
| `dW_out` | `Hᵀ dY` | `2BDF` |
| `dH`     | `dY W_outᵀ` | `2BDF` |
| `dW_in`  | `Xᵀ dH` | `2BDF` |
| `dX`     | `dH W_inᵀ` | `2BDF` |

So backward `= 8BDF` = **2× forward** ≈ `4BP`. Combined:

$$\text{Training} \approx \underbrace{4BDF}_{\text{fwd}} + \underbrace{8BDF}_{\text{bwd}} = 12BDF = 6BP$$

This is the **"6ND" rule**: training a model with `N` parameters on `D` tokens costs roughly `6ND` FLOPs. 

**Crucial consequence for memory.** `dW_out = Hᵀ dY` needs `H`, and `dW_in = Xᵀ dH` needs `X`. So **both** activations must survive from the forward pass until the backward pass reaches them. Generalize to `L` layers and you retain one activation tensor *per layer* —which is exactly why activation memory scales with batch `B` **and** depth `L`, and why gradient checkpointing (recomputing `H` instead of storing it) is the standard escape hatch.

---

## 5. What lives in GPU memory in training

For the block (`P = 2DF` parameters):

| Item | Size | Bytes | Scales with |
|---|---|---|---|
| Params `W_in,W_out` | `2DF` | 2 (bf16) | model |
| Grads | `2DF` | 2–4 | model |
| Adam `m` | `2DF` | 4 (fp32) | model |
| Adam `v` | `2DF` | 4 (fp32) | model |
| **Activations `X,H`** | `B·(D+F)` | 2 (bf16) | **batch B** |

- Parameters, Gradients, Optimizers depend only on the **model architecture**, never on batch size.
- Activations scale with **`B`**. Stack `L` layers and a batch of millions of tokens, and this row blows up to terabytes while the other rows stay in the tens of GB.

---

## 6. Summary

For a feed-forward block with `D = d_model`, `F = d_ff`, `P = 2DF`, batch `B`:

- **Forward**  `H=XW_in`, `Y=HW_out` · `2BP` FLOPs · retain `X, H` 
-  **Backward**  `dW=XᵀdY`, `dX=dYWᵀ` · `4BP` FLOPs · chain `dY→dH→dX` 
- **FLOPs**  ≈ `6 · params · tokens` = `12BDF`/block 
- **Memory**  `8DF ` (fixed) + `2·B·(D+F)` activations (dominant) 

Everything in large-scale training why data parallelism shards activations; why FSDP also shards the fixed model block; why gradient checkpointing exists: falls out of these two facts:

-  keep activations alive for backward
- carry gradients + optimizer state.
