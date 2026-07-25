---
layout: post
title: "Cheetsheet for Model Training Parallism"
author: "Lingze Zeng"
image: dragon.png
tags: [Model Scaling]
---

> Details and performance roorline analysis plz refer to [How to Scale Your Model](https://jax-ml.github.io/scaling-book/training/#tensor-parallelism)

One 2-layer MLP (`X · W_in → H · W_out → Out`) across four sharding schemes. Toy shapes:
`X` is `2×4` — `B = 2` token-rows and `d_model = 4` — with `d_ff = 8`, so `W_in` is `4×8`
and `W_out` is `8×4`.

**The MLP is position-wise.** Every token flows through `W_in`/`W_out` independently —
tokens only ever mix inside attention, never in the MLP. So the batch axis `B` is just a
stack of independent rows, free to be split across chips (DDP, FSDP) or replicated (TP)
without changing the result.

## 1 · DDP — replicate weights, split batch

Every chip holds the **whole model** and runs its own token. The forward pass is entirely local (no communication). The only synchronization is a single **AllReduce of the gradients** in the backward pass. Simple and fast, but memory-heavy: the full model lives on every chip.

![DDP](/img/20260724/parallelism_ddp.png)

------

## 2 · FSDP (ZeRO-3) — split batch **and** weights

Same data-parallel batch split as DDP, but the **weights are sharded** into row-slices. Each layer does an **AllGather** to rebuild the full weight just before the matmul, then frees it; gradients are **ReduceScattered** back into slices. Peak memory holds only one layer's weights instead of the whole model.

![FSDP](/img/20260724/parallelism_fsdp.png)

------

## 3 · Tensor Parallelism — split weights, replicate batch

The **batch is not sharded — it is replicated**: every chip holds all `B` tokens, and they all go through the same weights and the same process (this is why the two input rows in the figure share one coloring instead of being split by chip). What is split is the model: the **weights** (`W_in` by columns, `W_out` by matching rows) and the **activations along `d_model`** (`In[B, D_Y]`). An **AllGather** rebuilds the full embedding before the matmul, and a **ReduceScatter** at the end keeps the output sharded (`Out[B, D_Y]`). This costs no more communication than a plain AllReduce (`AllReduce = ReduceScatter + AllGather`) but cuts stored activation memory to `1/Y` per chip.

![Tensor Parallelism](/img/20260724/parallelism_tp.png)

------

## 4 · Mixed FSDP + TP (2×2 = 4 chips)

Mixed runs FSDP and TP together on a **2×2 mesh** (4 chips): axis **X = FSDP** (data), axis
**Y = TP** (model). With batch = 2, **token 0 lives on the X0 track and token 1 on the X1
track**; within a track the two Y-chips split the model. Everything is *doubly-sharded*: the
input is `In[B_X, D_Y]` — **rows = tokens split on X, columns = `d_model` split on Y** — and
the weights are quadrants, `W_in[D_X, F_Y]` and `W_out[F_Y, D_X]`.

 ![Mixed FSDP + TP](/img/20260724/parallelism_mixed.png)

**Starting layout — each chip holds its token, half the embedding:**
 
- `(X0,Y0)`: token 0, `d_model[0:2]` &nbsp; · &nbsp; `(X0,Y1)`: token 0, `d_model[2:4]`
- `(X1,Y0)`: token 1, `d_model[0:2]` &nbsp; · &nbsp; `(X1,Y1)`: token 1, `d_model[2:4]`
**Forward pass — four collectives  around two matmuls:**
 
1. **AllGather Y (activations)** — complete each token's `d_model`. Within a track the two
   Y-chips swap embedding halves, so both hold the full token (`X full`, `1×4`, replicated
   across Y).
2. **AllGather X (`W_in`)** — rebuild the `d_model` rows of `W_in` → `W_in[d_model, F_Y]`
   (replicated across X, still sharded on Y).
3. **matmul 1**: `X full · W_in` → `H[B_X, F_Y]` (hidden, sharded on `d_ff` by Y). No comms —
   the contracted `d_model` is fully present.
4. **AllGather X (`W_out`)** — rebuild the `d_model` columns of `W_out` → `W_out[F_Y, d_model]`.
5. **matmul 2**: `H · W_out` → a partial `Out[B_X, d_model]` (each Y-chip summed only over
   its half of `d_ff`).
6. **ReduceScatter Y (output)** — add the two partials and re-shard along `d_model`,
   giving `Out[B_X, D_Y]` — back to the same doubly-sharded layout the next block expects.
So the forward pass is exactly **one AllGather-Y** (activations), **two AllGather-X** (the two
weights), and **one ReduceScatter-Y** (output).
 
**Two views running at once.** Along **X it is data parallel** — the X0 and X1 tracks run
*identical-weight* computations on *different tokens* (steps 2 and 3 reconstruct the same
weight on both X-chips), and they sync only in the backward pass, via a **ReduceScatter-X**
on the gradients. Along **Y it is tensor parallel** — same token, weights split, output
reduced. The two axes are **orthogonal**: X-gathers move *weights*, Y-gathers/scatters move
*activations*, so they never interfere.

------

**In one line:** DDP splits the *batch* · FSDP also slices *weights by row* (gather to use) · TP slices *weights by column/row* and replicates the batch (reduce the output) · Mixed does both at once on a 2-D mesh — AllGather-X for weights, AllReduce-Y for the output.
