# Agent Skills（手册节选）

> 来源：《大模型 Agent 实战从 0 到 1 学习手册》第 9 篇：Agent Skills。以下为原文节选，按小节重排。

## 9.1 为什么需要 Skills

假设你有 20 个 Agent，都需要知道：
“协会技术分享会应该怎么策划”
最差做法：
复制 20 份 Prompt
更好的方式：
Skill 常包含：
🎹 触发描述
操作说明
SOP
示例
脚本
参考资料
模板

## 9.2 Tool 和 Skill 不是一个东西

Tool：
“我能执行什么动作？”
Skill：
“遇到这一类任务，我应该怎样做？”
例如：
Tool:
get_room_status

## 9.3 Skill 目录

skills/
└── event-planning/
├── SKILL.md

## 9.5 Progressive Disclosure

不应该在 Agent 启动时读完所有 Skills。
graph TD;
A["Agent Starts"] --> M["Load Skill Metadata"];
M --> T["Current Task"];
T --> R{"Skill Relevant"};
R -->|否| C["Continue"];
R -->|是| S["Load SKILL.md"];
S --> N{"Need Reference"};
N -->|是| F["Load Specific Reference"];
N -->|否| C;
F --> C;
这种按需加载可以显著控制 Context。

## 9.6 Skill Trigger 设计

坏描述：
Useful skill.
好描述：
Use this skill to plan or review computer association
technical events, including venue, agenda, budget,
staffing, promotion and risk checks.
Trigger 描述本身就是检索入口。

## 9.7 Skill 不应该装什么

不要：
API Key
数据库密码
用户隐私全集
几百 MB 无关文档
Secret 必须由安全的 credential 系统提供。

## 9.8 Skill 和 RAG 的区别

RAG：
根据 Query 检索知识片段
Skill：
按任务加载一套“怎么做”的流程与资源
两者可以组合：
加载 event-planning Skill
→ Skill 要求查制度
→ 调 RAG 检索 venue rules

## 9.9 Skill 和 MCP 的区别

Skill 可以告诉 Agent：
如何做活动策划
MCP Server 可以暴露：
venue_search
event_create
一个偏“工作方法”，一个偏“能力连接”。

## 9.10 实战：写一个 Skill Loader

from dataclasses import dataclass
from pathlib import Path
再加一个 metadata index：
SKILL_INDEX = [
{
"name": "event-planning",
"description": "Plan or review association events",
"path": "skills/event-planning",
},
{
"name": "python-review",
"description": "Review Python projects and code quality",
"path": "skills/python-review",
真正的 Agent 再通过语义/规则选择要加载哪个 Skill。

## 9.11 Skill 测试

Skill 也需要 Eval。
正例
“帮我策划一次 100 人的分享会”
→ 应加载 event-planning
负例：
“解释什么是 TCP”
→ 不应加载 event-planning
指标：
trigger precision
trigger recall
task success
extra token cost

## 9.12 协会 Skill Library 建议

第一批：
event-planning
technical-writing
python-project-review
web-project-review
security-check
contest-preparation
meeting-summary
new-member-onboarding
每个 Skill：
🎼 owner
version
changelog
test cases
不要把 Skill 变成没人维护的“Prompt 垃圾场”。
