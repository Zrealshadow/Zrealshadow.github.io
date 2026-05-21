---
layout: post
title: "A Row-Level Perspective on Feature Store"
author: "Lingze Zeng"
tags: [Feature Store, IDEA]
---

inspired by [Paper](https://www.vldb.org/pvldb/vol17/p563-wooders.pdf) (RALF) published in VLDB 2023

## Overview

RALF proposes an accuracy-aware scheduling policy for feature stores: when featurization is expensive and updates can't keep up, prioritize updating the keys whose staleness hurts downstream prediction accuracy the most.

But the more interesting contribution — and the focus of this note — is the **perspective shift** the paper implicitly introduces. RALF treats the feature store as a *big table* where maintenance happens at the **row level (per-entity)**, in contrast to the **column-level (per-feature batch)** maintenance that dominates today's systems like Feast, Hopsworks, Tecton, and Databricks Feature Store.

---

## Two Views of Feature Store Maintenance

A feature store is fundamentally a large table: rows = entities, columns = features.
![img](/img/20260521/column_vs_row_maintenance.svg)
### Column-Level Maintenance (Mainstream Today)

Most production feature stores treat maintenance as a **per-column batch job**:

* `user_click_sum_7d` → SQL aggregation `GROUP BY user_id` over the click stream
* `merchant_avg_order_30d` → rolling window over orders
* Each refresh updates **the entire column for all entities at once**

This fits relational aggregations beautifully. The unit of work is a whole feature pipeline; scheduling means deciding *when* to rerun each pipeline. Systems like **FeathrPO** optimize this dimension — point-in-time joins, layout selection, materialized view reuse.

### Row-Level Maintenance (RALF's Contribution)

RALF flips the axis. The unit of work becomes **one entity's value for one feature**, computed independently:

* `user_embedding[user_42]` → run the encoder on user_42's recent activity
* `vm_stl_decomp[vm_17]` → fit STL on vm_17's CPU history
* Each refresh updates **one cell**, and cells can be prioritized independently

This fits **model-derived per-entity features** — embeddings, time-series fits, learned representations. The featurization is expensive (one model call per entity) and naturally parallel across keys.

---

## RALF: Accuracy-Aware Row-Level Scheduling

### Motivation

If you accept the row-level view, an immediate question follows: **with millions of rows and a fixed compute budget, which cells do you refresh first?**

Existing systems answer this with **Round-Robin** or **FIFO** — process updates in arrival order, treat all keys symmetrically. This wastes budget badly. In production:

* Some keys are queried far more than others (Zipfian access patterns)
* Some keys' features drift fast (active users, bursty workloads); others barely change (dormant entities)
* The *value of updating a cell* varies by orders of magnitude across rows

RALF's insight: **let the downstream model tell you which cells matter**. The model's prediction errors carry exactly the signal needed — if a feature is stale enough to hurt accuracy, the predictions made with it will be wrong, and that wrongness can be fed back to the scheduler.

### Workload Example: Anomaly Detection on Cloud VMs



Modeled on Splunk's production system: a cloud platform monitors 275,077 VMs, each emitting a CPU reading every 5 minutes. For each VM, the feature is an **STL decomposition** (trend + daily seasonality) fitted over the last 72 hours of readings, used downstream to predict expected CPU values and flag anomalies. Each VM is one row in the feature table; its STL fit is the feature value.

![img](/img/20260521/stl_workload_row_level.svg)

**Why it's row-level**: each fit depends only on that VM's own history — vm_002's STL is independent of any other VM. Cells can be updated in any order.

**Why scheduling matters**: each fit costs ~0.3 seconds. Refreshing all 275K rows continuously would need ~22 hours of CPU per round, so only a subset can be updated each interval.

**Why row priorities differ**: every VM is queried equally, but staleness sensitivity is wildly non-uniform. Stable VMs (vm_001, vm_003, vm_005) produce accurate predictions even with old features; drifting VMs (vm_002, vm_004) need fresh features to avoid large errors. Column-style "refresh everyone" wastes budget on the stable majority; RALF routes it to the drifting rows.

This pattern recurs whenever features are learned representations of per-entity history — user embeddings, document encodings, per-merchant fraud profiles.


---

## The Trend: Row-Level Will Matter More

Features are shifting from **statistical** (counts, averages, ratios) to **semantic** (vector embeddings produced by neural models). Statistical features are cheap and batch well — column-level maintenance handles them fine. Semantic features are one expensive model call per entity.

**Concrete example**: "For each user, summarize their last 30 days of support interactions with GPT-4 and store the embedding." Each update costs a real API call (~$0.01–$0.10) and hits hard rate limits. Refreshing 10M users hourly is economically infeasible — *and* wasteful, since most users haven't interacted.

This is exactly RALF's target: expensive, per-entity, independent, with non-uniform update value. As LLM-derived features become standard, row-level prioritization moves from edge case to first-class concern.

---

## Vision: A Hybrid Feature Store

Can a single feature store natively support both column-level and row-level maintenance? This is the feature-store analog of the **OLTP vs OLAP** split — same logical data, fundamentally different access patterns and optimization techniques. HTAP systems work hard to bridge them under one roof; feature stores face the same challenge.

|                  | Column-level                             | Row-level                          |
| ---------------- | ---------------------------------------- | ---------------------------------- |
| **Unit of work** | Whole feature column                     | Single (entity, feature) cell      |
| **Cost driver**  | Data volume scanned                      | Per-entity model invocation        |
| **Optimization** | PIT joins, materialized views (FeathrPO) | Priority scheduling, regret (RALF) |
| **Bottleneck**   | I/O, scan cost                           | Compute, API rate limits           |

Today's feature stores commit to one side — Feast and Hopsworks are column-oriented, RALF is pure row-level. But real ML pipelines mix both: a fraud predictor uses `card_txn_count_24h` (column) *and* `cardholder_embedding` (row) in the same prediction.

A hybrid system needs to resolve:

* **Unified API** for defining both feature kinds with one abstraction
* **Global budget allocation** between SQL aggregations and embedding refreshes
* **Storage layout** — shared tier, or separate (vector DBs vs aggregation stores)?
* **Cross-mode consistency** for point-in-time joins spanning both feature types



Leave it to the future work.

