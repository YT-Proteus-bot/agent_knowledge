# vLLM 执行层：分布式、CUDA Graph 与注意力后端

## Worker 与并行

每个 GPU 一个 Worker 进程，由 Executor 统一驱动。

- **张量并行（TP）**：把每层的权重切分到多卡（列并行 + 行并行），每层做 all-reduce。`tensor_parallel_size=N`。
- **流水并行（PP）**：层切段，跨段传激活。`pipeline_parallel_size`。
- **专家并行（EP）**：MoE 场景，专家分布到不同卡，token 走 all-to-all。
- **多机**：通过 Ray 做集群调度，Executor 换成 RayExecutor。

TP 的通信量随层数线性增长，所以**优先单机 NVLink**；跨机尽量用 PP 或 DP（多实例）。

## 显存预算怎么算

启动时 vLLM 会做一次 profiling：

1. 加载权重，量出权重占用。
2. 用 `gpu_memory_utilization`（默认 0.9）算出可用总量。
3. 减去权重、激活峰值、临时 buffer，剩下的全部给 KV cache。
4. 按 `block_size` 切成块，得到"总块数"。

所以启动日志里的 "GPU blocks: N" 就是并发能力的上限来源。调参时先看这个数。

几个常见 OOM 原因：

- `max_model_len` 设太大 → 单序列要的块多
- `max_num_seqs` 太大 → 同时占的块多
- 权重本身 + 激活峰值就超了（长 prompt 的 prefill 激活很占）

## CUDA Graph

decode 阶段每步的 kernel 很多（每层好几个），kernel 启动的 CPU 开销会累积。CUDA Graph 把"固定形状的一批 kernel"录下来，之后一次 replay，省掉启动开销。

vLLM 会为一批常见 batch size（如 1,2,4,8,...）预先捕获图，运行时选最接近的。

- `enforce_eager=True` 可以禁用（调试用，性能会掉）。
- 副作用：图捕获要求形状固定，所以动态 shape 场景（变长 prefill）通常不走图，只有 decode 走。

## 注意力后端

vLLM 支持多种 backend，按模型/显卡/是否分页选择：

- **FlashAttention**：通用、稳定，支持 paged KV。
- **FlashInfer**：更激进的优化（尤其 decode），对某些 GPU 更快。
- **Triton**：可移植性好，方便改。
- **XFormers**：老后端，逐步被替代。

选择通常自动，出问题时可手动指定排查。

## 分离式部署（P/D Disaggregation）

把 prefill 和 decode 拆到不同实例：

- Prefill 实例：算力型，处理长 prompt。
- Decode 实例：带宽型，长期驻留 KV cache。
- 中间通过高速网络传 KV cache。

好处是两类负载互不干扰、可以独立扩缩容。代价是 KV 传输开销和架构复杂度。这是近两年推理基础设施的热点方向。

## 追问链（三层）

**Q：TP 为什么比 PP 延迟低？**
A：TP 把单层的计算切开，所有卡同时算同一层，延迟接近 1/N（加上通信）。PP 是把层分段，一个 token 要依次穿过所有段，延迟不降（甚至因为流水填充更慢）。

**追问：那 PP 有什么用？**
A：省显存、跨机扩展。当单机放不下模型时，PP 是必须的；TP 受限于单机 NVLink 规模。

**再追问：TP 的通信瓶颈在哪？**
A：每层两次 all-reduce。层数越多、通信越多。所以 TP 通常不超过单机 8 卡，跨机用 PP。

**Q：CUDA Graph 为什么只对 decode 有用？**
A：因为 decode 的 batch 形状相对固定（每步一个 token），而 prefill 的形状随 prompt 长度变化。图要求静态形状，所以只适合 decode。

**Q：`gpu_memory_utilization` 设 0.95 是不是更好？**
A：不一定。留太少显存会导致激活峰值时 OOM，或者被其他进程（监控、其他服务）挤爆。0.85～0.92 是常见区间，要结合实际情况。
