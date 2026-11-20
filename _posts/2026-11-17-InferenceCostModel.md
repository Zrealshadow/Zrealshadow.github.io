---
layout: post
title: "深入 Cost Model：推理瓶颈在哪"
author: "Lingze Zeng"
image: ink_wafer.jpg
tags: [Qwen3.8-27B, SGLang]
---

*把 prefill 和 decode 统一成"搬运 + 计算"，再把每请求 GPU 时间拆成四个因子，用来给每一项优化做归因。*

系列（按顺序阅读）：1. [混合注意力 LLM 离线推理](/HybridModelConcurrency/) · 2. [推理瓶颈在哪](/InferenceCostModel/) · 3. [提高推理 QPS](/OfflineInferenceOptimizationLog/) · 4. [稀疏 MoE 与 FP4](/BeyondH100/)

---

## 0. 设定与符号

以 Qwen3.8-27B 为例，bf16 部署在 H100 上。它是混合注意力模型：64 层里 48 层是 GDN（线性注意力），16 层是 full attention。对搬运量而言，只需要知道每个在途请求在显存里驻留两份状态：

- **KV cache**：由 16 个 full attention 层产生，随序列长度增长，每 token 64 KB（bf16）。
- **GDN 递归状态**：由 48 个 GDN 层产生，大小固定，每请求 148 MB，与长度无关。

所以一个长度为 $$L$$ 的请求驻留 $$s(L) = L \times 64\text{KB} + 148\text{MB}$$ 字节，decode 每步都要把它完整读一遍。

| 符号 | 含义 |
|-|-|
| $$P$$ | 模型参数量（约 28B，含 embedding）。bf16 下权重 $$2P \approx 57$$ GB，每 token 前向约 $$2P$$ FLOPs |
| $$M$$ | 一步前向同时处理的 token 数 |
| $$n$$ | 运行时并发：同时在 batch 里的请求数 |
| $$L$$ | 序列长度（输入 + 已生成） |
| $$C$$ | 一步里权重之外的额外搬运：$$C = n \times s(L)$$ |

---

## 1. Cost Model

Prefill 和 Decode 都概括成两个过程：从 HBM 搬运参数（Delivering）+ Tensor Core 计算 GEMM（Computing）。两者异步进行，取 MAX：

$$\text{Step Time} \approx \max\Big(\underbrace{\frac{2P + C}{\text{HBM Bandwidth}}}_{\text{Delivering}},\ \underbrace{\frac{2P \cdot M}{\text{TC Peak}}}_{\text{Computing}}\Big)$$

取哪一边由**一次权重搬运产出多少个 token**（即 $$M$$）决定：

- Delivering：每步把 $$2P$$ 字节权重和 $$C$$ 字节状态搬一遍，与 $$M$$ 无关。
- Computing：$$M$$ 个 token 各做一遍 GEMM，随 $$M$$ 线性扩大。

$$H = \frac{\text{Computing}}{\text{Delivering}} = \underbrace{\frac{\text{Bandwidth}}{\text{TC Peak}}}_{1/k,\ \text{硬件常数}} \times \underbrace{\frac{2P \cdot M}{2P + C}}_{\text{负载项}} = \frac{M}{k \cdot (1 + C/2P)}$$

> $$H > 1$$：Computing-Bound，优化目标是减少计算。
> $$H < 1$$：Bandwidth-Bound，优化目标是减少搬运、打高并发。

H100 SXM（HBM3 带宽 3.35 TB/s，规格见 [NVIDIA H100 datasheet](https://www.nvidia.com/en-us/data-center/h100/)）的硬件常数：

| 计算精度 | TC Peak | $$k$$ = TC Peak ÷ Bandwidth |
|-|-|-|
| BF16 | 989 TFLOPS | ≈ 300 FLOP/Byte |
| FP8 | 1979 TFLOPS | ≈ 590 FLOP/Byte |

**Prefill 阶段**

prefill 几乎不搬运权重之外的数据（只回读本请求已算完的前缀 KV，相对 $$2P$$ 可忽略），取 $$C \approx 0$$；长 prompt 会被引擎切成固定大小的 chunk 分步前向（SGLang 里由 `chunked_prefill_size` 控制），所以一步的 token 数 $$M = \mathtt{chunked\_prefill\_size}$$：

$$H_{\text{prefill}} = \frac{M}{k} = \frac{\mathtt{chunked\_prefill\_size}}{k}$$

代入 chunked_prefill_size = 8192、$$k \approx 300$$（bf16）：$$H \approx 27$$。prefill 是 compute-bound：tensor core 打满，权重搬运完全藏在计算后面。

**Decode 阶段**

decode 一步的 $$M$$ = batch 内在途请求数 $$n$$（每请求只出 1 个 token），而 $$C$$ 不再可忽略，每步要读全部 $$n$$ 个请求的 KV + GDN 状态：

$$H_{\text{decode}} = \frac{n}{k \cdot (1 + C/2P)} = \frac{1}{k\cdot\left(\frac{1}{n}+\frac{s(L)}{2P}\right)}$$

举例：$$L = 12\text{K}$$ 时 $$s(L) \approx 0.77\text{GB} + 0.15\text{GB} \approx 0.92\text{GB}$$；并发 $$n = 26$$ 时 $$C \approx 24\text{GB}$$，$$C/2P \approx 24/57 \approx 0.42$$；代入 $$k \approx 300$$：$$H \approx 26 / (300 \times 1.42) \approx 0.06$$。

decode 是 bandwidth-bound：tensor core 九成以上时间在等数据，一步耗时基本等于把 $$2P + C$$ 这几十 GB 过一遍总线的时间。所以 decode 的优化只有两条路：**把 $$n$$ 抬上去，或把要搬的字节砍下来**。

---

## 2. Metrics：QPS 不是一个纯粹的引擎指标

优化过程中出现过这样一个极端例子：

> 一次 4-bit 权重量化（W4A8）实验中，decode 速度增加了 19%：权重以 4-bit 存放，计算时在 kernel 内反量化成 FP8 再进 FP8 TensorCore，搬运的权重字节减半。但量化带来精度损失，模型反复 decode 重复的 token，平均 output token 变多。模型 decode 变快了，QPS 却没变。

> 模型效率变快，但 decode output token 变多，QPS 依旧不会提升。模型输出的质量一定程度上也会影响模型的效率。

设每请求输入 $$N_{\text{in}}$$ 个 token、输出 $$N_{\text{out}}$$ 个 token，把每请求的 GPU 时间拆开：

$$T_{gpu} = N_{\text{in}} \times \frac{c^{\,p}_{\text{step}}}{M_p} + N_{\text{out}} \times \frac{c^{\,d}_{\text{step}}}{M_d}$$

其中 $$c^{\,p}_{\text{step}}$$、$$c^{\,d}_{\text{step}}$$ 是 prefill、decode 一步的耗时，$$M_p$$、$$M_d$$ 是这一步同行的 token 数。

两阶段同一形式：消费的 token 数 × 每 token 成本。每 token 成本 = 一步成本 $$c_{\text{step}}$$ ÷ 这一步同行的 token 数 $$M$$（Cost Model 里的 $$M$$ 或 $$n$$）。

再按两阶段的 bound 化简：

- prefill computing-bound：一步成本正比于塞进的 token 数，$$c_{\text{step}}/M$$ 是常数，记作 $$c_{\text{in}}$$；
- decode bandwidth-bound：一步成本就是搬一遍权重，计算可忽略，几乎不随并发变化；每请求一步出 1 个 token，$$M_d$$ 就是运行时并发，下面记作 $$B$$（即上一节的 $$n$$），$$c^{\,d}_{\text{step}}$$ 简记为 $$c_{\text{step}}$$。

$$T_{gpu} \approx N_{\text{in}} \times c_{\text{in}} + \frac{N_{\text{out}}}{B} \times c_{\text{step}}, \qquad \text{QPS} = \frac{n_{GPU}}{T_{gpu}}$$

$$n_{GPU}$$ 是 GPU 数（每卡一个副本时）。

四个因子里，$$N_{\text{in}}$$、$$N_{\text{out}}$$ 是**负载属性**（prompt 多长、模型输出多少步），$$c$$、$$B$$ 才是**引擎属性**（每 token 多少成本、一次搬运摊给多少 token）。

引擎优化得再好，如果输出变长、decode 步数变多，QPS 一样起不来；反过来 QPS 涨了也未必是引擎变好，可能只是输出变短了。所以每项优化都要归因：动的是哪个因子。

| 因子 | 含义 | 典型手段 |
|-|-|-|
| $$N_{\text{in}}$$ | 输入 token 量 | 压缩 prompt、复用所有请求共享的前缀 |
| $$N_{\text{out}}$$ | 输出 token 与 decode 步数 | 压缩输出格式、投机解码；开 thinking 会让步数成倍增加 |
| $$c$$ | 单位 token 的引擎成本（$$c_{\text{in}}$$、$$c_{\text{step}}$$） | 低精度权重 / KV、kernel 融合、更快的 attention 后端 |
| $$B$$ | 一次权重读取摊给多少 token（运行并发） | 省出显存给状态（低精度权重 / KV）、调整两份状态的显存切分 |

具体做法见[第 3 篇](/OfflineInferenceOptimizationLog/)。

因子之间有耦合，同一个改动常常同时动两个因子：

1. 换成官方 FP8 权重，既降 $$c$$（用上 H100 的 FP8 TensorCore，计算更快），又抬 $$B$$（权重显存减半，省出的显存给 KV 和 GDN 状态，能容纳更多并发）。
2. 也可能一正一负：把 GDN 递归状态从 FP32 降到 bf16，每请求状态更小，抬高了 $$B$$；但质量退化使 $$N_{\text{out}}$$ 变长，吞吐反而下降。

> 尽量选择只影响单一因子的优化手段；对影响多个因子的手段，按上面的等式归因，才能解释吞吐收益。
