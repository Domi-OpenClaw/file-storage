# LangGraph Functional API（函数式 API）

> **译文说明**：本文档由 Domi 翻译自 https://docs.langchain.com/oss/python/langgraph/functional-api（2026-07 版）。本文讲解 LangGraph 的 **Functional API**——用标准 Python 函数 + 装饰器写 Agent，无需显式定义 Graph 结构。

---

## 一、什么是 Functional API？

**Functional API** 让你用 **最小改动** 把 LangGraph 的核心能力（持久化、记忆、HITL、流式输出）添加到现有代码。

**适用场景**：现有代码已经用标准语言原语（`if`、`for`、函数调用）做控制流，不想强制重构为 DAG。

**两大核心构件**：

| 构件 | 说明 |
|------|------|
| **`@entrypoint`** | 标记函数为工作流入口；封装逻辑、管理执行流（处理长运行任务 + Interrupt） |
| **`@task`** | 代表一个可执行的工作单元（如 API 调用、数据处理）；返回 future-like 对象，可 await 或同步 resolve |

---

## 二、Functional API vs Graph API

| 维度 | Functional API | Graph API |
|------|----------------|-----------|
| **控制流** | 不需要思考图结构；用标准 Python 控制流（通常代码更少） | 显式 Node + Edge |
| **短期记忆** | `@entrypoint` 和 `@task` **不需要显式 state 管理**——状态限定在函数内，不跨函数共享 | 需要声明 `State`、可能需要定义 `Reducer` |
| **Checkpointing** | 任务结果保存到 **现有 checkpoint**（entrypoint 关联） | 每个 **superstep** 后生成新 checkpoint |
| **可视化** | ❌ 不支持（图在运行时动态生成） | ✅ 容易可视化（用 Mermaid PNG） |
| **混用** | ✅ 同一应用可混用 | ✅ 同一应用可混用 |

**结论**：Functional API 适合 **快速集成到现有代码**；Graph API 适合 **需要可视化和精细控制的复杂场景**。

---

## 三、Hello World 示例

写一篇 Essay 并在提交前 Interrupt 请求人类 review：

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.func import entrypoint, task
from langgraph.types import interrupt

@task
def write_essay(topic: str) -> str:
    """写一篇关于某个主题的散文"""
    time.sleep(1)  # 模拟长任务
    return f"An essay about topic: {topic}"

@entrypoint(checkpointer=InMemorySaver())
def workflow(topic: str) -> dict:
    """写 essay + Interrupt 请人类 review"""
    essay = write_essay("cat").result()
    is_approved = interrupt({
        # 任何 JSON 可序列化的 payload
        # 会在 streaming 时作为 Interrupt 暴露给客户端
        "essay": essay,
        # 可以加任何额外信息
        # 比如加 "action" 字段给人类指令
        "action": "Please approve/reject the essay",
    })

    return {
        "essay": essay,
        "is_approved": is_approved,
    }
```

**运行**：

```python
import uuid
from langgraph.types import Command

thread_id = str(uuid.uuid4())
config = {"configurable": {"thread_id": thread_id}}

# 首次运行——撞到 interrupt 后暂停
stream = workflow.stream_events("cat", config, version="v3")
_ = stream.output

print({"write_essay": stream.interrupts[0].value["essay"]})
print({"__interrupt__": stream.interrupts})
# {'write_essay': 'An essay about topic: cat'}
# {
#   '__interrupt__': [
#     Interrupt(
#       value={
#           'essay': 'An essay about topic: cat',
#           'action': 'Please approve/reject the essay'
#       },
#       id='369d44b3d93d4a631ae583367ac6b5cc'
#     )
#   ]
# }

# 人类 review 后恢复
human_review = True
resumed_stream = workflow.stream_events(Command(resume=human_review), config, version="v3")
print(resumed_stream.output)
# {'essay': 'An essay about topic: cat', 'is_approved': True}
```

**关键点**：
- `write_essay` 是 `@task`——执行时结果保存到 checkpoint
- `interrupt({...})` 暂停工作流，把 payload 暴露给调用方
- **恢复时**，工作流从 **头重新执行**，但 `write_essay` 的结果从 checkpoint 加载（不重算）

---

## 四、`@entrypoint` 详解

### 定义规则

用 `@entrypoint` 装饰函数定义工作流入口：

1. **函数必须接受 1 个位置参数**（作为工作流输入）
2. **多个数据用 dict 包装**为第一个参数
3. 装饰函数返回 **`Pregel` 实例**（LangGraph 底层运行时对象）
4. 通常传 **`checkpointer`** 参数启用持久化和 HITL

### 同步版

```python
from langgraph.func import entrypoint

@entrypoint(checkpointer=checkpointer)
def my_workflow(some_input: dict) -> int:
    # 一些可能涉及长运行任务（如 API 调用）的逻辑
    # 可能被 HITL interrupt
    ...
    return result
```

### 异步版

```python
from langgraph.func import entrypoint

@entrypoint(checkpointer=checkpointer)
async def my_workflow(some_input: dict) -> int:
    # 异步逻辑
    ...
    return result
```

---

## 五、`@task` 详解

`@task` 代表一个 **可独立执行的离散工作单元**（如 API 调用、数据库查询、LLM 调用）：

- 返回 **future-like 对象**（类似 asyncio.Future）
- 可以 `.result()`（同步）或 `await`（异步）
- 任务结果 **自动保存到 checkpoint**——重跑时不重复执行

### 同步 Task

```python
from langgraph.func import task

@task
def fetch_user(user_id: int) -> dict:
    response = requests.get(f"https://api.example.com/users/{user_id}")
    return response.json()

@entrypoint(checkpointer=checkpointer)
def workflow(user_id: int) -> dict:
    user = fetch_user(user_id).result()
    return {"user": user}
```

### 异步 Task

```python
@task
async def fetch_user(user_id: int) -> dict:
    async with aiohttp.ClientSession() as session:
        async with session.get(f"https://api.example.com/users/{user_id}") as resp:
            return await resp.json()

@entrypoint(checkpointer=checkpointer)
async def workflow(user_id: int) -> dict:
    user = await fetch_user(user_id)
    return {"user": user}
```

---

## 六、典型场景对比

### 场景 A：长跑研究任务

```python
@task
def search_papers(query: str) -> list:
    # 学术搜索 API
    return [...]

@task
def summarize_paper(paper: dict) -> str:
    # LLM 总结
    return summary

@entrypoint(checkpointer=PostgresSaver(...))
def research_workflow(query: str) -> str:
    papers = search_papers(query).result()
    summaries = [summarize_paper(p).result() for p in papers]

    # 让人类决定要不要继续深挖某篇
    chosen = interrupt({
        "papers": summaries,
        "prompt": "Select a paper to deep-dive:"
    })

    return chosen
```

**优势**：每步自动 checkpoint，任何一步失败都能恢复。

### 场景 B：审批工作流

```python
@task
def execute_transfer(amount: float, to_account: str) -> str:
    # 银行 API 调用
    return transaction_id

@entrypoint(checkpointer=PostgresSaver(...))
def transfer_workflow(amount: float, to_account: str) -> str:
    # 大额转账需要审批
    if amount > 10000:
        approved = interrupt({
            "amount": amount,
            "to": to_account,
            "prompt": "Approve large transfer?"
        })
        if not approved:
            return "rejected"

    tx_id = execute_transfer(amount, to_account).result()
    return f"Transfer {amount} to {to_account}: {tx_id}"
```

---

## 七、与 Graph API 混用

两个 API 共享底层运行时，可在 **同一应用** 混用：

```python
# 用 Graph API 定义主流程
graph = StateGraph(MainState)
graph.add_node("research", research_node)  # research_node 是 Graph API
graph.add_node("summarize", summarize_subgraph)  # summarize_subgraph 是 Functional API
graph.add_edge(START, "research")
graph.add_edge("research", "summarize")
graph.add_edge("summarize", END)
```

---

## 八、生产建议

| 实践 | 说明 |
|------|------|
| ✅ **生产必用持久化 checkpointer** | PostgresSaver / SqliteSaver |
| ✅ **任务粒度合理** | 太粗：丢失检查点收益；太细：checkpoint 开销大 |
| ✅ **Interrupt payload 要明确** | 让前端/客户端能解析 |
| ✅ **错误处理在 task 内** | Task 抛异常会被 entrypoint 捕获并传播 |
| ✅ **可观测性** | 配合 LangSmith trace，每个 task 自动 trace |

---

## 九、何时选哪个 API？

| 场景 | 推荐 |
|------|------|
| 现有 Python 代码集成 LangGraph | **Functional API**（改动最小） |
| 复杂多 Agent 协作 | **Graph API**（可视化 + 精细控制） |
| 简单一次性脚本 | **Functional API**（更快上手） |
| 需要团队理解/调试 | **Graph API**（可视化是杀手锏） |
| 频繁重启 / 长期运行任务 | **两者都 OK**（都支持 checkpointing） |

---

## 十、参考资源

- **官方教程**：[Use Functional API](https://docs.langchain.com/oss/python/langgraph/use-functional-api)
- **API 参考**：[reference.langchain.com/python/langgraph](https://reference.langchain.com/python/langgraph)
- **完整示例库**：[github.com/langchain-ai/langgraph/tree/main/examples](https://github.com/langchain-ai/langgraph/tree/main/examples)

---

> **下一篇（建议）**：LangGraph 与你的业务集成方案
