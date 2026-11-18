# Agent UI 与 Human-in-the-loop（手册节选）

> 来源：《大模型 Agent 实战从 0 到 1 学习手册》第 14 篇：Agent UI 与 Human-in-the-loop。以下为原文节选，按小节重排。

## 14.1 为什么聊天框不够

传统 Chat：
用户问
→ 等
→ Agent 回
长 Agent：
运行 5 分钟
用户必须知道发生了什么。

## 14.2 UI 应显示哪些状态

建议：
Queued
Planning
Running
Waiting for approval
Waiting for user input

## 14.4 SSE 事件示例

Server：
event: run.started
data: {"run_id":"r1"}
event: tool.started
data: {"tool":"search_docs"}
event: tool.completed

## 14.6 Approval 不是再问一次模型

❌错误：
模型说这是安全的
→ 自动批准
Approval Policy 应由程序定义：

## 14.7 Pause / Resume

等待审批期间：
不占用一个 HTTP worker
不一直持有内存对象
可以服务重启后继续

## 14.8 Editable Approval

比 Yes/No 更强。
例如 Agent 提议：
{
"tool": "send_notification",
"audience": "all_members"
}
用户修改：
{
"audience": "core_members"
}
💛 Runtime 应把修改后的参数作为新 Action，重新校验，再执行。

## 14.9 Cancel

UI 中的 Cancel 必须真的传播：
Frontend
→ API
→ Run State
→ Worker
→ Active Tool / Subagent
如果只是把按钮隐藏，任务仍在后台烧 token。

## 14.10 AG-UI

当前 AG-UI 的定位是开放、轻量、事件驱动的 Agent↔Frontend 协议。
💛 它关注：
streaming
state
tool events
HITL
generative UI
agent steering
你不一定必须采用 AG-UI，但它代表一个重要趋势：
Agent 前后端需要比传统 REST request/response 更丰富的事件协议。

## 14.11 实战：Agent 运行面板

做一个最小前端：
左栏
当前任务
Plan
Todo
中栏：
右栏
Tool Timeline
Artifacts
Cost
底部：
Cancel
💛 验收：
Tool Call 实时显示
审批能真正暂停
刷新页面不丢 run
Cancel 后 worker 停止
Completed 后 artifact 可下载
