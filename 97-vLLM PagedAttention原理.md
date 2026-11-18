# PagedAttention：vLLM 的立身之本

## 问题：KV cache 的碎片

每个序列的 KV cache 大小是 `2 × L × H_kv × d_head × seq_len`，而且**长度事先未知**（生成到哪算哪）。

朴素做法有两种，都浪费：

- **预分配最大长度**：按 `max_model_len` 一次占满。一个 4k 上限的请求实际只生成 200 token，浪费 95%。
- **连续追加**：每次扩容都要连续内存，产生外部碎片；不同序列交错分配后，空闲块被切碎。

实测下来，朴素方案的有效显存利用率常只有 20%～40%。**显存利用率低 = 并发上不去 = 吞吐上不去。**

## PagedAttention 的做法

借操作系统的虚拟内存分页：

- 物理 KV cache 被切成固定大小的 **block**（默认 16 个 token 一块）。
- 每个序列维护一张 **block table**：逻辑块号 → 物理块号。
- 物理块**不需要连续**，按需分配，用完归还。
- 注意力计算时按 block table 去 gather（kernel 支持非连续块）。

效果：

- **内部碎片**：只有最后一个块可能没填满，浪费 < 一个 block（16 token）。总浪费通常 < 4%。
- **外部碎片**：不存在，因为块是固定大小的。
- 显存利用率能到 90%+，同样显存能跑更多并发。

## 顺带解决的两件事

**1. 前缀共享（prefix sharing）**
相同前缀的多个序列可以指向同一批物理块（引用计数 + 写时复制）。典型场景：

- 同一 prompt 采样多个回答（parallel sampling / best-of-n）
- beam search 的多个分支
- 系统提示词相同的所有请求

**2. 前缀缓存（prefix caching）**
把"已经算过的前缀"的块缓存下来，新请求命中就直接复用。命中判据是**块内容的哈希**（token 序列 + 前缀哈希），不是简单的字符串匹配。

vLLM 的 prefix caching 是块粒度的：只有完整块能命中，最后一个不完整的块不缓存。

## 写时复制（Copy-on-Write）

共享块被某个序列修改时（比如采样分叉后内容不同），才复制一份。这和操作系统的 fork + COW 是同一个思路：

- 没分叉时共享，省显存
- 分叉时才复制，保证正确性
- 引用计数归零才真正释放

## 代价

- 注意力 kernel 要支持非连续访问（paged attention kernel），实现比连续内存复杂。
- 块表本身有元数据开销（每块一个 int，相对很小）。
- 块大小是权衡：太小 → 元数据多、gather 频繁；太大 → 内部碎片多。16 是常见默认。

## 追问链（三层）

**Q：为什么不能像传统 KV cache 那样连续分配？**
A：因为序列长度不可预知且不断增长，连续分配要么预留过多（浪费），要么频繁 realloc（碎片）。分页把"逻辑连续"和"物理连续"解耦了。

**追问：block table 存在哪？**
A：在 CPU 内存里（每个序列一张小表），计算前传到 GPU。表很小（几 KB），传输开销可忽略。

**再追问：注意力 kernel 怎么处理非连续块？**
A：kernel 按 block table 索引，逐块加载 K/V 到片上，做分块 attention（类似 FlashAttention 的 tiling）。所以 PagedAttention kernel 通常是 FlashAttention 的变体。

**Q：prefix caching 的命中率怎么提高？**
A：让前缀稳定且长：系统提示词固定、few-shot 固定、工具 schema 排序固定。任何前缀里的动态内容（时间戳、随机 id）都会让缓存从那里失效。

**Q：块大小怎么选？**
A：默认 16。序列短而多时，小块减少浪费；序列长且稳定时，大块减少元数据开销。一般不用改，除非有明确瓶颈。

**Q：PagedAttention 和 FlashAttention 是竞争关系吗？**
A：不是。FlashAttention 解决"注意力计算不落盘大矩阵"，PagedAttention 解决"KV cache 怎么存"。vLLM 的注意力 backend 通常同时用两者的思想。
