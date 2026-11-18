# vLLM 架构总览：一次请求在引擎里怎么走

## 分层结构

```
API Server（OpenAI 兼容 /v1/chat/completions, /v1/completions）
   ↓ 请求入队
LLMEngine（对外接口：add_request / step）
   ↓
EngineCore（核心：调度器 + KV cache 管理 + 执行器）
   ├─ Scheduler          决定这一轮跑哪些请求
   ├─ KVCacheManager     分配/释放物理块、前缀缓存
   └─ Executor           调 Worker 执行模型
        └─ Worker（每卡一个进程）
             ├─ ModelRunner   前向计算
             ├─ AttentionBackend（FlashAttention / FlashInfer / …）
             └─ Sampler       采样出 token
```

V1 引擎把「调度 + KV 管理 + 执行」放进 EngineCore，与 API 层解耦，方便做多进程/多卡/分离式部署。

## 一次 step 做什么

一个 `step()` 就是一次迭代：

1. **调度**：Scheduler 从等待队列和运行队列里选出这一批要跑的序列，决定它们是 prefill 还是 decode。
2. **分配 KV 块**：为新增的 token 分配物理块，更新 block table。
3. **执行**：Worker 把这一批组成一个 batch，跑一次前向（prefill 用变长 attention，decode 用 paged attention kernel）。
4. **采样**：从 logits 里采样出下一个 token。
5. **更新状态**：追加 token、检查是否结束（EOS / max_tokens / stop 字符串）、释放已结束序列的块。

这个循环的节奏就是 continuous batching 的本质：**每一轮都重新组批**。

## 关键设计：为什么必须"按迭代调度"

传统做法是"一个请求跑完再下一个"或"固定 batch 跑完再换"。问题是：

- 序列长度差异巨大（10 token 和 4000 token），固定 batch 里短的等长的，GPU 空转。
- 新请求要等整批结束，TTFT 很差。

迭代级调度让每个 step 重新决定批次，短请求完成立刻退出、新请求立刻进入。

## KV cache 是核心资源

整个引擎的调度本质上是在管理一个有限的显存池：

- 显存不够 → 要么抢占（preempt）某个序列，要么拒绝新请求。
- 块分配失败是"这一轮能不能跑"的主要判据。

所以 vLLM 的很多参数（`gpu_memory_utilization`、`max_model_len`、`max_num_seqs`）本质都在算"能装下多少 KV 块"。

## 追问链（三层）

**Q：vLLM 和 HuggingFace `generate()` 的本质区别？**
A：HF 是单请求、静态批、每步重算或简单 cache；vLLM 是迭代级调度 + 分块 KV 管理 + 优化 kernel。单请求延迟可能差不多，但并发吞吐差一个量级。

**追问：为什么吞吐能差这么多？**
A：三个来源：连续批处理（GPU 不空转）、PagedAttention（显存利用率从 30% 提到 90%+）、以及 prefix caching（重复前缀不重算）。

**再追问：单请求会更慢吗？**
A：可能略慢，因为有调度和分块的开销。vLLM 的收益在并发场景。

**Q：为什么 vLLM 要做 OpenAI 兼容层？**
A：降低接入成本——现有应用改个 base_url 就能用。这也是它能快速普及的原因之一。

**Q：EngineCore 和执行器分开有什么好处？**
A：调度是 CPU 逻辑，执行是 GPU 逻辑。分开后可以多进程（避免 GIL）、可以换后端（单机/多机/Ray）、也能做 prefill/decode 分离部署。
