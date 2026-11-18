# 主流 Agent 框架（手册节选）

> 来源：《大模型 Agent 实战从 0 到 1 学习手册》第 11 篇：主流 Agent 框架。以下为原文节选，按小节重排。

## 概述

这一篇不是“框架排行榜”。框架选型的正确问题是：
我的系统最需要哪一种抽象？
你首先要知道框架分别帮你解决哪一层。

## 11.1 先把框架放进技术栈

不同框架强调的层不同。
🏆 本篇重点：
OpenAI Agents SDK
LangGraph
Deep Agents
Google ADK 2.0
Microsoft Agent Framework
其他框架作为补充认识，而不是每个都学一遍。

## 11.2 OpenAI Agents SDK：少量原语的 Agent Runtime（1/2）

🎨 截至本文时间，官方把核心 Agent 定义描述为：
instructions
tools
handoffs
guardrails
structured outputs
runtime behavior
并通过 Agent + Runner 管理循环。
📍 同时还有：
Sessions
Human-in-the-loop
Tracing
MCP
Realtime
Sandbox Agents
最小示例
pip install openai-agents
from agents import Agent, Runner
agent = Agent(
name="Association Assistant",
instructions=(
"You help members of a computer association. "
"Be accurate and clearly mark unknown facts."
加 Tool
from agents import Agent, Runner, function_tool
@function_tool
Agents as Tools
一个 Manager 调另一个 Agent：
Manager
→ Research Agent
→ Manager
→ Final
这种模式里 Manager 保持控制权。
Handoff
Handoff 更像：
当前 Agent
→ 把后续会话交给 Specialist
区别一定要掌握：

## 11.2 OpenAI Agents SDK：少量原语的 Agent Runtime（2/2）

Agent-as-Tool：父 Agent 仍是总控
Handoff：会话控制权转移
Session
适合维护多轮历史。
但注意：
Session 主要是会话历史管理，不应该自动等同于你业务上的长期用户 Memory。
Tracing
🌅 SDK 当前内置 tracing，可记录：
model generation
tool calls
handoffs
guardrails
custom events
这对学习 Agent 执行过程非常有价值。
Sandbox Agents
2026 年值得重点关注。
🏕️ 它把：
workspace
file staging
filesystem
shell
filesystem
shell
sandbox lifecycle
snapshot / resume
进一步整合进 Agent 运行方式。
何时选 OpenAI Agents SDK
🥛 适合：
想要少抽象
Tool Agent
Handoff
OpenAI 模型/生态
Sandbox / Realtime
想快速从手写 Loop 迁移到 SDK
注意
不要把业务逻辑全部写进：
instructions
或者：
@function_tool
业务逻辑仍应独立成 service/domain 层。

## 11.3 LangGraph：显式状态图与 Durable Orchestration

如果你的系统更像：
状态
→ 节点
→ 条件路由
→ 中断
→ 恢复
LangGraph 的抽象很自然。
核心心智模型
🎨 你显式定义：
State
Node
Edge
Conditional Edge
为什么对生产系统友好
很多业务流程本来就有固定部分：
权限检查
→ Agent 分析
→ 人工审批
💡 这种流程用图表达比“让 Agent 自己随便循环”更容易：
Debug
Review
Test
Audit
最小骨架
这个例子故意不用 LLM，因为它展示一个重要理念：
Graph 中的节点不必都是模型。
🌰 适合 LangGraph 的场景
固定流程中夹 Agent
中断 / resume
durable workflow
适合 LangGraph 的场景
固定流程中夹 Agent
中断 / resume
durable workflow
state 很重要
审批复杂
多分支
希望显式控制执行拓扑

## 11.4 Deep Agents：Agent Harness

Deep Agents 更接近：
“给我一套适合长任务 Agent 的完整运行脚手架”
而不是只提供图原语。
🌟 其典型能力包括：
file system
context management
memory
skills
subagents
code/sandbox integration
long-running task support
心智模型
示例骨架
API 会随版本演进，当前官方思路可概括为：
实际使用时以当前版本官方 quickstart 为准。
📌 适合
Deep Research
Coding Agent
大量文件
长任务
子 Agent
Skill 驱动
适合
Deep Research
Coding Agent
大量文件
长任务
子 Agent
Skill 驱动

## 11.5 Google ADK 2.0（1/2）

2026 年 ADK 的一个重要变化是 2.0 Workflow Runtime。
🏆 官方 2.0 重点：
Graph-based workflows
Dynamic workflows
Collaborative workflows
Python / TypeScript / Go 2.0
也就是说，它不只是在“定义一个 LLM Agent”，而是在往完整的工作流运行时发展。
安装
pip install google-adk
最小 Agent
官方 Python quickstart 风格：
from google.adk.agents.llm_agent import Agent
def get_current_room(room_id: str) -> dict:
"""Return example room information."""
return {
"status": "success",
"room_id": room_id,
"available": True,
}
root_agent = Agent(
开发运行通常可：
adk run my_agent
或者：
adk web
但官方明确指出开发 Web UI 是开发/调试用途，不等于生产部署界面。
2.0 的工程意义
ADK 2.0 从层级 Agent executor 进一步转向 graph-based execution engine。
因此学习时要注意：
老 1.x 教程
Tool Error

## 11.5 Google ADK 2.0（2/2）

2.0 官方迁移说明特别提醒：
不要在 Tool 里用过宽的：
except Exception:
...
然后吞掉所有异常。
✍️
原因是 Runtime 需要知道节点失败，才能做：
retry
interrupt
error handling
更不要随意捕获 BaseException。
这是很好的通用工程原则。
🦄 适合
Google Cloud
Graph Workflow
A2A
适合
Google Cloud
Graph Workflow
A2A
Live Agent
多语言 SDK
完整 Agent 生命周期

## 11.6 Microsoft Agent Framework

当前官方学习路线已经很清晰：
Agent
→ Tools
→ Sessions
→ Memory / Persistence
→ Workflows
说明它不是只做“一个聊天 Agent”，而是把 Agent 和 Workflow/Harness 统一考虑。
Harness
Harness 的官方定位是给长、多步骤任务提供运行时脚手架。
📍 包括：
planning / execution mode
todo
context compaction
file memory
file access
approval
session
optional loop
observability
Python 骨架
from agent_framework import create_harness_agent
from agent_framework.openai import OpenAIChatClient
client = OpenAIChatClient(
model="your-model",
一个重要设计
官方把：
Harness Instructions
和：
Agent Instructions
分开。
前者更像运行方式：
后者更像角色任务：
你是研究 Agent
这个分层值得借鉴。

## 11.7 框架选型不是“选一个永远不换”

建议先按核心需求：
核心需求
优先研究
核心需求
优先研究
轻量 Tool/Handoff Agent
OpenAI Agents SDK
轻量 Tool/Handoff Agent
OpenAI Agents SDK
显式状态图
LangGraph
显式状态图
LangGraph
长任务 Harness
Deep Agents
长任务 Harness
Deep Agents
Google 生态/Workflow Runtime
Google ADK 2.0
Google 生态/Workflow Runtime
Google ADK 2.0
Microsoft 生态/Workflow/Harness
Microsoft Agent Framework
Microsoft 生态/Workflow/Harness
Microsoft Agent Framework
🍞 然后问：
是否支持我的模型
是否支持 self-host
状态怎么持久化
怎么做 HITL
Trace 怎么导出
Tool 权限怎么控制
能否自定义 runtime
版本升级风险

## 11.8 实战：同一个任务用两种框架实现

任务：
用户给一个技术主题
→ 搜索资料
→ 整理三条结论
→ 输出来源
版本 A：
OpenAI Agents SDK
版本 B：
LangGraph
🦄 比较：
代码量
state 可见性
trace
resume
tool integration
学习成本
通过这个练习，你会真正理解“抽象不同”，而不是背文档。

## 11.9 框架无关的业务层

推荐：
src/
├── domain/
│ ├── events.py
│ └── rooms.py
├── services/
│ ├── event_service.py
│ └── room_service.py
├── agent/
│ ├── tools.py
Tool Adapter：
def get_room_status_tool(room_id: str):
return room_service.get_status(room_id)
不要让框架侵入所有业务代码。
