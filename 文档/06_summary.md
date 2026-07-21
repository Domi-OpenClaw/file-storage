# LangGraph 中文文档包（总结篇）

> **文档维护**：Domi（李军召的 AI 助手）
> **翻译日期**：2026-07-21
> **原版来源**：https://docs.langchain.com/oss/python/langgraph/

---

## 一、本文档包结构

本中文文档包覆盖 LangGraph **官方核心章节**，按学习顺序排列：

| # | 章节 | 中文翻译 | 原版链接 |
|---|------|----------|----------|
| 1 | Overview 概览 | 01_overview.md | [link](https://docs.langchain.com/oss/python/langgraph/overview) |
| 2 | Quickstart 快速入门 | 02_quickstart.md | [link](https://docs.langchain.com/oss/python/langgraph/quickstart) |
| 3 | Persistence 持久化 | 03_persistence.md | [link](https://docs.langchain.com/oss/python/langgraph/persistence) |
| 4 | Interrupts 中断机制（HITL） | 04_interrupts.md | [link](https://docs.langchain.com/oss/python/langgraph/interrupts) |
| 5 | Functional API 函数式 API | 05_functional_api.md | [link](https://docs.langchain.com/oss/python/langgraph/functional-api) |

---

## 二、LangGraph 核心心智模型（One-Page Summary）

```
┌────────────────────────────────────────────────────────┐
│              LangGraph = 图 + 状态 + 持久化            │
├────────────────────────────────────────────────────────┤
│                                                        │
│   ┌──────────┐      ┌──────────┐      ┌──────────┐   │
│   │  Node 1  │ ───> │  Node 2  │ ───> │  Node 3  │   │
│   │ (LLM/Tool)│      │(Function)│      │ (HITL?)  │   │
│   └──────────┘      └──────────┘      └──────────┘   │
│        │                  │                 │          │
│        └──────────┬───────┴────────┬────────┘          │
│                   ▼                ▼                   │
│            ┌──────────┐     ┌──────────┐              │
│            │  State   │     │Checkpoint│              │
│            │ (State)  │     │ (持久化) │              │
│            └──────────┘     └──────────┘              │
│                                                        │
└────────────────────────────────────────────────────────┘
```

**5 个心智关键**：

1. **Graph（图）**：节点是函数，边是控制流（顺序 / 条件）
2. **State（状态）**：节点间共享的可变数据（TypedDict + Reducer）
3. **Checkpoint（检查点）**：每步自动持久化（PostgresSaver 生产可用）
4. **Thread（线程）**：用 `thread_id` 标识一次会话 / 任务
5. **Interrupt（中断）**：任意节点暂停，等人类决策后继续

---

## 三、决策树：什么时候用 LangGraph？

```
你的 Agent 需要：
├── 长期运行（分钟/小时/天）？
│   ├── ✅ 是 → 用 LangGraph（持久化 + 断点恢复）
│   └── ❌ 否 → 简单 LLM 调用就够
├── 多步推理 + 工具调用循环？
│   ├── ✅ 是 → 用 Graph API
│   └── ❌ 否 → Functional API（更轻）
├── HITL 审批（关键决策前要人确认）？
│   ├── ✅ 是 → 用 LangGraph Interrupt
│   └── ❌ 否 → 不需要 Interrupt
├── 状态需要跨重启保留？
│   ├── ✅ 是 → 配置 PostgresSaver
│   └── ❌ 否 → InMemorySaver
├── 跨会话记忆（用户偏好、历史事实）？
│   ├── ✅ 是 → 用 Store
│   └── ❌ 否 → 不用 Store
└── 需要可视化调试（团队多人协作）？
    ├── ✅ 是 → Graph API + LangSmith
    └── ❌ 否 → 哪个都行
```

---

## 四、5 个最常用代码片段（速查）

### 1. 最小 Agent

```python
from langgraph.graph import StateGraph, MessagesState, START, END

def my_node(state: MessagesState):
    return {"messages": [{"role": "ai", "content": "hi"}]}

graph = StateGraph(MessagesState).add_node("n", my_node)
graph.add_edge(START, "n").add_edge("n", END)
agent = graph.compile()

agent.invoke({"messages": [{"role": "user", "content": "hello"}]})
```

### 2. 带持久化

```python
from langgraph.checkpoint.postgres import PostgresSaver

checkpointer = PostgresSaver.from_conn_string("postgresql://...")
checkpointer.setup()
graph = builder.compile(checkpointer=checkpointer)

# 用 thread_id 标识会话
graph.invoke({"messages": [...]}, {"configurable": {"thread_id": "user-123"}})
```

### 3. HITL Interrupt

```python
from langgraph.types import interrupt, Command

def approval_node(state):
    decision = interrupt({"question": "Approve?"})
    return {"approved": decision}

# 首次运行
stream = graph.stream_events(input, config, version="v3")
# 中断时 stream.interrupted == True

# 人类决策后恢复
graph.stream_events(Command(resume="yes"), config, version="v3")
```

### 4. 长期记忆（Store）

```python
from langgraph.store.memory import InMemoryStore

store = InMemoryStore()
graph = builder.compile(store=store)

# 在节点中读写 store
def update_user_pref(state, store):
    user_id = state["user_id"]
    store.put(("users",), user_id, {"pref": "dark_mode"})
```

### 5. Functional API

```python
from langgraph.func import entrypoint, task

@task
def step1(x: int) -> int:
    return x * 2

@entrypoint(checkpointer=checkpointer)
def workflow(x: int) -> int:
    a = step1(x).result()
    decision = interrupt("Continue?")
    if not decision:
        return -1
    b = step1(a).result()
    return b
```

---

## 五、与 OpenClaw / Domi 现状的集成思路

**朗新 + Domi 当前现状**：
- ✅ 有 OpenClaw 框架 + Agent Hub 调度
- ✅ 有 subagent + skill 调度机制
- ✅ 有钉钉/飞书通道（支持 HITL 消息触发）
- ❌ 缺：统一的 Agent 持久化层
- ❌ 缺：图编排能力（现在 skill 之间是松散组合）
- ❌ 缺：可视化 trace

**集成方案（3 个阶段）**：

### Phase 1：试点业务（1 周）
挑 **客户拜访 → 立项材料** 这条流程，用 LangGraph 重新编排：
- 把 `bd__customer_visit` skill 改成 LangGraph 节点
- 中间加 HITL Interrupt（小海哥审批关键决策）
- 用 PostgresSaver 持久化（防止中途崩溃）

### Phase 2：知识库 agent 化（1 月）
把 **洛书知识库查询 + 政策分析** 流程 graph 化：
- 节点 1：意图识别（搜索 vs 政策 vs 业务）
- 节点 2：混合检索（语义 + KG）
- 节点 3：LLM 综合 + 引用
- 节点 4：HITL（关键政策让小海哥确认解读）

### Phase 3：多 Agent 协作（2-3 月）
搭 **多 Agent 工作流**：
- 数据采集 agent + 政策分析 agent + 投标 agent
- 用 LangGraph subgraphs 编排
- LangSmith 全面 trace

---

## 六、学习资源

| 资源 | 链接 |
|------|------|
| 官方文档 | https://docs.langchain.com/oss/python/langgraph/ |
| 官方论坛 | https://forum.langchain.com |
| LangChain Academy（免费课） | https://academy.langchain.com/courses/intro-to-langgraph |
| 完整代码示例 | https://github.com/langchain-ai/langgraph/tree/main/examples |
| API 参考 | https://reference.langchain.com/python/langgraph |
| JS/TS 版 | https://github.com/langchain-ai/langgraphjs |

---

## 七、版本与维护

| 项 | 值 |
|----|----|
| 英文原版 | 持续更新（最近 2026-07-10，1.2.9 release） |
| 本中文版 | 2026-07-21 首版 |
| 同步策略 | **每季度** 检查原版更新，重新翻译变更章节 |
| 维护人 | Domi（可自动跟进，可手动补充） |

---

> **下一步建议**：
> 1. 优先读 `02_quickstart.md`（30 分钟上手）
> 2. 看 `03_persistence.md` 理解状态/线程/存储
> 3. 看 `04_interrupts.md` 学 HITL（这个对你的售前审批流最有用）
> 4. 实战：挑一个 skill 改成 LangGraph 实现，跑 1 周看稳定性
