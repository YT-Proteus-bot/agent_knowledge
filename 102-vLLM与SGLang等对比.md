# vLLM 与 SGLang / TGI / TensorRT-LLM 的对比

## 五个主流推理引擎

| 引擎 | 核心优势 | 代价 | 适合 |
|---|---|---|---|
| **vLLM** | PagedAttention、模型覆盖广、社区大 | 极致性能不如 TRT-LLM | 通用自建推理，默认选择 |
| **SGLang** | RadixAttention（前缀树复用）、结构化输出强 | 生态比 vLLM 小 | 多轮对话、Agent、大量共享前缀 |
| **TGI** | HF 官方、Rust 实现、生产特性全 | 灵活性和性能调优空间小 | 已有 HF 生态、要开箱即用 |
| **TensorRT-LLM** | NVIDIA 官方，kernel 融合到极致，最快 | 要为每个模型/GPU 编译，绑定强 | 追求极致性能、模型固定 |
| **LMDeploy** | TurboMind 引擎，国产模型适配好 | 通用性弱一些 | InternLM/Qwen 等国产模型 |
| **llama.cpp** | CPU/量化 GGUF，边缘部署 | 吞吐不适合服务端 | 本地、端侧、低配机器 |

## vLLM vs SGLang：最常被拿来比

两者都做 continuous batching + KV cache 复用，差异在**前缀复用的粒度**：

- vLLM 的 prefix caching 是**块粒度哈希**，命中需要完整块一致。
- SGLang 的 RadixAttention 用**基数树**组织前缀，能更细粒度地复用（部分匹配也能命中），在多轮对话、树状采样、Agent 场景优势明显。

另外 SGLang 的前端 DSL（生成 + 控制流）对复杂编排更友好。

结论：**通用场景 vLLM 够用且生态好；前缀复用极重（Agent、多轮、few-shot 相同）时 SGLang 可能更快。**

## vLLM vs TensorRT-LLM

- TRT-LLM 把算子融合、量化、kernel 都编译进 engine，单卡性能通常最高。
- 但"编译"意味着：换模型要重编、换 GPU 架构要重编、调试困难、模型支持滞后。
- vLLM 是"加载即用"，灵活但 kernel 层面没那么极致。

一句话：**TRT-LLM 是"性能上限"，vLLM 是"工程下限"（能跑、好改、覆盖广）。**

## 选型建议（面试可讲）

1. 先把 vLLM 跑起来，拿到基线。
2. 如果瓶颈在共享前缀 → 试 SGLang。
3. 如果瓶颈在 kernel 效率且模型固定 → 试 TRT-LLM。
4. 如果要边端/CPU → llama.cpp。
5. 别一上来就上最复杂的，调优空间比引擎差异更大。

## 追问链（三层）

**Q：为什么 RadixAttention 能比块哈希命中更多？**
A：块哈希以"完整块"为单位，块边界不对齐就命不中；基数树按 token 前缀组织，任意长度的公共前缀都能复用，回收时按 LRU 淘汰叶子。

**追问：那 vLLM 为什么不改？**
A：块粒度是 PagedAttention 的设计前提（物理块固定大小）。改成树状会动到内存管理的根基，工程量大。vLLM 也在演进（V1 的 prefix caching 已经改进很多）。

**再追问：实际差距有多大？**
A：取决于前缀重复率。共享前缀占 80% 时差距明显；每次请求都是全新内容时，两者差不多。

**Q：为什么 TRT-LLM 编译这么麻烦还快？**
A：因为它能针对具体 GPU 架构做算子融合、内存布局优化、kernel 自动调优。这些在"通用加载"模式下做不到。

**Q：自建推理最先该优化什么？**
A：先看是不是真的受限——很多时候瓶颈在业务逻辑、数据库、网络，不在推理引擎。确认是引擎瓶颈后，再按"批处理 → 前缀缓存 → 量化 → 并行"的顺序来。
