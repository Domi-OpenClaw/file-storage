# LangGraph 快速入门（Quickstart）

> **译文说明**：本文档由 Domi 翻译自 https://docs.langchain.com/oss/python/langgraph/quickstart（2026-07 版）。本教程演示如何用 **Graph API** 或 **Functional API** 搭建一个 Calculator Agent（计算器 Agent）。

---

## 一、选哪种 API？

| API | 适合 |
|-----|------|
| **Graph API** | 喜欢把 Agent 定义为 **节点（Node）+ 边（Edge）** 的图结构 |
| **Functional API** | 喜欢把 Agent 定义为 **单个函数**，使用标准 Python 控制流（`if`、`for`、函数调用） |

> 本篇重点讲 **Graph API**（更主流）。两 API 共享同一个底层运行时，可混用。

---

## 二、环境准备

本示例使用 **Claude Sonnet 4.6**（Anthropic），需要：
1. 注册 [Anthropic](https://www.anthropic.com/) 账号并获取 API Key
2. 设置环境变量：
   ```bash
   export ANTHROPIC_API_KEY=sk-ant-***
   ```

---

## 三、6 步搭建 Calculator Agent

### Step 1：定义 Tools 和 Model

```python
from langchain.tools import tool
from langchain.chat_models import init_chat_model

# 初始化模型
model = init_chat_model(
    "claude-sonnet-4-6",
    temperature=0
)

# 定义三个工具：加、乘、除
@tool
def multiply(a: int, b: int) -> int:
    """Multiply `a` and `b`.

    Args:
        a: First int
        b: Second int
    """
    return a * b


@tool
def add(a: int, b: int) -> int:
    """Adds `a` and `b`.

    Args:
        a: First int
        b: Second int
    """
    return a + b


@tool
def divide(a: int, b: int) -> float:
    """Divide `a` and `b`.

    Args:
        a: First int
        b: Second int
    """
    return a / b


# 把 LLM 与工具绑定
tools = [add, multiply, divide]
tools_by_name = {tool.name: tool for tool in tools}
model_with_tools = model.bind_tools(tools)
```

---

### Step 2：定义 State（图状态）

LangGraph 的 State 用于在节点间传递数据。

```python
from langchain.messages import AnyMessage
from typing_extensions import TypedDict, Annotated
import operator


class MessagesState(TypedDict):
    # Annotated + operator.add 表示：每次节点返回的 messages 会 **追加** 到现有列表，而非覆盖
    messages: Annotated[list[AnyMessage], operator.add]
    llm_calls: int
```

**关键点**：
- `TypedDict` 是 Python 标准类型，用于定义 schema
- `Annotated[list, operator.add]` 是 **Reducer（聚合器）**——指定如何合并新值
- State 在整个 Agent 执行期间 **持续存在**（持久化）

---

### Step 3：定义 Model Node（决定要不要调用工具）

```python
from langchain.messages import SystemMessage


def llm_call(state: dict):
    """LLM 决定是否调用工具"""
    return {
        "messages": [
            model_with_tools.invoke(
                [
                    SystemMessage(
                        content="你是一个有帮助的助手，专门对一组输入执行算术运算。"
                    )
                ]
                + state["messages"]
            )
        ],
        "llm_calls": state.get('llm_calls', 0) + 1
    }
```

---

### Step 4：定义 Tool Node（执行工具调用）

```python
from langchain.messages import ToolMessage


def tool_node(state: dict):
    """执行工具调用"""
    result = []
    for tool_call in state["messages"][-1].tool_calls:
        tool = tools_by_name[tool_call["name"]]
        observation = tool.invoke(tool_call["args"])
        result.append(ToolMessage(content=observation, tool_call_id=tool_call["id"]))
    return {"messages": result}
```

---

### Step 5：定义路由逻辑（条件边）

```python
from typing import Literal
from langgraph.graph import StateGraph, START, END


def should_continue(state: MessagesState) -> Literal["tool_node", END]:
    """判断继续循环还是停止（基于 LLM 是否发起了 tool_call）"""

    messages = state["messages"]
    last_message = messages[-1]

    # 如果 LLM 发起 tool_call，去执行工具
    if last_message.tool_calls:
        return "tool_node"

    # 否则停止（回复用户）
    return END
```

---

### Step 6：Build + Compile（构建与编译）

```python
# 1. 创建图构建器
agent_builder = StateGraph(MessagesState)

# 2. 添加节点
agent_builder.add_node("llm_call", llm_call)
agent_builder.add_node("tool_node", tool_node)

# 3. 添加边（连接节点）
agent_builder.add_edge(START, "llm_call")
agent_builder.add_conditional_edges(
    "llm_call",
    should_continue,
    ["tool_node", END]
)
agent_builder.add_edge("tool_node", "llm_call")  # 工具执行完回到 LLM

# 4. 编译为可执行 Agent
agent = agent_builder.compile()

# 5. 可视化（可选，需要 IPython）
from IPython.display import Image, display
display(Image(agent.get_graph(xray=True).draw_mermaid_png()))

# 6. 调用 Agent
from langchain.messages import HumanMessage
messages = [HumanMessage(content="Add 3 and 4.")]
messages = agent.invoke({"messages": messages})
for m in messages["messages"]:
    m.pretty_print()
```

---

## 四、完整代码（复制即可运行）

```python
# Step 1: Define tools and model
from langchain.tools import tool
from langchain.chat_models import init_chat_model

model = init_chat_model("claude-sonnet-4-6", temperature=0)

@tool
def multiply(a: int, b: int) -> int:
    """Multiply `a` and `b`."""
    return a * b

@tool
def add(a: int, b: int) -> int:
    """Adds `a` and `b`."""
    return a + b

@tool
def divide(a: int, b: int) -> float:
    """Divide `a` and `b`."""
    return a / b

tools = [add, multiply, divide]
tools_by_name = {tool.name: tool for tool in tools}
model_with_tools = model.bind_tools(tools)

# Step 2: Define state
from langchain.messages import AnyMessage
from typing_extensions import TypedDict, Annotated
import operator

class MessagesState(TypedDict):
    messages: Annotated[list[AnyMessage], operator.add]
    llm_calls: int

# Step 3: Define model node
from langchain.messages import SystemMessage

def llm_call(state: MessagesState):
    """LLM decides whether to call a tool or not"""
    return {
        "messages": [
            model_with_tools.invoke(
                [SystemMessage(content="你是一个有帮助的助手，专门对一组输入执行算术运算。")]
                + state["messages"]
            )
        ],
        "llm_calls": state.get('llm_calls', 0) + 1
    }

# Step 4: Define tool node
from langchain.messages import ToolMessage

def tool_node(state: dict):
    """Performs the tool call"""
    result = []
    for tool_call in state["messages"][-1].tool_calls:
        tool = tools_by_name[tool_call["name"]]
        observation = tool.invoke(tool_call["args"])
        result.append(ToolMessage(content=observation, tool_call_id=tool_call["id"]))
    return {"messages": result}

# Step 5: Define logic to determine whether to end
from typing import Literal
from langgraph.graph import StateGraph, START, END

def should_continue(state: MessagesState) -> Literal["tool_node", END]:
    messages = state["messages"]
    last_message = messages[-1]
    if last_message.tool_calls:
        return "tool_node"
    return END

# Step 6: Build agent
agent_builder = StateGraph(MessagesState)
agent_builder.add_node("llm_call", llm_call)
agent_builder.add_node("tool_node", tool_node)
agent_builder.add_edge(START, "llm_call")
agent_builder.add_conditional_edges("llm_call", should_continue, ["tool_node", END])
agent_builder.add_edge("tool_node", "llm_call")
agent = agent_builder.compile()

# Invoke
from langchain.messages import HumanMessage
messages = [HumanMessage(content="Add 3 and 4.")]
messages = agent.invoke({"messages": messages})
for m in messages["messages"]:
    m.pretty_print()
```

---

## 五、图结构（可视化）

```
START
  ↓
[llm_call] ──should_continue──> [tool_node]
  ↑                                │
  └────────────────────────────────┘
  │
  └─ 无 tool_call → END
```

---

## 六、生产建议

1. **配置 LangSmith 做 trace**：设 `LANGSMITH_TRACING=true` + `LANGSMITH_API_KEY`，所有调用会自动 trace
2. **配置持久化 Checkpointer**（见 Persistence 章节）——否则重启丢状态
3. **考虑 LangSmith Engine**——自动检测问题并提议 PR

---

> **下一篇**：Persistence（持久化）—— Checkpointer 与 Store 的设计哲学
