---
layout: post
title: "Deep Retrieval in Bytedance RecSys"
author: "Lingze Zeng"
image: dr_header.png
tags: [RecSys, Paper]
---
*Learning a retrievable structure end-to-end for large-scale recommendation in Bytedance*
Paper: [arXiv:2007.07203](https://arxiv.org/pdf/2007.07203)

---

## 1. Motivation

Classic recall uses **two-tower + ANN**: user and item are each embedded into a vector, similarity is an inner product, and an ANN index approximates top-K search over hundreds of millions of items.

Two problems with this:

- **Two-stage mismatch.** The model learns inner products; the ANN index is built *afterwards* to approximate that model. The index is not optimized directly against user–item interaction data.
- **Inner-product ceiling.** Squeezing every interaction into a single dot product limits expressiveness.

DR takes a different route: **learn the retrievable structure itself, end-to-end from click data.** Each item is encoded onto a discrete **path**, and retrieval walks that structure instead of scanning vectors.

**Overview.** Using the paper's example (`K=100, D=3`): the structure is `D` columns of `K` nodes, giving `K^D ≈ 1M` possible paths. A user walks one MLP-softmax per layer to pick a path, and each path is a **cluster holding many items**.

![overview](/img/20260708/dr_algorithm_overview.svg)
---

## 2. Algorithm Design

### a. The structure: paths

A path is a sequence of node choices across `D` columns, each column having `K` nodes (e.g. `K=100, D=3`). There are `K^D` possible paths, each acting as an **item cluster**. Many items hang on one path — DR is built for *retrieval*, not fine-grained ranking, so items sharing a path need not be distinguished.

Key efficiency: parameters are only `Θ(K·D²)`, yet they express `K^D` clusters.

### b. Assigning a user a path (per-layer prediction)

Given a user `x`, DR predicts the path **layer by layer**. Each layer has its own MLP whose softmax outputs a distribution over the `K` nodes. Crucially, each layer's input is the **user vector plus the path embeddings of nodes already chosen** — this makes the layers interdependent.

The figure below shows one concrete path to assign a user a path.

![user](/img/20260708/user.png)

- Input: `emb(x)` is the user-side input to every layer, in this paper is a **GRU over the user's behaviro sequnece**. The GRU encoding is trained jointly with DR network.
- MLP: one per layer, softmax over `K` nodes.
- Label: the item's path supplies the target of each layer's softmax. (item never enters the network as input)

**Path probability** for a user is the product of per-layer softmax probabilities:
$$
p(c|x) = \prod p(c_d | x, c_{<d})
$$
At serving time there is no label: the user walks the network and **beam search** keeps the top-b paths per layer, yielding the most likely paths for that user.

### c. Assigning an item a path (the mapping π, learned via EM)

The mapping `π: item → path(s)` is **discrete and non-differentiable**, so it cannot be learned by gradient descent. DR uses **EM-style alternation**:

- **E-step (fix π, train the network).** Use each item's current path as the label; users walk the network; standard softmax loss + backprop updates the MLPs and the path embeddings.
- **M-step (fix the network, re-assign π).** No gradients — just score and pick:

  For item `i`, take the **set of users who clicked it**. Each such user walks the network and produces `p(c | x)` for a candidate path `c`. Score the path by summing over those users, then subtract a **load-balancing penalty** (paths already holding many items are penalized, so popular items don't all collapse onto a few paths):
  
  


  $$\text{score}(i, c) = \sum_{x \in X_i} p(c \mid x) \;-\; \underbrace{\text{penalty}\big(\#\{\text{items on } c\}\big)}_{\text{load balancing}}, \qquad X_i = \{\,x : x \text{ clicked } i\,\}$$


  Sort by score, keep the **top-J** paths as item `i`'s assignment.

**Note:** the item never runs a forward pass. It only provides a *label* (E-step) and a *set of fans* (M-step). Users do the walking; the item is positioned by the users who like it.

**Multi-path (`J`, default 3):** an item may occupy several paths to cover multiple semantic facets (e.g. an action-romance film sits in both an action cluster and a romance cluster).

*Parameters to keep separate:* `D` = path length, `K` = nodes per layer, `J` = paths per item.

### d. Serving (path → item)

The learned π is stored as an **inverted index** (`path → item set`). At recall time: user → beam search → top-b paths → collect all items on those paths → hand the candidate set to the ranking model. Retrieval cost is largely decoupled from the total item count. And user path is determined by **beam search** to recall possible items.



### e. Online Learning / Training

**E-step run online, M-step runs periodically, to keep freshness of structure**:

New clicks keep updating the network by SGD. Reassigned `π` sums over all of an item's users, so we only refresh it on a schedule.

**New users are free**:

A user get it's `emb(x)` based on historical behavior sequence. A unseen user just gets encoded and walked through the way.

**New items need one incremental M-step**:

Score the new item against existing paths, keep its top-J. Cold start is the only catch.

---

## 3. Advantages

- **End-to-end retrievable structure** — the index *is* the model; no separate "learn model, then build ANN" stage with mismatched objectives.
- **Low complexity at scale** — `Θ(K·D²)` parameters express `K^D` clusters; beam search + inverted index scale naturally to hundreds of millions of items.
- **Multi-facet coverage** — `J` paths per item capture multiple semantic meanings that a single vector would average away.
- **More interpretable** — discrete, hierarchical paths are easier to inspect and debug than an opaque dense vector space. Analyst can manually assign topic for those paths based on the semantics.
