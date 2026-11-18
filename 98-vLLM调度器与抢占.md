# vLLM 调度器：连续批处理、chunked prefill 与抢占

## 三个队列

- **waiting**：还没开始（新请求、或被抢占后回来）
- **running**：正在生成
- **swapped**：被换出到 CPU 的（如果启用了 swap）

每个 step，调度器从 waiting 和 running 里选出这一批，组成一次前向。

## 连续批处理（continuous batching）

传统静态批：一批请求一起开始、一起结束，短的等长的。
连续批处理：**每个 step 重新组批**，完成的立刻退出，等待的立刻补位。

收益：GPU 利用率大幅提升，尤其当序列长度差异大时。这是 vLLM 吞吐高的第一来源。

## Chunked Prefill

问题：一个 32k 的 prompt，prefill 一次要跑很久，期间所有 decode 都被卡住，TPOT 飙升。

做法：把 prefill 按 `max_num_batched_tokens` 切成多块，每个 step 只算一块，和 decode 混在同一批里。

效果：

- 长 prompt 不再"独占"GPU，decode 的 TPOT 稳定
- 代价是长请求的 TTFT 变长（要分多次算完）

这是"延迟公平性"和"吞吐"之间的显式取舍，vLLM 默认开启（V1 里 chunked prefill 是常态）。

## 抢占（Preemption）

当 KV cache 物理块不够时，必须牺牲某些序列腾出空间。策略：

- **选谁**：通常是最近加入的序列（LIFO），因为它们最可能被放弃，且重算成本低。
- **怎么处理**：
  - **Recompute（默认）**：丢掉它的 KV cache，回到 waiting 队列，之后重新 prefill（包括已经生成的部分）。省显存、但浪费计算。
  - **Swap**：把 KV 块换到 CPU 内存，之后换回来。省计算、但占 CPU 内存和 PCIe 带宽。

选择依据：序列短、重算便宜 → recompute；序列长、CPU 内存够 → swap。

## 关键参数

- `max_num_seqs`：同时跑的序列数上限。越大吞吐越高、单请求延迟越差。
- `max_num_batched_tokens`：一个 step 处理的 token 上限（prefill chunk 大小也由它决定）。太小 → 长 prompt 慢；太大 → 显存峰值高、decode 延迟抖动。
- `scheduling_policy`：FCFS 或 priority。有 SLA 分级时用 priority。
- `enable_chunked_prefill`：V1 默认行为，V0 需要显式开。

## 一个必须理解的矛盾

**吞吐、TTFT、TPOT 三者不可能同时最优。**

- 想吞吐高 → 大 batch → 每个请求分到的时间少 → TPOT 变差。
- 想 TTFT 低 → 优先跑 prefill → decode 被插队 → TPOT 抖动。
- 想 TPOT 稳 → 限制 prefill 比例（chunked prefill）→ 长 prompt 的 TTFT 变长。

所以线上调参的本质是：**先定 SLO（比如 TTFT < 2s、TPOT < 50ms），再在这个约束下最大化吞吐。**

## 追问链（三层）

**Q：为什么 chunked prefill 能降低 TPOT？**
A：因为一次 prefill 的时间正比于 token 数。不切块时，一个 32k 的 prefill 会占用整个 step，期间 decode 全部等待。切块后每个 step 只花固定预算在 prefill 上，decode 能持续得到调度。

**追问：那为什么不把 chunk 设得很小？**
A：因为每次前向都有固定开销（kernel 启动、权重读取），块太小会让开销占比过高，prefill 总时间反而变长。

**再追问：怎么找平衡点？**
A：看 `max_num_batched_tokens` 的实测曲线：横轴是值、纵轴是 TTFT 和 TPOT。通常取"TPOT 满足 SLO 的最小值"。

**Q：抢占之后序列要重新 prefill，会不会很浪费？**
A：会。所以好的调度应该尽量避免抢占——比如提前估算显存、限制 `max_num_seqs`、或者对长序列做准入控制。

**Q：为什么默认选 LIFO 抢占？**
A：因为最新加入的序列通常生成得最少，重算成本最低；而且它最可能是"用户刚发的、还没等太久"的，抢占它比抢占一个已经跑了很久的序列更不伤体验（这个取舍有争议，可按策略调整）。
