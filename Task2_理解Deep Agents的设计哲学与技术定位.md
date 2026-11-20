# 理解 Deep Agents 的设计哲学与技术定位

## 一、Agent 开发三层架构

**三者关系**：自底向上层层构建，可组合使用，不是互相替代。

### Agent Runtime（运行时层）— LangGraph

解决 Agent 如何可靠运行（持久化执行、流式输出、人机协作、状态管理）。被比作 Agent 世界的"操作系统"。

### Agent Framework（框架层）— LangChain

提供开发抽象——模型抽象、工具接口、Agent 循环、中间件。构建在 LangGraph 之上，核心价值是标准化 + 易上手。

### Agent Harness（工具层）— Deep Agents

开箱即用的 Agent 套件，预置文件系统、子 Agent、任务规划、记忆等能力。构建在前两者之上。

## 二、Agent 共性能力

**文件系统操作：**&#x8BFB;写、搜索、编辑文件。缺了它就无法读取任务材料、也无法落地修改。

**任务规划：**&#x628A;大任务拆成可追踪的小步骤。缺了它就会在一个大任务下反复打转、组织不出步骤。

**子任务委派：**&#x628A;子任务交给专门的 Agent，用独立上下文处理。缺了它就要把所有工作压给单个 Agent，模型能力有限、复杂任务效果差。

**上下文管理：**&#x9632;止对话过长导致模型"失忆"。缺了它会超出上下文上限，模型开始答非所问、忘记之前做过的事。

## 三、Context Engineering

为 LLM 构建高效获取和管理信息的基础设施，而不是一次性把全部信息塞入 prompt。

### 虚拟文件系统

上下文里只留当前步骤真正需要的信息：

- read_file 按需读取；大文件用 offset / limit 只读局部
- write_file 把中间结果写到文件，需要时再取回
- grep / glob 按需搜索定位

七个文件工具：read_file、write_file、edit_file、delete、ls、glob、grep。

## 四、具体实践步骤

### 最小 Agent

配好模型后写个 `get_weather`（只返回固定测试文本），交给 `create_deep_agent()`，再 `invoke()` 提问。

核心：配模型 → 创建 Agent → 运行。

```python
model = ChatOpenAI(model=os.getenv("AGENTSEEK_MODEL"),
                   base_url=os.getenv("OPENAI_API_BASE"),
                   api_key=os.getenv("OPENAI_API_KEY"),
                   model_kwargs={"parallel_tool_calls": False})

def get_weather(city: str) -> str:
    """返回指定城市的天气（固定测试文本）。

    Args:
        city: 要查询的城市名。
    """
    return f"{city}今天晴，26 摄氏度。"

agent = create_deep_agent(model=model, tools=[get_weather], system_prompt="你是一个天气助手，回答要简短。")
result = agent.invoke({"messages": [{"role": "user", "content": "北京天气怎么样？"}]})
print(result["messages"][-1].content)
```

实跑结果：

```text
北京今天天气晴朗，气温26摄氏度。
消息序列：['human', 'ai', 'tool', 'ai']
```

`human → ai → tool → ai` 说明一次 `invoke()` 里模型先决定调工具、工具结果写回、模型再生成最终回复——这就是模型—工具循环，不是单次模型请求。

### 自定义工具

按三要素（类型标注、docstring、默认值）写两个本地工具。`calculate` 用 `ast` 解析而不是 `eval()`，限制长度≤200、节点≤64，只放行四则运算，避免执行模型生成的任意代码；`convert_currency` 用固定汇率，`to_currency` 默认 `"CNY"`。

```python
def convert_currency(amount: float, from_currency: str, to_currency: str = "CNY") -> float:
    """按固定汇率换算金额。

    Args:
        amount: 要换算的金额。
        from_currency: 源货币代码 USD / CNY / EUR。
        to_currency: 目标货币代码，默认 CNY。
    """
    rates = {"USD": 7.2, "CNY": 1.0, "EUR": 7.8}
    return amount * rates[from_currency] / rates[to_currency]
```

提问"把 100 美元换算成人民币，再乘以 1.08 的通胀系数"，实跑结果：

```text
100美元按当前汇率换算成人民币是720元，再乘以1.08的通胀系数后，最终结果为777.6元。
消息序列：['human', 'ai', 'tool', 'tool', 'ai']
```

这次连续调了两个工具：先换算、再计算。类型标注决定输入形态，docstring 决定调用时机，默认值减少必填项。

### 研究助手

写 `internet_search` 包一层 Tavily，配研究提示词，显式传入 `TodoListMiddleware()`（v0.7 任务规划默认关闭），创建 Agent 后提问。

```python
agent = create_deep_agent(
    model=model,
    tools=[internet_search],
    system_prompt=RESEARCH_PROMPT,
    middleware=[TodoListMiddleware()],
)
result = agent.invoke({"messages": [{"role": "user", "content": "用中文简要介绍什么是 LangGraph，附来源链接。"}]})
```

实跑结果：

```text
LangGraph 是由 LangChain 团队开发的一个开源框架，用于构建状态化、多代理（Multi-Agent）系统。它通过图结构（Graph）实现复杂的动态工作流，支持循环、分支、状态持久化以及人工干预等核心功能，能够帮助开发者构建具备自主决策能力的 AI 工作流系统。相比传统的线性 LLM 调用链，LangGraph 提供了更灵活、可扩展的架构，适用于对话机器人、审批流程、研究助手等场景。

来源链接：
1. [一文了解LangGraph是什么？——构建智能体的新一代框架](https://www.53ai.com/news/langchain/2025051436094.html)
2. [LangGraph 入门教程 | 菜鸟教程](https://www.runoob.com/ai-agent/langgraph-quick-start.html)
3. [什么是 LangGraph? | IBM](https://www.ibm.com/cn-zh/think/topics/langgraph)

工具调用：['internet_search']
消息序列：['human', 'ai', 'tool', 'ai']
```

消息序列和最小 Agent 一样是 `human → ai → tool → ai`：模型先决定搜索、拿到结果再写报告。抽查来源 3（IBM 那条），HTTP 200 可访问。

系统提示词只写研究目标、证据要求和输出边界——`internet_search` 的参数和接口由工具 Schema 表达，不必在提示词里重复。

## 五、遇到的问题与处理

### 1. 小模型在复杂链路上工具调用不稳

免费 7B（课程示例 `Qwen2.5-7B-Instruct`）单次调用正常，但任务规划、多子 Agent 编排这类长链路可能写坏工具调用格式。准备阶段本机就遇到过：GLM-4-9B 把 `write_todos` 格式写坏、卡在第一步，换成 `Qwen/Qwen3-8B` 后跑通。

### 2. `.env` 改了却没生效

`.env` 不会自动被 Python 进程读到，脚本里要 `load_dotenv()` 显式加载；改完配置后已经起来的进程要重启才会生效。

### 3. LangSmith 区域与 Key 不匹配

`LANGSMITH_ENDPOINT`、API Key、查看 Trace 的控制台必须属于同一区域；切换 Endpoint 不会迁移已有 Trace。只保留一个生效的 Endpoint，URL 末尾不要加 `/`。

## 资料来源

- 课程主页：<https://datawhalechina.github.io/deepagents-in-action/>
- 快速上手 — 5 分钟构建你的第一个 Deep Agent：<https://datawhalechina.github.io/deepagents-in-action/chapters/ch02-quickstart/>
- 从 Agent Framework 到 Agent Harness — Deep Agents 的诞生逻辑：<https://datawhalechina.github.io/deepagents-in-action/chapters/ch01-agent-harness/>
- SiliconFlow 模型广场：<https://cloud.siliconflow.cn/models>
- SiliconFlow API 文档：<https://docs.siliconflow.cn/>
- Tavily：<https://app.tavily.com/>
- LangSmith：<https://smith.langchain.com/>
