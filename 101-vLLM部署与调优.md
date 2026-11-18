# vLLM 部署与调优：参数、监控与常见故障

## 最小启动

```bash
vllm serve <model> --tensor-parallel-size 2 --max-model-len 8192
```

默认起一个 OpenAI 兼容服务（`/v1/chat/completions`），可以直接把现有应用的 base_url 改过来。

## 关键参数（按重要性）

| 参数 | 作用 | 调错的表现 |
|---|---|---|
| `--max-model-len` | 单序列最大长度 | 设太大 → 启动 OOM；设太小 → 长请求被拒 |
| `--gpu-memory-utilization` | 预留给模型的显存比例（默认 0.9） | 太高 → 激活峰值 OOM；太低 → KV 块少、并发低 |
| `--max-num-seqs` | 同时跑的序列数 | 太大 → 延迟抖动；太小 → 吞吐上不去 |
| `--max-num-batched-tokens` | 单 step token 上限 | 太小 → TTFT 差；太大 → TPOT 抖动 |
| `--enable-prefix-caching` | 前缀缓存 | 不开 → 多轮/Agent 场景浪费大量重复 prefill |
| `--quantization` | 量化方式（awq/gptq/fp8） | 与显卡/kernel 不匹配会退化成慢路径 |
| `--kv-cache-dtype` | KV cache 精度（auto/fp8） | 长上下文时不开 fp8 → 并发上不去 |
| `--swap-space` | CPU 交换空间（GB） | 太小 → 抢占只能 recompute |
| `--enforce-eager` | 禁用 CUDA Graph | 调试用，生产开了会掉性能 |

## 调参顺序（面试可讲的方法论）

1. 先定 `max-model-len`：业务需要多长就设多长，别贪大。
2. 看启动日志的 "GPU blocks"，确认显存能撑多少并发。
3. 用压测找 `max-num-seqs`：固定输入输出长度，画"并发 vs TPOT"曲线，取满足 SLO 的最大值。
4. 再调 `max-num-batched-tokens`：让长 prompt 的 TTFT 和 decode 的 TPOT 都达标。
5. 最后开 prefix caching / 量化 / KV fp8，逐个验证收益。

一次只改一个参数，否则不知道是谁的功劳。

## 监控指标

- **TTFT 分布**（P50/P95/P99）：首字延迟
- **TPOT / ITL 分布**：词间延迟
- **吞吐**：tokens/s、requests/s
- **队列等待时间**：调度是否积压
- **KV cache 使用率**：块占用率，接近 100% 就会开始抢占
- **抢占次数**：频繁抢占说明并发开太大或显存不够
- **GPU 利用率 / 显存**：区分计算瓶颈还是带宽瓶颈

vLLM 暴露 Prometheus `/metrics`，上面这些基本都有。

## 常见故障与排查

**启动 OOM**
→ 降 `max-model-len`、降 `gpu-memory-utilization`、用 TP 分摊、或量化。

**启动很慢**
→ 大模型加载 + CUDA Graph 捕获 + 编译 kernel 都要时间。可以预热，或者用 `enforce_eager` 定位是不是图捕获的问题。

**首字很慢**
→ 前缀缓存没命中、`max-num-batched-tokens` 太小、或队列里长 prompt 太多。

**吞吐上不去**
→ `max-num-seqs` 太小、KV 块不够（抢占多）、或没开 prefix caching。

**输出延迟抖动**
→ 长 prefill 插队。开 chunked prefill、限制单请求长度、或者做长短请求分池。

## 追问链（三层）

**Q：为什么 `gpu_memory_utilization` 不能设 1.0？**
A：因为要留空间给激活峰值、CUDA context、通信 buffer、以及可能的其他进程。设满会在 prefill 大 batch 时 OOM。

**追问：那怎么知道该留多少？**
A：看启动后实际峰值。压测时监控显存，留 5%～10% 余量。不同模型/序列长度的激活峰值差异很大。

**再追问：OOM 是发生在 prefill 还是 decode？**
A：多数在 prefill（激活随 batch×seq 增长）。decode 的激活很小，OOM 通常是 KV 块不够导致的调度问题，而不是真的显存爆。

**Q：prefix caching 会不会导致"缓存污染"？**
A：会有显存竞争：缓存块也占 KV 池，命中率低时反而挤占可用块。所以短前缀、唯一前缀多的场景，收益可能为负。

**Q：多实例还是单实例大 TP？**
A：看瓶颈。单实例 TP 大 → 单请求延迟低、但扩展性受 NVLink 限制；多实例 → 吞吐线性扩展、但每实例都要装一份权重（显存浪费）。通常按"模型能否单机放下"来决定。
