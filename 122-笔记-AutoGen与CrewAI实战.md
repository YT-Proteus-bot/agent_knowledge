# 主流 Agent 开发框架对比（AutoGen / CrewAI / MetaGPT）（笔记节选）

> 来源：《大模型 AI Agent 知识从 0-1 笔记》第 38～44 页。以下为原文节选，已去掉行号与图像占位。

## 主流 Agent 开发框架对比（AutoGen / CrewAI / MetaGPT）

代码块
# 问题:不同 Agent 给出不同的答案
Agent1: " 这个方案可行 "
Agent2: " 这个方案有⻛险 "
Agent3: " 这个方案成本太高 "
# 解决:投票或仲裁机制
class ConflictResolver:
def resolve(self, opinions):
# 方法 1 :投票
votes = {"agree": 0, "disagree": 0}
for opinion in opinions:
if opinion["stance"] == "positive":
votes["agree"] += opinion["confidence"]
else:
votes["disagree"] += opinion["confidence"]
if votes["agree"] > votes["disagree"]:
return " 采纳方案 "
else:
return " 否决方案 "
# 方法 2 :专家仲裁
expert = ExpertAgent()
final_decision = expert.judge(opinions)
return final_decision

## 7. 主流Agent开发框架对比（1/10）

7.1框架总览对比
7.2LangChain深度解析
核心概念
代码块
from langchain.agents import create_react_agent, AgentExecutor
from langchain_openai import ChatOpenAI
from langchain.tools import Tool
from langchain.prompts import PromptTemplate
# 1. LLM :大脑
llm = ChatOpenAI(model="gpt-4", temperature=0)
# 2. Tools :工具
tools = [
Tool(
name="Search",
func=search_function,
description=" 搜索互联网信息 "
),
Tool(
name="Calculator",
func=calculator_function,
description=" 执行数学计算 "
),
]
# 3. Prompt :指令模板
prompt = PromptTemplate.from_template("""
你是一个 AI 助手。使用以下工具来回答问题:
{tools}
格式:
Question: {input}
Thought: 思考过程
Action: 工具名称
Action Input: 工具输入
Observation: 工具输出
... ( 重复 )

## 7. 主流Agent开发框架对比（2/10）

Thought: 我现在知道答案了
Final Answer: 最终答案
Question: {input}
{agent_scratchpad}
""")
# 4. Agent :组装
agent = create_react_agent(llm, tools, prompt)
# 5. Executor :执行器
agent_executor = AgentExecutor(
agent=agent,
tools=tools,
verbose=True,
max_iterations=5,
)
# 6. 执行
result = agent_executor.invoke({"input": "2024 年 AI Agent 市场规模是多少? "})
LangChain的优势
代码块
# 优势 1 :丰富的集成
from langchain.vectorstores import Chroma
from langchain.embeddings import OpenAIEmbeddings
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.document_loaders import WebBaseLoader
# 一站式 RAG 方案
loader = WebBaseLoader("https://example.com/article")

## 7. 主流Agent开发框架对比（3/10）

documents = loader.load()
text_splitter = RecursiveCharacterTextSplitter(chunk_size=1000)
chunks = text_splitter.split_documents(documents)
vectorstore = Chroma.from_documents(
documents=chunks,
embedding=OpenAIEmbeddings()
)
# 优势 2 :灵活的链式组合
from langchain.chains import LLMChain
chain = (
prompt
| llm
| output_parser
)
# 优势 3 :强大的记忆管理
from langchain.memory import ConversationBufferWindowMemory
memory = ConversationBufferWindowMemory(k=5)  # 保留最近 5 轮对话
7.3AutoGen多Agent协作
AutoGen的特色
代码块
from autogen import ConversableAgent, GroupChat, GroupChatManager
# 定义多个 Agent
product_manager = ConversableAgent(
name="PM",
system_message=" 你是产品经理,负责需求分析和产品规划。 ",

## 7. 主流Agent开发框架对比（4/10）

llm_config={"model": "gpt-4"},
)
engineer = ConversableAgent(
name="Engineer",
system_message=" 你是工程师,负责技术实现。 ",
llm_config={"model": "gpt-4"},
)
designer = ConversableAgent(
name="Designer",
system_message=" 你是设计师,负责 UI/UX 设计。 ",
llm_config={"model": "gpt-4"},
)
# 创建群聊
group_chat = GroupChat(
agents=[product_manager, engineer, designer],
messages=[],
max_round=10,
)
# 管理者
manager = GroupChatManager(
groupchat=group_chat,
llm_config={"model": "gpt-4"},
)
# 开始协作
product_manager.initiate_chat(
manager,
message=" 我们需要开发一个 AI 聊天应用,大家讨论一下方案。 ",
)
# 输出示例:
#PM: " 我建议使用 WebSocket 实现实时通信 ..."
#Engineer: " 技术上可行,我可以用 FastAPI+WebSocket 实现 ..."

## 7. 主流Agent开发框架对比（5/10）

#Designer: "UI 应该简洁,参考 Slack 的设计 ..."
#PM: " 同意,我们先做 MVP..."
#...
AutoGen的并行执行
代码块
# 多个 Agent 同时工作
from autogen import UserProxyAgent
user_proxy = UserProxyAgent(
name="User",
human_input_mode="NEVER",  # 不需要人工输入
code_execution_config={"work_dir": "workspace"},
)
# 并行任务
tasks = [
" 搜索 2024 年 AI 发展趋势 ",
" 分析主要竞争对手 ",
" 设计产品架构 ",
]
import asyncio
async def parallel_execution():
results = await asyncio.gather(*[
agent.a_generate_reply({"content": task})
for task in tasks
])
return results
7.4CrewAI⻆色扮演框架
CrewAI的团队概念
代码块
from crewai import Agent, Task, Crew
# 定义⻆色
researcher = Agent(
role="Research Analyst",
goal=" 发现 AI Agent 领域的最新趋势 ",

## 7. 主流Agent开发框架对比（6/10）

backstory=" 你是一个经验丰富的 AI 研究分析师 ...",
tools=[search_tool],
verbose=True,
)
writer = Agent(
role="Content Writer",
goal=" 撰写吸引人的技术文章 ",
backstory=" 你是一个技术作家,擅⻓将复杂概念简化 ...",
tools=[],
verbose=True,
)
# 定义任务
task1 = Task(
description=" 研究 2024 年 AI Agent 的市场趋势 ",
agent=researcher,
expected_output=" 详细的市场分析报告 ",
)
task2 = Task(
description=" 基于研究结果撰写一篇博客文章 ",
agent=writer,
expected_output="1500 字的博客文章 ",
)
# 组建团队
crew = Crew(
agents=[researcher, writer],
tasks=[task1, task2],
verbose=True,
process="sequential",  # 顺序执行
)
# 启动
result = crew.kickoff()
7.5Dify可视化平台
Dify的工作流编排
代码块
[ 用戶输入 ]
↓
[LLM 节点 : 理解意图 ]
↓
[ 条件分支 ]

## 7. 主流Agent开发框架对比（7/10）

├─  需要搜索  →  [ 搜索节点 ] →  [LLM 节点 : 总结 ]
└─  不需要搜索  →  [LLM 节点 : 直接回答 ]
↓
[ 输出节点 ]
Dify的优势:
✅ 拖拽式设计,无需代码
✅ 内置RAG、Agent、工作流模板
✅ 可视化调试和监控
✅ 一键部署API
适用场景:
快速原型验证
业务人员使用
低代码场景
7.6框架选择指南
代码块
def choose_framework(scenario):
if scenario == " 学习和探索 ":
return "LangChain - 生态最完善,教程最多 "
elif scenario == " 多 Agent 协作 ":
return "AutoGen - 专为多 Agent 设计 "
elif scenario == " 团队流程模拟 ":
return "CrewAI - ⻆色扮演最自然 "
elif scenario == " 快速原型 ":
return "Dify - 可视化,上手快 "
elif scenario == " 国内部署 ":
return "LazyLLM - 中文优化,商汤支持 "
elif scenario == " 生产级应用 ":
return "LangChain + 自定义优化 "
else:
return "LangChain - 最保险的选择 "
8. Agent实战案例与代码实现
8.1案例一:智能客服Agent
需求分析

## 7. 主流Agent开发框架对比（8/10）

代码块
场景:电商平台的客服系统
功能要求:
回答常⻅问题( FAQ )
查询订单状态
处理退换货
转人工客服(复杂问题)
技术挑战:
需要访问数据库(订单系统)
需要记忆上下文(多轮对话)
需要情感识别(判断用戶情绪)
完整实现
代码块
from langchain.agents import create_react_agent, AgentExecutor
from langchain_openai import ChatOpenAI
from langchain.tools import Tool
from langchain.memory import ConversationBufferMemory
import sqlite3
# 1. 定义工具
def query_order(order_id: str) -> str:
""" 查询订单状态 """
conn = sqlite3.connect('ecommerce.db')
cursor = conn.cursor()
cursor.execute(
"SELECT status, items, total FROM orders WHERE order_id = ?",
(order_id,)
)
result = cursor.fetchone()
conn.close()
if result:
status, items, total = result

## 7. 主流Agent开发框架对比（9/10）

return f" 订单状态:{status},商品:{items},总金额:{total}元 "
else:
return " 未找到该订单 "
def search_faq(question: str) -> str:
""" 搜索 FAQ 知识库 """
faq_db = {
" 退货 ": " 退货政策: 7 天无理由退货,商品需保持完好 ...",
" 发货 ": " 发货时间:工作日下单当天发货,节假日顺延 ...",
" 支付 ": " 支付方式:支持微信、支付宝、信用卡 ...",
}
# 简单的关键词匹配
for key, answer in faq_db.items():
if key in question:
return answer
return " 未找到相关 FAQ ,建议转人工客服 "
def detect_emotion(text: str) -> str:
""" 检测用戶情绪 """
negative_words = [" 生气 ", " 不满意 ", " 糟糕 ", " 差 ", " 垃圾 "]
for word in negative_words:
if word in text:
return "negative"
return "neutral"
# 2. 创建工具列表
tools = [
Tool(
name="QueryOrder",
func=query_order,
description=" 查询订单状态。输入订单号,返回订单详情。 "
),

## 7. 主流Agent开发框架对比（10/10）

Tool(
name="SearchFAQ",
func=search_faq,
description=" 搜索常⻅问题答案。输入问题关键词。 "
),
Tool(
name="DetectEmotion",
func=detect_emotion,
description=" 检测用戶情绪。输入用戶消息文本。 "
),
]
# 3. 创建客服 Agent
llm = ChatOpenAI(model="gpt-4", temperature=0.7)
