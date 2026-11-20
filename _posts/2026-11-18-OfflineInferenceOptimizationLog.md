---
layout: post
title: "CheatSheet：提高推理 QPS"
author: "Lingze Zeng"
image: ink_wafer.jpg
tags: [Qwen3.8-27B, SGLang]
---

*Qwen3.8-27B（GDN + full attention）在 SGLang / H100 上做离线批推理时用到的技巧，按影响的因子归类。*

系列（按顺序阅读）：1. [混合注意力 LLM 离线推理](/HybridModelConcurrency/) · 2. [推理瓶颈在哪](/InferenceCostModel/) · 3. [提高推理 QPS](/OfflineInferenceOptimizationLog/) · 4. [稀疏 MoE 与 FP4](/BeyondH100/)

负载是 prefill heavy 的：每请求约 12K 输入、约 260 输出，所有请求共享同一段 system 前缀，输出是 schema 约束的 JSON。下面不少技巧（track_interval、共享前缀、紧凑 JSON）正是针对这种形态，换负载前先核对前提。

因子来自[成本模型](/InferenceCostModel/)对每请求 GPU 时间的拆分：

$$T_{gpu} \approx N_{\text{in}} \times c_{\text{in}} + \frac{N_{\text{out}}}{B} \times c_{\text{step}}, \qquad \text{QPS} = \frac{n_{GPU}}{T_{gpu}}$$

| 因子 | 含义 | 属性 |
|-|-|-|
| $$N_{\text{in}}$$ | 每请求输入 token 数 | 负载 |
| $$N_{\text{out}}$$ | 每请求输出 token 数（decode 步数） | 负载 |
| $$c_{\text{in}}$$ | prefill 每 token 成本（compute-bound，近似常数） | 引擎 |
| $$c_{\text{step}}$$ | decode 一步成本（bandwidth-bound，≈ 搬一遍权重 + 状态） | 引擎 |
| $$B$$ | 运行时并发，一次权重搬运摊给多少 token | 引擎 |

$$c$$ 泛指 $$c_{\text{in}}$$ 与 $$c_{\text{step}}$$。

**背景速记**（细节见[第 1 篇](/HybridModelConcurrency/)）：

- 模型 64 层 = 48 层 GDN（线性注意力）+ 16 层 full attention，bf16 权重约 57 GB。
- 每个在途请求驻留两份状态：KV cache 每 token 64 KB（bf16），随长度增长；GDN 递归状态每请求 148 MB，固定大小。
- SGLang 把两者放进两个显存池：kv cache 和 mamba cache（按槽分配）。两池合计受 `gpu_memory_utilization`（gmu）约束，切分比例由 `max_mamba_cache_size` 决定。
- 并发上限 $$n = \min(n_{mamba}, n_{kv})$$，其中 $$n_{mamba} \approx \mathtt{max\_mamba\_cache\_size}/5$$（每请求约占 5 个槽），$$n_{kv}$$ = KV 池 ÷ 每请求 KV。
- 所有请求共享的前缀由 radix 树（前缀树）缓存复用；prefill 在 SGLang 里叫 extend。
- TP 指张量并行（模型切到多卡）；fa3 指 FlashAttention-3 attention 后端；Hopper 是 H100 的架构。

> 显存重切永远放在每轮最后做：任何改变显存占用或序列长度的改动之后都要重来一遍。

---

## 官方 FP8 ckpt × TP=1

- **机制**：权重以 FP8 存盘（每 128 元素一块共享一个 scale），28.5 GB 单卡放得下，部署从 TP=2 改为 TP=1：每卡一个独立副本，省 TP 通信。
- **因子**：$$c$$ + $$B$$。精度损失有限，因为是官方校准后的 FP8 版本：[Qwen3.8-27B-FP8](https://huggingface.co/Qwen/Qwen3.8-27B-FP8)。

FP8 可以使用 Hopper 自带的 FP8 Tensor Core。

> 虽然是 FP8 版本，但不是所有 layer 都量化了，量化的大多是 linear layer：64 层 FFN / 48 层 GDN 的 in_proj_qkv、out_proj / 16 层 full attention 的 q/k/v/out_proj。其余都保持原始精度。
>
> 量化逻辑：只量化大矩阵乘 GEMM；精度敏感或不走 GEMM 的部分（如 RMSNorm、conv）保持原精度；KV cache 用 bf16，GDN 递归状态用 fp32。

模型参数从 57 GB 降到 28.5 GB，H100 80 GB 可以用 TP=1 部署：

1. 收益：省去卡间通信与跨卡协同（没有 All-Reduce），GEMM 不再切半，kernel 效率更高。
2. 代价：权重不分摊，KV cache 和 mamba cache 也不再跨卡共享，单卡并行减少。

---

## 显存重切

- **机制**：重新调 gmu 和两池切分，让 $$n_{mamba}$$ 与 $$n_{kv}$$ 同时到顶（原理见第 1 篇「并发数确定」）。
- **因子**：$$B$$。无损（纯显存分配）。

两件事：

1. gmu 0.8 时显存峰值约 75 GB，还有空余，调到 0.9。
2. 不指定 `max_mamba_cache_size` 时 SGLang 按默认逻辑切 KV 与 mamba 池，不符合负载需要的平衡点。TP=1 后默认并发上限 22，实测只有 19.8：mamba 槽闲置、KV 先满。设 `max_mamba_cache_size = 130`，并发到 26，与估计相符。

KV FP8 省下显存后要再切一轮（当时版本 170，升级引擎后又回到 130）。

---

## mamba_track_interval

- **机制**：GDN 状态每 `mamba_track_interval` 个 token 落一次快照供前缀复用。
- **因子**：$$c$$。无损。

GDN 状态是原地覆盖的，要在序列中间复用前缀，就必须从快照恢复；track_interval 控制快照落点密度：

- 调小：中途命中总能找到近的快照，恢复代价小，但拷贝频率高。序列长时高频拷贝是额外开销；decode 中撞上状态拷贝还会出长尾。
- 调大：拷贝开销小，但序列中间没有快照，中途复用退化为重新计算。

本负载：

1. 固定单轮，复用命中全部落在共享 system 前缀，序列中间不存在复用点，即使中途落了快照也没人用；
2. $$N_{\text{in}} \approx 12\text{K}$$ 很长，小 interval 会放大拷贝开销。

所以把默认的 256 调成 65536。

---

## KV FP8（fp8_e4m3）

- **机制**：KV cache 从 bf16 变成 fp8_e4m3，每 token 显存减半；attention 里的 KV 过 FP8 tensor core 更快。
- **因子**：$$c$$ + $$B$$。需要校准 + 质量验证。

decode attention 是带宽 bound，字节减半直接意味着搬运量减半，而且省出的显存提高并发。

担忧在精度：k/v 投影 GEMM 输出的 K、V 是 bf16 激活，写入 cache 时量化成 fp8；fa3 后端在 KV 是 fp8 时会把 q 从 bf16 cast 成 fp8，这里也会丢精度。

> Full attention 里 k/v 投影的实际流程：
> 1. 输入激活在线量化成 fp8；权重本来就是 fp8 block-128（带 scale）。
> 2. fp8 tensor core 做乘法，累加器是 fp32，中间结果精度远高于 fp8。
> 3. 激活 scale × 权重 scale 逐块乘回 fp32 部分和，再 cast 成 bf16 输出。
>
> 开 FP8 KV 时，在 bf16 输出后再 cast 成 fp8 写入 cache。

**误差校准**：SGLang 支持从 ckpt 加载逐层静态 k_scale / v_scale，fa3 会把它们作为 descale 传入 FP8 attention kernel，缩小 KV 写入时的量化误差。做法：用一份稳定的 workload 在 FP8 ckpt 上跑校准前向，统计每层写入 KV cache 的 K/V 激活，得出逐层 k_scale / v_scale 写回 ckpt，用 llmcompressor 即可。

---

## SGLang 引擎修复与升级

- **因子**：$$c$$。

旧版本里 torch profile 定位到的三处引擎问题，曾以插件形式修补，0.5.20 已由上游覆盖：

1. prefill 下 mamba 状态写一半就进前缀树的竞态（上游 issue #24221）；
2. GDN extend 路径每层 7 次拷贝 + 1 次 cat（上游 PR #36267）；
3. 多余的 GPU→CPU 同步点。

升级到 0.5.20 后，启用 KV FP8 会使 `attention_backend`（默认 auto）从 fa3 回退成 FlashInfer。要显式声明 fa3：在 Hopper 上 fa3 效果比 FlashInfer 好（实验验证）。

---

## GDN kernel 融合

- **机制**：重写 GDN 相关的 triton kernel，做 kernel 融合，减少中间变量读写。
- **因子**：$$c$$。无损，纯引擎侧。

extend 侧 5 处：norm / 激活核与下游 GEMM 的 fp8 输入量化合成单核：

- gated RMSNorm → out_proj
- conv1d + l2norm
- SiLU·mul → down_proj
- add + RMSNorm → gate_up_proj
- sigmoid·mul → o_proj

每处省一次 bf16 写回、一次读入和一次 kernel 启动；GEMM 计算量本身不变。

---

## 紧凑 JSON

- **机制**：SGLang 的 JSON 约束默认允许任意空白，模型按训练习惯输出缩进 + 换行，空白占输出 token 19.1%；关掉后每行结构 token 172 → 106。
- **激活**：`constrained_json_disable_any_whitespace: true`。
- **因子**：$$N_{\text{out}}$$。输出分布会变化，需业务质量确认；理论上语义一致。

pretty JSON 和 compact JSON 语义相同，但前者有约 20% 的空白缩进 token。

```json
{
  "uid": 10086,
  "items": [
    {"id": 501, "cat": "3c", "price": 2999.0},
    {"id": 502, "cat": "food", "price": 15.5}
  ],
  "ts": "2026-11-17 12:00:00"
}
```

```json
{"uid":10086,"items":[{"id":501,"cat":"3c","price":2999.0},{"id":502,"cat":"food","price":15.5}],"ts":"2026-11-17 12:00:00"}
```

---

## Prompt 瘦身

- **机制**：改写 prompt，保持语义一致，用不同格式换取更少的 input token。下面三种里实际采用的是共享前缀 + 数值改写。
- **因子**：$$N_{\text{in}}$$。虽然是同语义替换，输出分布会变化，需要质量校验。

**共享前缀**：各请求相同的说明与全局基线行从 user 挪进 system prompt；system 里对应补充各段说明原文、基线行、全零行的省略规则。user 侧 token 节省约 6%。总 token 没减少，但挪进前缀的部分由 radix 缓存复用，不再重复计算。

```text
[0] BASELINE — 该用户整周的汇总，先一行总计，再按自然日拆分。
B global baseline cohort=all users=N metric_a=+0.238 metric_b=+0.211 metric_c=+0.106
B hard_neg event_x=0 event_y=0 event_z=0
[1] CONDITIONS — 该用户身上度量到的可执行条件。每行末尾是该条件对上其余时候的差与标准误。
```

```text
[0] BASELINE
（删去：全局基线行，所有用户相同）
（删去：全零行）
[1] CONDITIONS
```

**数值改写**：比率乘 100 写成百分数、比率差写成百分点、去末尾 0，数值相同，每个数省 1–3 token。user 侧合计约 −12% token。

```text
B priors attr_a=inactive attr_b=0.0 attr_c=1.0
C HOUR_21_21 n=50(35.7%) sessions=2 metric_a=0.160(-13pp) metric_b=0.360(+22pp) metric_c=0.060(+4pp) cold=0.0% dev=+0.092
```

```text
B priors attr_a=inactive attr_b=0 attr_c=1
C HOUR_21_21 n=50(35.7%) sessions=2 metric_a=16%(-13pp) metric_b=36%(+22pp) metric_c=6%(+4pp) cold=0% dev=+9.2
```

更激进的做法是把数值预处理成分位数，直接换成文字。

**key=value 行改 Markdown 表**：键名只在表头出现一次。

```text
O target_1 n=56 metric_a=0.143(-17pp)* metric_b=0.429(+35pp)* metric_c=0.054(+3pp) cold=0.0% evidence_days=2
O target_2 n=0
```

```text
| 目标 | n | metric_a | metric_b | metric_c | cold | evidence_days |
|---|---|---|---|---|---|---|
| target_1 | 56 | 0.143(-17pp)* | 0.429(+35pp)* | 0.054(+3pp) | 0.0% | 2 |
| target_2 | 0 |  |  |  |  |  |
```

在当时的 prompt 版本里这样做 input token 反而变多，未采用；但信息密度大、稀疏列少时，是值得用的压缩方法。
