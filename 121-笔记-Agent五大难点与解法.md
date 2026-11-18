# 构建 Agent 的五大难点与解决方案（笔记节选）

> 来源：《大模型 AI Agent 知识从 0-1 笔记》第 21～29 页。以下为原文节选，已去掉行号与图像占位。

## 构建 Agent 的五大难点与解决方案

这个视⻆帮助你:
设计Agent的职责范围
定义输入输出格式
考虑错误处理
视⻆2:把Agent看作「循环系统」
代码块
输入  →  [ 感知  →  思考  →  决策  →  行动  →  观察 ] →  输出
↑ _______________________________|
反馈循环
这个视⻆帮助你:
优化循环次数
设置终止条件
调试中间过程
视⻆3:把Agent看作「大脑+工具」
代码块
大脑( LLM ):
理解语言
推理规划
生成文本
↕  通信
工具箱:
搜索引擎
计算器
API 接口
数据库
这个视⻆帮助你:
扩展Agent能力(添加新工具)
优化工具选择(描述要精确)
提升执行效率(工具要快)


## 5. 构建Agent的五大难点与解决方案（1/2）

5.1难点一:无限循环与任务卡死
问题描述
Agent可能陷入死循环:
代码块
Thought: 我需要搜索信息
Action: Search("2024 AI")
Observation: [搜索结果]
Thought: 我需要搜索更详细的信息
Action: Search("2024 AI 详细 ")
Observation: [搜索结果]
Thought: 我需要搜索更详细的信息
Action: Search("2024 AI 更详细 ")
... (永远循环)
解决方案
方案1:设置最大迭代次数
代码块
agent_executor = AgentExecutor(
agent=agent,
tools=tools,
max_iterations=10,  # 最多执行 10 轮
max_execution_time=60,  # 最多执行 60 秒
early_stopping_method="generate"  # 超时时生成答案
)
方案2:优化提示词,明确终止条件
代码块
prompt = """
你必须在 5 步内完成任务。
如果 5 步内无法完成,请给出 " 基于现有信息的最佳答案 " 。
当前已经执行了 {iterations} 步。
"""
方案3:实现智能终止判断
代码块
def should_continue(agent_state):
# 检查是否在重复相同的行动

## 5. 构建Agent的五大难点与解决方案（2/2）

last_3_actions = agent_state.history[-3:]
if len(set(last_3_actions)) == 1:  # 最后 3 个动作都相同
return False  # 终止
# 检查是否进展缓慢
if agent_state.iterations > 5 and not has_new_info(agent_state):
return False
return True
5.2难点二:工具选择错误
问题描述
代码块
Agent 选择了错误的工具:
任务:" 计算 2024 年有多少天 "
错误选择:
Action: WebSearch("2024 年有多少天 ")  ❌  不需要搜索
正确选择:
Action: Calculator("365 if not is_leap_year(2024) else 366")  ✅
根本原因

## 1. 工具描述不清晰（1/2）

2. Prompt没有给出使用规则
3. LLM对任务理解有偏差
解决方案
方案1:改进工具描述
代码块
❌  不好的描述
calculator_tool = Tool(
name="Calculator",
description=" 计算 "
)
# ✅  好的描述
calculator_tool = Tool(
name="Calculator",
description="""
用于精确的数学计算。
适用场景:
- 算术运算(加减乘除)
- 数学函数(平方、开方、三⻆函数)
- 逻辑判断(比较大小)
输入格式: Python 数学表达式字符串
示例: "2 + 2", "sqrt(16)", "2024 % 4 == 0"
不适用场景:
- 需要最新数据(用 Search )
- 需要上下文信息(用 Memory )
"""
)
方案2:添加工具使用示例
代码块
prompt = """
工具使用示例:
问题: " 今天的天气如何? "
正确: Action: WeatherAPI(" 北京 ")
错误: Action: Calculator(" 天气 ")  # 天气不是数学问题
问题: "2 的 100 次方是多少? "
正确: Action: Calculator("2**100")
错误: Action: Search("2 的 100 次方 ")  # 不需要搜索
现在请处理: {input}
"""
方案3:实现工具推荐系统
代码块

## 1. 工具描述不清晰（2/2）

def recommend_tool(task_description, available_tools):
""" 根据任务描述推荐工具 """
# 使用 LLM 分析任务
analysis = llm.invoke(f"""
任务:{task_description}
请分析这个任务需要:

## 3. 历史信息吗? →  需要 Memory（1/6）

推荐工具( JSON 格式):
{{"recommended": ["tool1", "tool2"], "reason": "..."}}
""")
return analysis.recommended
5.3难点三:上下文窗口溢出
问题描述
⻓对话或复杂任务会导致:
代码块
Prompt + 历史对话  + 工具描述  + 中间结果  = 超过 Token 限制
GPT-4: 128K tokens
Claude: 200K tokens
一次复杂任务可能消耗 50K+ tokens
→  超出限制或成本过高
解决方案
方案1:智能压缩上下文
代码块
from langchain.memory import ConversationSummaryBufferMemory
memory = ConversationSummaryBufferMemory(
llm=llm,
max_token_limit=4000,  # 保留最近 4000 tokens
moving_summary_buffer="",  # 旧的对话自动总结
)
# 效果:
# 旧对话: " 用戶询问了天气、新闻、股票 ... 共 20 轮对话 "
# ↓  总结为
# " 用戶主要关心北京天气(晴)、今日新闻( AI 发展)、股票(上涨) "
方案2:分层记忆
代码块
class HierarchicalMemory:
def init(self):

## 3. 历史信息吗? →  需要 Memory（2/6）

self.recent_memory = []  # 最近 3 轮,完整保留
self.mid_term_memory = []  # 中期 10 轮,保留关键信息
self.long_term_memory = vectorstore  # ⻓期,向量检索
def add(self, message):
self.recent_memory.append(message)
if len(self.recent_memory) > 3:
# 移到中期记忆,提取关键信息
old_message = self.recent_memory.pop(0)
key_info = extract_key_info(old_message)
self.mid_term_memory.append(key_info)
if len(self.mid_term_memory) > 10:
# 移到⻓期记忆,存入向量库
old_info = self.mid_term_memory.pop(0)
self.long_term_memory.add_texts([old_info])
def get_context(self, query):
# 组合三层记忆
recent = "\n".join(self.recent_memory)
mid = "\n".join(self.mid_term_memory)

## 3. 历史信息吗? →  需要 Memory（3/6）

relevant = self.long_term_memory.similarity_search(query, k=2)
return f"{recent}\n\n 相关历史:{mid}\n{relevant}"
方案3:动态工具加载
代码块
# 不要一次性加载所有工具
all_tools = {
"search": search_tool,
"calculator": calc_tool,
"database": db_tool,
"api": api_tool,
# ... 50 个工具
}
# 根据任务动态选择工具
def select_tools(task):
# 分析任务需要哪些类别的工具
if " 搜索 " in task or " 查询 " in task:
return [all_tools["search"]]
elif " 计算 " in task:
return [all_tools["calculator"]]
# ...
# 默认返回最常用的 5 个工具
return list(all_tools.values())[:5]
# 创建 Agent 时只加载需要的工具
tools = select_tools(user_input)
agent = create_agent(llm, tools)
5.4难点四:错误处理与鲁棒性
问题描述
各种错误会导致Agent崩溃:
工具调用失败
API超时
返回格式错误
LLM输出异常
解决方案

## 3. 历史信息吗? →  需要 Memory（4/6）

方案1:工具层面的错误处理
代码块
def robust_tool(func):
""" 装饰器:让工具更健壮 """
def wrapper(*args, **kwargs):
max_retries = 3
for attempt in range(max_retries):
try:
result = func(*args, **kwargs)
return {
"success": True,
"data": result
}
except TimeoutError:
if attempt == max_retries - 1:
return {
"success": False,
"error": " 工具执行超时,请尝试简化输入或使用其他工具 "
}
time.sleep(2 ** attempt)  # 指数退避
except Exception as e:
return {
"success": False,
"error": f" 工具执行失败:{str(e)}"
}
return wrapper
@robust_tool
def search_tool(query):
# 实际搜索逻辑
return search_engine.query(query)
方案2:Agent层面的降级策略
代码块
class RobustAgent:
def execute(self, task):
try:
# 尝试完整流程

## 3. 历史信息吗? →  需要 Memory（5/6）

return self.full_execution(task)
except AgentError:
# 降级:使用简化流程
return self.simplified_execution(task)
except Exception:
# 最终降级:直接用 LLM 回答
return self.fallback_execution(task)
def full_execution(self, task):
# ReAct 完整循环,可能使用多个工具
return self.react_loop(task)
def simplified_execution(self, task):
# 简化版:只用一个最重要的工具
tool = self.select_best_tool(task)
result = tool.run(task)
return self.llm.summarize(result)
def fallback_execution(self, task):
# 保底:直接用 LLM 知识回答
return self.llm.invoke(f" 请基于你的知识回答:{task}")
方案3:实时监控与告警
代码块
import logging
class AgentMonitor:
def init(self):
self.metrics = {
"total_tasks": 0,
"success": 0,
"failures": 0,
"avg_time": 0,
}

## 3. 历史信息吗? →  需要 Memory（6/6）

def log_execution(self, task, result, duration):
self.metrics["total_tasks"] += 1
if result["success"]:
self.metrics["success"] += 1
else:
self.metrics["failures"] += 1
logging.error(f"Task failed: {task}, Error: {result['error']}")
# 计算平均时间
self.metrics["avg_time"] = (
self.metrics["avg_time"] * (self.metrics["total_tasks"] - 1) + duration
) / self.metrics["total_tasks"]
# 告警
failure_rate = self.metrics["failures"] / self.metrics["total_tasks"]
if failure_rate > 0.3:  # 失败率超过 30%
send_alert(f"Agent failure rate: {failure_rate:.2%}")
5.5难点五:成本控制
问题描述
Agent的成本可能很高:
代码块
一次复杂任务:

## 10 轮 ReAct 循环（1/4）

每轮 2K tokens 输入  + 500 tokens 输出
总计: (2000+500) * 10 = 25K tokens
GPT-4 价格(假设):
输入:$0.03 / 1K tokens
输出:$0.06 / 1K tokens
单次任务成本:
输入: 20K * 0.03 = $0.60
输出: 5K * 0.06 = $0.30
总计:$0.90
如果每天 1000 个任务  = $900/ 天  = $27,000/ 月
解决方案
方案1:模型分级使用
代码块
class CostOptimizedAgent:
def init(self):
self.models = {
"cheap": ChatOpenAI(model="gpt-3.5-turbo"),  # $0.002/1K
"standard": ChatOpenAI(model="gpt-4"),        # $0.03/1K
"premium": ChatOpenAI(model="gpt-4-turbo"),   # $0.01/1K
}
def select_model(self, task_complexity):
# 简单任务用便宜模型
if task_complexity < 3:
return self.models["cheap"]
elif task_complexity < 7:
return self.models["standard"]
else:

## 10 轮 ReAct 循环（2/4）

return self.models["premium"]
def estimate_complexity(self, task):
# 根据任务特征估计复杂度
factors = {
"num_steps": len(parse_steps(task)) * 2,
"need_tools": 3 if requires_tools(task) else 0,
"domain_specific": 2 if is_specialized(task) else 0,
}
return sum(factors.values())
方案2:缓存机制
代码块
from functools import lru_cache
import hashlib
class CachedAgent:
def init(self):
self.cache = {}
def execute(self, task):
# 计算任务的哈希值
task_hash = hashlib.md5(task.encode()).hexdigest()
# 检查缓存
if task_hash in self.cache:
logging.info(f"Cache hit for task: {task[:50]}...")
return self.cache[task_hash]
# 执行任务
result = self.agent.run(task)
# 存入缓存
self.cache[task_hash] = result

## 10 轮 ReAct 循环（3/4）

return result
# 效果:
# 第一次: " 北京明天天气 " →  调用 LLM →  成本 $0.05
# 第二次: " 北京明天天气 " →  从缓存读取  →  成本 $0
方案3:批处理
代码块
def batch_process(tasks):
""" 批量处理相似任务 """
# 将相似任务分组
groups = group_similar_tasks(tasks)
results = []
for group in groups:
# 一次性处理一组任务
batch_prompt = f"""
请处理以下{len(group)}个相似任务:
{"\n".join([f"{i+1}. {t}" for i, t in enumerate(group)])}
请输出 JSON 格式的结果列表。
"""
batch_result = llm.invoke(batch_prompt)
results.extend(parse_batch_result(batch_result))
return results
# 效果:
# 单独处理 10 个任务: 10 次 LLM 调用
# 批量处理 10 个任务: 2-3 次 LLM 调用(节省 70% 成本)
方案4:设置预算限制
代码块
class BudgetControlledAgent:
def init(self, daily_budget_usd=10):

## 10 轮 ReAct 循环（4/4）

self.daily_budget = daily_budget_usd
self.today_spent = 0
self.task_count = 0
def execute(self, task):
# 检查预算
estimated_cost = self.estimate_cost(task)
if self.today_spent + estimated_cost > self.daily_budget:
return {
"success": False,
"message": f" 预算不足。今日已用 ${self.today_spent:.2f},预算 ${self.daily_budget}"
