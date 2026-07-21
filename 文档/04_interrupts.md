# LangGraph 中断机制（Interrupts）—— Human-in-the-Loop

> **译文说明**：本文档由 Domi 翻译自 https://docs.langchain.com/oss/python/langgraph/interrupts（2026-07 版）。本文讲解 LangGraph 的 **Interrupts** 机制——在 **任意节点** 暂停执行、等待外部输入、然后继续。

---

## 一、什么是 Interrupt？

**Interrupt（中断）** 允许你在 **任意图节点** 暂停执行，等待外部输入后再继续。这是 **Human-in-the-Loop（HITL，人机协同）** 的核心机制。

**触发 Interrupt 时 LangGraph 的行为**：
1. 在 `interrupt()` 调用点 **暂停执行**
2. 用 **Checkpointer 保存当前图状态**
3. 把 **payload 返回给调用方**（在 `stream.interrupts` 或 `result["__interrupt__"]`）
4. **无限等待**直到你用 `Command(resume=...)` 恢复
5. 你的响应 **作为 `interrupt()` 的返回值传回节点**

---

## 二、与"静态断点（Static Breakpoints）"的区别

| 维度 | Interrupt | Static Breakpoint |
|------|-----------|-------------------|
| 位置 | **任意代码位置**，动态触发 | 固定节点前/后 |
| 条件 | 可基于运行时条件动态判断 | 无 |
| 灵活性 | ⭐⭐⭐ 高 | ⭐ 低 |

**Interrupt 是动态的**——可以放在任何地方，可以基于应用逻辑条件触发。

---

## 三、使用 Interrupt 的 3 个前置条件

要在节点中使用 `interrupt()`：

1. **Checkpointer**（持久化图状态；生产用持久化 checkpointer）
2. **thread_id**（在 config 里指定，让 runtime 知道恢复哪个 state）
3. **调用 `interrupt()`**（payload 必须是 JSON 可序列化）

```python
from langgraph.types import interrupt

def approval_node(state: State):
    # 暂停并请求审批
    approved = interrupt("Do you approve this action?")

    # 当你恢复时，Command(resume=...) 的值会作为返回值传回这里
    return {"approved": approved}
```

---

## 四、完整流程详解

调用 `interrupt()` 时会发生什么：

1. **图执行被挂起** —— 在 `interrupt()` 调用点精确暂停
2. **State 被保存** —— Checkpointer 写入当前图状态（生产环境用数据库）
3. **值返回调用方**：
   - 用 [Event Streaming](https://docs.langchain.com/oss/python/langgraph/event-streaming)（`graph.stream_events(..., version="v3")`）时，出现在 `stream.interrupts`
   - 用默认 `invoke()` API 时，出现在 `__interrupt__`
   - 可以是任何 JSON 可序列化的值（字符串、对象、数组等）
4. **图无限等待** —— 直到你用响应恢复
5. **响应传回节点** —— 作为 `interrupt()` 的返回值

---

## 五、恢复 Interrupt

Interrupt 暂停后，用 `Command(resume=...)` 重新 invoke 图：

```python
from langgraph.types import Command

# 首次运行——撞到 interrupt 后暂停
config = {"configurable": {"thread_id": "thread-1"}}
stream = graph.stream_events({"input": "data"}, config=config, version="v3")
final = stream.output  # 等待最终 state

# 检测是否中断
if stream.interrupted:
    print(stream.interrupts)
    # > (Interrupt(value='Do you approve this action?'),)

# 用人类响应恢复
resumed = graph.stream_events(Command(resume=True), config=config, version="v3")
final = resumed.output
```

> **注意**：默认 `graph.invoke(...)` API 也能用，Interrupt 出现在 `result["__interrupt__"]`。不需要 streaming 时用它。

---

## 六、恢复的关键点

| 要点 | 说明 |
|------|------|
| ✅ **必须用同一个 thread_id** | 跨调用保持 thread_id 一致才能恢复 |
| ✅ `Command(resume=...)` 的值就是 `interrupt()` 的返回值 | payload 可以是任何 JSON 值 |
| ✅ 节点会从 **开头** 重跑 | `interrupt()` 之前的代码会重新执行 |
| ✅ 任何 JSON 值 | 不限于 bool，可以传 dict、list 等 |

> ⚠️ **`Command(resume=...)` 是 invoke/stream/stream_events 的 **唯一合法输入** Command 模式**。其他 Command 参数（`update`、`goto`、`graph`）是 [节点函数返回值](https://docs.langchain.com/oss/python/langgraph/graph-api#command) 用的，不要用作多轮对话的输入。

---

## 七、典型应用场景

| 场景 | 说明 |
|------|------|
| ✅ **审批工作流** | 执行关键动作（API 调用、数据库变更、金融交易）前暂停 |
| ✅ **处理多个 Interrupt** | 单次 invoke 中多个 interrupt 时，用 Interrupt ID 配对 resume 值 |
| ✅ **Review & Edit** | 让人类审查并修改 LLM 输出或 tool_call 后再继续 |
| ✅ **中断 Tool Call** | 执行 tool_call 前暂停，让人类审查/修改 |
| ✅ **校验人类输入** | 进入下一步前校验人类输入是否合法 |

---

## 八、HITL 完整循环（推荐模式）

用 **Event Streaming** + **循环** 消费消息 chunks 和 state snapshots：

```python
from langgraph.types import Command

stream_input: dict | Command = initial_input

while True:
    stream = graph.stream_events(stream_input, config=config, version="v3")

    # 流式输出 LLM token（包含 subgraphs）
    for message in stream.messages:
        for token in message.text:
            display_streaming_content(token)

    # 检查是否中断
    if not stream.interrupted:
        final_state = stream.output
        break

    # 处理 interrupt
    interrupt_info = stream.interrupts[0].value
    user_response = get_user_input(interrupt_info)
    stream_input = Command(resume=user_response)
```

**关键字段**：
- `stream.messages` —— Chat-model 输出（content blocks），遍历 `message.text` 拿 token delta
- `stream.values` —— 每步 state snapshot
- `stream.interrupted` —— bool，是否暂停等输入
- `stream.interrupts` —— payload 列表
- `stream.subgraphs[*].messages` —— 嵌套 subgraphs 的消息 chunks

---

## 九、典型代码模式

### 模式 1：审批（Approve/Reject）

```python
def approval_node(state: State):
    decision = interrupt({
        "question": "Approve this transaction?",
        "amount": state["transaction"]["amount"],
        "recipient": state["transaction"]["recipient"]
    })
    return {"approved": decision == "approve"}
```

### 模式 2：Tool Call 前审查

```python
def tool_review_node(state: State):
    last_msg = state["messages"][-1]
    if last_msg.tool_calls:
        # 暂停让人类审查 tool_call
        edited_call = interrupt({
            "tool_call": last_msg.tool_calls[0],
            "instruction": "Review/edit this tool call"
        })
        # 用编辑后的 tool_call 继续
        last_msg.tool_calls[0] = edited_call
    return state
```

### 模式 3：校验人类输入

```python
def collect_user_info_node(state: State):
    user_input = interrupt({
        "prompt": "请输入你的邮箱",
        "validation": "must be valid email"
    })
    # 人类响应后继续
    return {"email": user_input["email"]}
```

---

## 十、生产建议

| 实践 | 说明 |
|------|------|
| ✅ **持久化 checkpointer** | 不要用 InMemorySaver 跑生产 |
| ✅ **明确的 Interrupt payload schema** | 让前端能解析 |
| ✅ **超时机制** | 防止 Interrupt 永久挂起（业务层加超时） |
| ✅ **审计日志** | 每次 Interrupt + 人类响应都记日志（合规） |
| ✅ **UI 集成** | 把 `stream.interrupts` 接到运营后台 / 钉钉 / 飞书 |

---

> **下一篇**：Functional API —— 用函数式风格写 Agent
