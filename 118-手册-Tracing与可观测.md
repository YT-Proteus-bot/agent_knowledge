# Tracing、Debugging 与 Observability（手册节选）

> 来源：《大模型 Agent 实战从 0 到 1 学习手册》第 18 篇：Tracing、Debugging 与 Observability。以下为原文节选，按小节重排。

## 18.1 日志不等于 Trace

日志：
2026-09-26 ERROR tool failed
Trace：
Run
├── Model Call
├── Tool Call
│ └── HTTP Request
Trace 能看到一次任务的因果链。

## 18.3 每个 Tool Span 应记录什么

推荐：
{
"tool": "search_docs",
"arguments": {
"query": "MCP v2"
},
"status": "ok",
"latency_ms": 382,
"result_size": 4,
"retry_count": 0
}
敏感参数需脱敏。

## 18.4 Model Span

可记录：
provider
model
input_tokens
output_tokens
latency
finish_reason
structured_output_valid
不要默认把所有原始 Prompt 都永久保留。

## 18.5 OpenAI Agents SDK Tracing

当前 SDK 内建 tracing，覆盖：
LLM generations
tool calls
handoffs
guardrails
custom events
这是学习 Agent 执行过程非常直接的方式。
注意数据保留与组织政策；官方也说明某些 Zero Data Retention 场景下 tracing 有限制。

## 18.6 Metrics

最少：
runs_total
run_success_rate
run_failure_rate
tool_calls_per_run
tool_error_rate
turns_per_run
input_tokens
output_tokens
cost_per_run
latency_p50
latency_p95
approval_rate
cancel_rate
RAG 再加：
retrieval_latency
chunks_retrieved
rerank_latency

## 18.7 为什么 p95 比平均值重要

假设延迟：
1s
1s
平均：
4.8s
但用户会明显感到偶发 20 秒卡死。
因此生产关注：
p50

## 18.8 Agent Debugging 的正确顺序

Agent 突然变差，不要直接换模型。
检查：
1. Input
用户输入是否变化。
2. Context
最终 Context 是否变化。
3. Prompt
版本是否变。
4. Tool Schema
description 是否改坏。
5. Tool Result
外部 API 是否变化。
6. RAG
索引是否更新。
7. Memory
是否写入错误信息。
8. Model
是否切换版本或配置。

## 18.9 Replay

理想调试：
保存 run input
保存 state
mock tool results
固定版本
→ 重放
可以把一次线上失败转成 regression test。

## 18.10 Structured Log

使用：
log_event(
"tool_completed",
run_id="run_123",
tool="search_docs",
latency_ms=420,
)

## 18.11 Correlation ID

跨服务：
Frontend
应该传：
trace_id
run_id
user_request_id
否则日志无法串起来。

## 18.12 Trace 隐私

高风险：
用户输入
文件内容
API 参数
身份数据
Secret
内部路径
做：
redaction
access control
retention

## 18.13 实战：Trace 驱动 Debug

故意制造 5 种失败：
1. Tool timeout
2. Wrong tool
3. RAG no result
4. Approval reject
5. Loop detected
目标：
不看源码，仅凭 Trace 能定位失败阶段。
如果做不到，你的 observability 还不够。
