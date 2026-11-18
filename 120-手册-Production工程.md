# Production Agent Engineering（手册节选）

> 来源：《大模型 Agent 实战从 0 到 1 学习手册》第 20 篇：Production Agent Engineering。以下为原文节选，按小节重排。

## 20.1 从 Demo 到 Production

Demo：
生产：
为什么多这些层？
因为生产会遇到：
长任务
并发
重试
超时
用户取消
服务重启
审批
成本
安全
版本升级

## 20.2 推荐项目结构

app/
├── api/
│ ├── routes.py
核心原则：
Agent Framework ≠ Domain Layer

## 20.3 API 设计

长任务不要：
HTTP
然后连接挂 20 分钟。
更适合：
HTTP
POST /runs
→ 202 Accepted
→ {"run_id":"r_123"}
再：
HTTP

## 20.5 数据库 Schema 示例

Approvals：
CREATE TABLE approvals (
approval_id TEXT PRIMARY KEY,
run_id TEXT NOT NULL,
action_type TEXT NOT NULL,
action_payload TEXT NOT NULL,
status TEXT NOT NULL,

## 20.6 Queue 和 Worker

优势：
📚 API 快速返回
worker 可扩容
失败可重试
长任务不占 HTTP connection

## 20.7 Worker 必须幂等

Queue 可能出现重复 delivery。
任务：
publish_event
如果 Worker 重跑，不能发布两次。
所以每个 Side Effect：
idempotency_key

## 20.8 Cancellation

用户点击取消：
长 Tool 也应该支持 cooperative cancellation，至少能 timeout。

## 20.9 Timeout 分层

不要只一个 timeout。
Model Timeout
例如：
60s
Tool Timeout
不同 Tool：
DB 3s
HTTP 10s
Code Test 120s
Run Timeout
整个任务：
Approval Timeout
可能：
24h
各层意义不同。

## 20.10 Budget

对每个 Run：
from dataclasses import dataclass
@dataclass
class RunBudget:
max_turns: int
max_tool_calls: int
max_seconds: int
max_cost_usd: float
max_subagents: int
Budget 不是“优化后再做”的东西，而是 Agent 的基本控制面。

## 20.11 Cost Observability

至少记录：
run_id
model
input_tokens
output_tokens
cached_tokens
tool_calls
subagent_calls
estimated_cost
再按：
user
team
agent
feature
day
聚合。

## 20.13 Graceful Degradation

如果：
Web Search 挂了
系统可以：
明确说明无法获取最新信息
而不是：
编造一个最新答案
如果：
Reranker 挂了
可以降级：
具体看风险。

## 20.14 Circuit Breaker

外部 API 连续失败：
Circuit Breaker：
Closed
→ failure threshold
→ Open
→ cooldown
→ Half-open

## 20.15 Rate Limiting

限制：
per user
per organization
per tool
per model
避免：
abuse
成本爆炸
外部 API 被打挂

## 20.16 Cache

适合：
embedding
静态 metadata
工具列表
不常变化规则
谨慎：
权限
实时库存
审批状态
错误缓存实时数据会让 Agent“非常稳定地答错”。

## 20.17 Version Everything

一次 Run 至少知道：
code_commit
agent_version
prompt_version
model
tool_schema_version
skill_version
否则线上出现：
昨天还能跑，今天不行
你连复现环境都没有。

## 20.18 Prompt Version

不要在代码里：
SYSTEM_PROMPT = """..."""
然后改完直接覆盖。
至少：
event_agent_prompt_v12
Trace 里记录版本。

## 20.19 Tool Schema Version

Tool 改参数：
v1:
create_event(title, date)
v2:
create_event(title, start_at, timezone)
长期任务恢复时，旧 checkpoint 可能仍引用 v1。
所以 schema evolution 也要设计。

## 20.20 Database Migration 与 Agent State Migration

Agent state：
{
"version": 2,
"phase": "review"
}
新代码要求：
{
"version": 3,
"phase": "approval",
需要 migration，而不是假设所有正在跑的任务瞬间消失。

## 20.21 FastAPI Skeleton

from fastapi import FastAPI
from pydantic import BaseModel
from uuid import uuid4
app = FastAPI()
这只是 API 层。
不要在 handler 里直接执行 15 分钟 Agent。

## 20.22 Health Checks

区分：
Liveness
服务进程还活着吗？
Readiness
依赖是否可用：
DB
Queue
但不要每次 readiness 都真的调用昂贵 LLM。

## 20.24 CI 中应该测什么

Unit
Tool validator
State transition
permission
memory store
Integration
MCP Server
DB
Queue
Sandbox
Agent Eval
task success
tool selection
RAG
safety
Security
injection
path traversal
unauthorized write

## 20.25 Deployment

典型组件：
不要为了“现代”一上来 Kubernetes。
小项目可以：
能承载需求就够。

## 20.26 Scaling

Agent Worker 常见瓶颈：
model provider rate limit
external API
sandbox creation
queue
DB connections
不是只看 CPU。

## 20.27 SLO

根据业务定义。
示例：
simple task success >= 98%
write action unauthorized execution = 0
p95 first progress event <= 2s
p95 simple run <= 20s
tool failure rate <= 1%
不要直接抄数字，先建立基线。

## 20.28 Incident Response

Agent 出事故：
1. disable high-risk tools
2. revoke credential if needed
3. stop active runs
4. preserve traces
5. identify affected actions
6. rollback prompt/model/tool version
7. add regression test
8. write postmortem

## 20.29 最终 Production Checklist

Product
☐ 目标任务清晰
☐ 有“不适合 Agent”的边界
☐ 用户能看到 Agent 正在做什么
Runtime
☐ max turns
☐ timeout
☐ cancellation
☐ checkpoint / resume
☐ budget
Tools
☐ schema validation
☐ timeout
☐ retry
☐ idempotency
☐ least privilege
Knowledge
☐ source attribution
☐ permissions
☐ retrieval eval
☐ freshness
Security
☐ prompt injection defenses
☐ secret isolation
☐ sandbox
☐ approval
☐ audit
Quality
☐ eval dataset
☐ regression gate
☐ trace
☐ metrics
Operations
☐ version everything
☐ alerting
☐ rollback
☐ kill switch
☐ incident playbook
