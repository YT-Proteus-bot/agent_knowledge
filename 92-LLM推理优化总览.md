# LLM 推理优化总览：两个阶段、四类指标、七种手段

## 先分清两个阶段

- **Prefill**：把输入 prompt 一次性算完，生成 KV cache。矩阵大、并行度高，**计算受限**（算力打满）。
- **Decode**：逐个生成 token，每步只算一个位置。矩阵小、并行度低，**显存带宽受限**（大部分时间在搬权重和 KV cache）。

这个区分决定了所有优化的方向：prefill 优化算力利用率，decode 优化带宽和批处理。

## 四个指标：TTFT、TPOT、吞吐、goodput

- **TTFT（Time To First Token）**：首字延迟，主要由 prefill 决定。
- **TPOT / ITL（每 token 时间 / 词间延迟）**：decode 速度，用户感知的"打字速度"。
- **吞吐（tokens/s 或 req/s）**：整体处理能力，决定成本。
- **Goodput**：满足 SLO（比如 TPOT < 50ms）的有效吞吐。比裸吞吐更有意义。

面试里能区分"吞吐"和"goodput"，基本就说明做过线上。

## 七种手段（按收益排序）

1. **Continuous batching**：不等整批结束，谁完成谁退出、新请求随时插入。吞吐提升最直接（几倍到十几倍）。
2. **PagedAttention / KV cache 分块管理**：把 KV cache 按固定大小块管理，消除碎片，显存利用率从 20～40% 提到 90%+。**这是 vLLM 的立身之本**。
3. **Prefix caching**：相同前缀（系统提示词、few-shot、工具 schema）只算一次，命中缓存直接复用 KV。多轮对话、Agent 场景收益巨大。
4. **Chunked prefill**：把长 prompt 切成块，和 decode 混在一个批次里跑，避免"一个长 prefill 卡住所有 decode"。
5. **量化**：权重/激活/KV cache 降到 FP8/INT8/INT4，省显存、提带宽利用率。
6. **投机解码**：小模型猜几个 token，大模型批量验证。不改变分布，2～3 倍加速（命中率高时）。
7. **并行与 kernel**：TP 分摊单卡计算；FlashAttention/FlashInfer 减少显存往返；CUDA Graph 消除 kernel 启动开销。

## 一个必须理解的取舍

**批越大，吞吐越高，但单请求延迟越差。** 因为 GPU 要同时服务更多请求，每个请求分到的时间片变少。所以线上要按 SLO 调 `max_num_seqs` / `max_num_batched_tokens`，不是越大越好。

## Roofline 视角（加分项）

- Prefill：算术强度高，接近算力上限 → 优化目标是"别让 kernel 空转"（融合算子、大 batch）。
- Decode：算术强度低（每读一个权重只做很少的计算）→ 优化目标是"少搬数据"（量化、KV cache 压缩、更大 batch 摊薄权重读取）。

一句话：**prefill 拼算力，decode 拼带宽。**

## 追问链（三层）

**Q：为什么 decode 阶段 GPU 利用率很低？**
A：因为每步只生成一个 token，矩阵乘的 batch 维度很小，算力单元大部分在等数据。此时瓶颈是显存带宽，不是 FLOPS。

**追问：那怎么提高 decode 的利用率？**
A：把多个请求拼成 batch（continuous batching），或者用投机解码让一次前向验证多个 token。本质都是"一次搬运权重，服务更多 token"。

**再追问：batch 大会不会影响首字延迟？**
A：会。新请求要等当前批次调度，TTFT 变长。所以生产上常做"prefill 优先"或"分池"（长请求和短请求分开）。

**Q：prefix caching 为什么在 Agent 场景特别有效？**
A：因为每轮请求的前缀（system、工具 schema、历史）高度重复。没有缓存时每轮都要重算几万 token 的 prefill，有了缓存只算新增部分。

**Q：量化会掉效果吗？**
A：会，但幅度取决于方法和模型。FP8 通常几乎无损；INT4 在某些任务上明显下降。要用自己的评测集验证，不能只看 perplexity。
