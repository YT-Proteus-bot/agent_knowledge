# Workspace、Sandbox 与 Agent Harness（手册节选）

> 来源：《大模型 Agent 实战从 0 到 1 学习手册》第 8 篇：Workspace、Sandbox 与 Agent Harness。以下为原文节选，按小节重排。

## 8.1 为什么现代 Agent 不能只靠聊天上下文

Coding / Research Agent 会产生大量中间产物：
代码
搜索结果
日志
测试输出
如果每一轮都塞进 Context：
成本越来越高
↓
重要信息越来越难找
↓
上下文溢出
因此需要 Workspace。

## 8.2 Workspace 是外部工作记忆

/workspace
├── task.md
├── todo.md
├── sources/
├── notes/
├── src/
├── tests/
└── outputs/
Agent 可以：
ls
search
read
write
edit
而不是每轮把所有文件全文复制给模型。

## 8.3 Context 和 Workspace 的区别

Context
Workspace
Context
Workspace
本轮模型直接看到
模型按需访问
本轮模型直接看到
模型按需访问
token 成本
磁盘/对象存储成本
token 成本
磁盘/对象存储成本
容量有限
可大很多
容量有限
可大很多
适合当前信息
适合中间产物
调用结束可能消失
可持久化

## 8.4 Sandbox 为什么必要

如果 Agent 能执行：
python script.py
pip install ...
rm ...
curl ...
它已经不是普通聊天应用，而是一个“能执行任意程序的自动化系统”。
Sandbox 的作用：
📚 隔离文件系统
隔离进程
限制 CPU
限制内存
限制磁盘
限制网络
限制 secret

## 8.6 Docker 不等于“默认安全”

简单：
docker run python:3.12 python script.py
不代表万事大吉。
生产需要考虑：
💡 mount
root user
capabilities
network
seccomp
resource limit
Docker daemon 权限
示例思路：
docker run --rm \
--memory=512m \
--cpus=1 \
这是教学示例，不等于完整生产 Sandbox。

## 8.7 OpenAI Sandbox Agents 的意义

2026 年 OpenAI Agents SDK 已提供 Sandbox Agents。
官方思路包括：
🐵 persistent workspace
Manifest
file staging
sandbox-native capabilities
shell / filesystem work
snapshots
resume
一个重要安全点：
UnixLocalSandboxClient 适合可信本地开发，但在 Linux 上它本身不提供 OS 级隔离；不可信命令应该选择配置恰当的 Docker 或 hosted sandbox。
因此名字叫 “sandbox client” 不代表每个 backend 的隔离强度一样。

## 8.8 Agent Harness 是什么

Harness 可以理解为：
围绕模型循环，提供让 Agent 能长期完成真实任务的一整套运行时脚手架。

## 8.9 Harness 解决的典型问题

简单 Agent：
模型
+ 5 tools
当任务变成 40 分钟：
🌰 Todo 在哪里？
旧上下文怎么压缩？
大结果放哪里？
子任务怎么委派？
中断如何恢复？
shell 如何隔离？
approval 如何做？
运行轨迹如何记录？
这就是 Harness 层出现的原因。

## 8.10 Deep Agents

Deep Agents 当前定位就是一个 Agent Harness。
其思路强调：
💡 filesystem
subagents
memory
skills
context management
code execution / sandbox
long-running work
因此它与 LangGraph 的关系更像：
LangGraph：底层 orchestration/runtime primitives
Deep Agents：更完整的 agent harness

## 8.11 Microsoft Agent Framework Harness

当前官方 Harness 提供的能力包括：
🥖 planning / execution modes
todo list
context compaction
file memory
file access
approval
session
optional looping
这说明 Harness 已经不是单一厂商词汇，而是一个越来越清晰的工程层。

## 8.12 实战：设计自己的 Mini Harness

即使不实现完整框架，也可以做：
mini_harness/
├── runtime.py
├── state.py
├── context.py
├── tools.py
├── approvals.py
├── workspace.py
└── trace.py
Runtime：
class MiniHarness:

## 8.13 Workspace Tool

from pathlib import Path
ROOT = Path("./workspace").resolve()
def safe_path(relative: str) -> Path:
path = (ROOT / relative).resolve()
if ROOT not in path.parents and path != ROOT:
raise PermissionError("path escapes workspace")
return path
def read_file(path: str) -> str:
target = safe_path(path)
return target.read_text(encoding="utf-8")
这只是路径边界，不是完整 sandbox。

## 8.14 实战任务：文件研究 Agent

给 Agent：
20 份 Markdown 技术资料
要求：
🎨 1. 不把全部文件一次塞入 Context。
先 list_files。
再 search_files。
只读相关区段。
中间笔记写入 notes.md。
最终报告写入 report.md。
记录每次文件读取。
最大 20 个 model turns。
验收：
报告正确度
