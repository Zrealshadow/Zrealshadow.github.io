---
layout: post
title: "Sharded Matrix Multiplication"
author: "Lingze Zeng"
image: dragon.png
tags: [Model Scaling]
---

Cheetsheet for the four cases of sharded matmul.

from *How To Scale Your Model* ([Part 3: Sharded Matmuls](https://jax-ml.github.io/scaling-book/sharding)).

The matmul under analysis is always:

```
A[I, J] · B[J, K] → C[I, K]
```

where **J** is the *contracting* (summed-over) dimension, and **I**, **K** are *non-contracting*.

---

## Notation

| Symbol | Meaning |
|--------|---------|
| `X`, `Y`, `Z` | Named hardware axes of the device **mesh** |
| `A[I, J]` | Fully **replicated** — every device holds the whole array |
| `A[I_X, J]` | The `I` dimension is sharded across mesh axis `X`; `J` is replicated |
| `A[I_XY, J]` | `I` sharded across **both** `X` and `Y` (flattened into one big split) |

**Rule:** a single mesh axis can shard at most one array dimension. `A[I_X, J_X]` is forbidden — once an axis is "spent" on one dimension, it can't shard another.

### Visualizing shardings

The same 2D array `[I, J]` laid out across **4 devices** (a `2×2` mesh with axes `X`, `Y`), under nine different shardings. Each panel shows the four device-local pieces; greyed boxes mean that device holds nothing for that shard. Cell labels are the global `(row, col)` indices.

![Sharding patterns](https://jax-ml.github.io/scaling-book/assets/img/sharding-colored5.png)

**Cost symbols (throughput-bound regime):**

- **V** = bytes of the **full / global** array being moved (e.g. `2·I·J` for `float16 A[I,J]`), *not* the per-shard size. The shard count cancels out of the cost.
- **W** = bidirectional ICI bandwidth.
- **N_axes** = the number of mesh axes gathered/reduced in parallel — not the number of chips per axis.

---

## Communication primitives

| Operation | What it does | Cost |
|-----------|--------------|------|
| **AllGather** | Reassembles a sharded array — every device ends up with the full copy | `V / (W · N_axes)` |
| **ReduceScatter** | Sums partial results across an axis, leaving the output sharded | `V / (W · N_axes)` |
| **AllReduce** | Sums partial results across an axis; every device gets the full sum (= ReduceScatter + AllGather) | `2V / W` |
| **AllToAll** | Reshards: gathers one axis and splits a different one along the same mesh axis | `V / (4W)`  (¼ of AllGather, bidir. ring) |

![prim_allgather](/img/20260629/prim_allgather.svg)



![prim_reducescatter](/img/20260629/prim_reducescatter.svg)



---



## Case 1 — neither input sharded on the contracting dim

**No communication.** The local shards already line up; the output is naturally sharded by whatever axes the inputs used.

```
A[I_X, J] · B[J, K_Y] → C[I_X, K_Y]
```

![matmul_case1](/img/20260629/matmul_case1.svg)

**If you want the output replicated** `C[I, K]`, that is a *separate* post-step: `AllGather_X` then `AllGather_Y` to strip both subscripts. The matmul itself still needs no communication.

**Latency:** `0` communication — no collective is needed (the matmul itself still costs compute). Replicating the output, if wanted, costs `V/W` per gathered axis.

---

## Case 2 — one input sharded on the contracting dim

**AllGather the sharded input** along `J` so the contraction is no longer split, then matmul locally.

```
A[I, J_X] · B[J, K] → C[I, K]
```

![matmul_case2](/img/20260629/matmul_case2.svg)

**Choice of where to gather:** if the output may stay replicated, you can instead multiply shards first → `C[I_X, K]` then AllGather the *result*. Both are `V/W`; gather whichever array (input `A` or output `C`) is smaller.

**Latency:** `V / W`, where `V` = bytes of the gathered array.

---

## Case 3 — both inputs sharded on the contracting dim

Each device can multiply its local shards, but because the contracting dim is split, each device only gets a **partial sum** of the result — it has added up *its* slice of `J`, not all of it. Summing those partial sums across the axis with an **AllReduce** (or a cheaper **ReduceScatter** if you can keep the output sharded, **AllReduce** = **ReduceScatter** + **AllGather** ) gives the final answer.

```
A[I, J_Y] · B[J_Y, K]  →  each device holds a partial sum of C[I, K]
AllReduce over Y       →  C[I, K]   (partial sums added up)
```

**Generalization (your observation):** the *contracting* axis `Y` is the only thing that creates the partial sums. The non-contracting dims are free to be sharded on **other** axes, and they pass straight through untouched:

```
A[I_X, J_Y] · B[J_Y, K]  →  partial sums of C[I_X, K]   →[AllReduce over Y]→   C[I_X, K]
```

![matmul_case3](/img/20260629/matmul_case3.svg)

**Latency:** `2V / W` for the AllReduce, or `V / W` if a `ReduceScatter` (output stays sharded, e.g. `→ C[I_X, K_Y]`) is acceptable. `V` = bytes of the result `C`.

---

## Case 4 — both inputs share a mesh axis on a *non*-contracting dim

```
A[I_X, J] · B[J, K_X]   ✗   would give C[I_X, K_X]
```

**Why it's invalid.** A mesh axis can shard only *one* array dimension, so `C[I_X, K_X]` — both output dims pinned to `X` — isn't a legal layout. Concretely, the chip at `X = x` holds only `A`'s rows for group `x` and `B`'s cols for group `x`, so it can only ever form the **diagonal** block `C[x, x]`. The off-diagonal blocks (`C₀₁`, `C₁₀`) are never produced by *any* chip. (`Y` here just replicates the same broken computation.)

![matmul_case4](/img/20260629/matmul_case4.svg)

**The fix — and why it's not as costly as it sounds.** AllGather *one* input along `X` so that input becomes **replicated across the X chips** — e.g. gather `A` → `A[I, J]` — which frees the axis and reduces the problem to Case 1: `A[I, J] · B[J, K_X] → C[I, K_X]`. This is an ordinary AllGather that *spreads a copy across the axis*; it does **not** pull all the shards onto a single device. So the cost is `≈ V/W` — the same order as Case 2, not catastrophic.

**The real takeaway: avoid this sharding by construction.** The wasted gather only exists because two output dimensions were assigned to the same mesh axis. Put `I` and `K` on *different* axes (or leave one replicated) from the start, and the corrective AllGather disappears entirely.

**Latency:** `V / W` for the corrective AllGather (`V` = bytes of the gathered input) — but ideally `0`, by choosing a sharding that never needs it.

---

## Summary

| Case | Condition | Operation | Latency |
|------|-----------|-----------|---------|
| **1** | Neither input sharded on `J` | none (output naturally sharded) | `0`  (+`V/W` per axis if you replicate) |
| **2** | One input sharded on `J` | **AllGather** that input | `V / W` |
| **3** | Both inputs sharded on `J` (same axis) | **AllReduce** result (or **ReduceScatter**) | `2V / W`  (or `V / W`) |
| **4** | Shared mesh axis on a non-contracting dim | **AllGather** one input | `V / W` |

**Two caveats for every estimate:**

1. These are **bandwidth-bound** costs. For tiny arrays (≲ 45 kB/shard on v5e) you hit the **latency floor** (~1 µs/hop), where cost *does* scale with axis length: `T = max( T_min · ΣX_i / 2 ,  V / (W · N_axes) )`.
2. **V is always the full global array size** (e.g. `2·I·J` for `float16`), not the per-shard size.
