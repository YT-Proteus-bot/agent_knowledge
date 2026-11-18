# vLLM 采样与结构化输出

## 采样管线

每步拿到 logits 后，按顺序过一串 logits processor：

1. **logit_bias**：对指定 token 加减分
2. **repetition_penalty / frequency_penalty / presence_penalty**：抑制重复
3. **temperature**：`logits / T`
4. **top_k**：保留前 k 个
5. **top_p（nucleus）**：累计概率截断
6. **min_p**：按最高概率的比例设阈值（比 top_p 更抗分布偏移）
7. **softmax → 采样**

这些都在 GPU 上按请求并行执行。每个请求有自己的 `SamplingParams`，同一个 batch 里可以混不同的采样配置。

## n、best_of 与 beam search

- `n`：同一个 prompt 生成 n 个独立样本（parallel sampling）。**共享前缀的 KV cache**（PagedAttention 的写时复制），所以成本远低于跑 n 次。
- `best_of`：生成更多候选，按累计 logprob 选最好的返回（返回 1 个）。
- `beam search`：维护 beam_width 条路径，也是靠前缀共享。

## 结构化输出（Guided Decoding）

要让模型输出合法的 JSON / 正则 / 语法，做法是**约束解码**：每一步把"会导致非法前缀"的 token 概率置零。

三种约束：

- **choice**：限定几个固定选项
- **regex**：正则
- **json_schema / grammar（EBNF）**：结构化语法

实现原理：把约束编译成有限状态机（FSM），每个 token 对应状态转移；采样时只保留能合法转移的 token。

vLLM 支持多种后端：**xgrammar**（较新，快）、**outlines**、**lm-format-enforcer**。不同后端对语法特性的支持有差异。

## 结构化输出的代价

- **编译开销**：复杂 schema 首次编译要时间，通常缓存。
- **约束过强会降质量**：模型被迫在受限空间里选，可能选到语义上不合适的 token。
- **流式输出**：约束解码也能流式，但要注意部分 JSON 在流式时可能暂时不合法（客户端要能处理）。

## 追问链（三层）

**Q：约束解码会不会改变模型分布？**
A：会。它把非法 token 的概率归零并重新归一化，所以输出分布是"原分布条件下的合法子集"。这是有意的，但如果约束太强，模型会"被迫"说错内容。

**追问：那为什么不用后处理修复 JSON？**
A：后处理只能修格式，修不了"字段缺失/值错误"。而且模型可能在格式错误时已经跑偏。约束解码从源头保证合法，省一次重试。

**再追问：什么时候不适合约束解码？**
A：输出很长、结构很复杂时，约束会显著增加解码开销，而且模型容易在受限空间里"卡住"。这时可以只约束关键字段，其余自由生成。

**Q：为什么 vLLM 要在 GPU 上做采样？**
A：因为 logits 在 GPU 上（`[batch, vocab]`，vocab 十万级）。搬到 CPU 再采样会成为瓶颈，尤其是大 batch。GPU 采样把整条管线留在显存里。

**Q：temperature=0 是贪心吗？**
A：是（严格说 top_k=1）。但即使 T=0，不同 batch 组合下也可能有微小数值差异，不保证跨请求完全一致。

**Q：parallel sampling 为什么便宜？**
A：因为 n 个样本共享同一段前缀的 KV cache（写时复制），只有分叉之后的部分才各自计算。跑 n 次的成本远高于 1 次带 n 采样的成本。
