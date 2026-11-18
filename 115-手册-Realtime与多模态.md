# Realtime 与 Multimodal Agent（手册节选）

> 来源：《大模型 Agent 实战从 0 到 1 学习手册》第 15 篇：Realtime 与 Multimodal Agent。以下为原文节选，按小节重排。

## 15.1 Realtime 的关键是“低延迟双向状态”

❌语音 Agent 不只是：
录音
→ STT
→ Chatbot
→ TTS
✅真正 Realtime Agent 需要：
持续输入
实时输出
turn detection
interruption
tool call
state

## 15.2 Barge-in

用户打断：
Agent：
“我找到三个场地，第一个是 A201……”
User：
“只看能坐 150 人以上的。”
🔥 系统需要：
1. 停止当前输出
2. 捕获新输入
更新任务约束
放弃旧分支
继续
如果不能打断，体验会非常像“语音版电话菜单”。

## 15.3 Voice Agent 的延迟预算

端到端延迟：
capture
+ network
+ inference
+ tool
camera
假设：
模型 300ms
工具 4s
那换更快模型也救不了整体体验。
所以 Trace 要分层测：
audio_to_first_event
model_first_token

## 15.4 实时 Tool Calling

用户：
“帮我查一下 A201 晚上有没有空。”
Realtime Agent：
UI 可以显示：
正在查询场地……
不要让用户面对 3 秒无声。

## 15.5 Multimodal Agent

输入可能是：
text
image
audio
PDF
spreadsheet
screen

## 15.6 不要把所有模态都“先 OCR 成文本”

例如 Excel：
直接 OCR 截图
会丢掉：
单元格结构
数值类型
公式
多 sheet
更好的方式：
用文件解析工具
→ 保留结构
→ 模型只看需要部分
图片则更适合视觉模型。

## 15.7 Multimodal State

设备故障 Agent：
{
"device_id": "lab_pc_03",
"symptom": "screen flickering",
"image_artifact": "artifact_img_102",
"current_step": "cable_check",
"previous_checks": [
"power_ok"
]
}
State 不应该只是：
一段长聊天

## 15.8 Google ADK Live

当前 ADK 也提供 Live Agent / Streaming 方向，可支持实时 voice/video 场景。
开发时重点不是死记 API，而是掌握：
session
events
tools
turn detection
client transport

## 15.9 Realtime 安全

语音用户可能说：
“是的，确认删除。”
高风险系统还要考虑：
当前说话者是否是授权用户
UI 是否清晰展示动作
是否要二次确认
是否有 audit
语音便利不能绕过安全策略。

## 15.10 实战：协会语音问答 Agent

第一版只做读操作：
查活动
查场地
查协会制度
❌不允许：
第二版再加入 Approval UI。
🔥 验收：
用户可打断
Tool 调用有反馈
3 次连续对话保持 Session
不能因为网页/文档中的指令执行写操作
网络断开可明确失败
