# LangGraph 持久化（Persistence）

> **译文说明**：本文档由 Domi 翻译自 https://docs.langchain.com/oss/python/langgraph/persistence（2026-07 版）。本文讲解 LangGraph 的持久化层：**Checkpointers（短期记忆）** 与 **Stores（长期记忆）**。

---

## 一、为什么需要持久化？

持久化（Persistence）让 LangGraph 应用能 **跨多次图执行保留有用信息**，适用场景：

- Agent **继续对话**（多轮上下文）
- **中断后恢复**（HITL、人工审批）
- **失败后恢复**（Fault Tolerance）
- **跨交互记忆**（记住用户偏好、历史事实）

---

## 二、两大互补的持久化系统

LangGraph 提供 **两个互补** 的持久化机制：

| 系统 | 中文 | 持久化什么 | 作用域 | 典型用途 |
|------|------|-----------|--------|----------|
| **Checkpointer** | 检查点器 | **图状态快照**（Graph State Snapshots） | **单线程**（thread-scoped） | 对话连续性、HITL、时光倒流（Time Travel）、故障恢复 |
| **Store** | 存储 | **应用自定义键值数据**（Key-Value Data） | **跨线程**（cross-thread） | 用户偏好、事实、共享知识 |

**最佳实践**：大多数应用 **两者都用**——Checkpointer 跟踪当前线程，Store 跟踪跨线程的持久数据。

---

## 三、快速上手

编译图时传入 checkpointer、store 或两者：

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.store.memory import InMemoryStore

checkpointer = InMemorySaver()
store = InMemoryStore()

graph = builder.compile(checkpointer=checkpointer, store=store)

result = graph.invoke(
    {"messages": [{"role": "user", "content": "Hi, my name is Bob."}]},
    {"configurable": {"thread_id": "thread-1"}},
)
```

> 💡 使用 **Agent Server**（LangSmith 部署）时，**不需要**手动配置 checkpointer 或 store——服务自动处理。

---

## 四、Checkpointer vs Store 详细对比

| 维度 | Checkpointer | Store |
|------|--------------|-------|
| **持久化对象** | 图状态快照 | 应用自定义键值数据 |
| **作用域** | 单个 thread | 跨 thread |
| **记忆类型** | 短期、线程级 | 长期、跨线程 |
| **典型用途** | 对话连续性、HITL、Time Travel、故障恢复 | 用户偏好、事实、共享知识 |
| **访问方式** | 在 config 里传 `thread_id` | 在节点或应用代码中读/写 item |
| **完整文档** | [checkpointers](https://docs.langchain.com/oss/python/langgraph/checkpointers) | [stores](https://docs.langchain.com/oss/python/langgraph/stores) |

---

## 五、Checkpointer 选型指南

LangGraph 提供多种 Checkpointer 实现：

| 实现 | 适用场景 |
|------|----------|
| `InMemorySaver` | **测试 / Demo**（数据在 RAM，进程重启丢失） |
| `SqliteSaver` | **本地开发**（文件存储） |
| `PostgresSaver` / `AsyncPostgresSaver` | **生产环境**（PostgreSQL，支持 async） |

---

## 六、常见问题与修复

### 问题 1：PostgresSaver `thread_id` 太长

**症状**：数据库报错（`thread_id` 列长度限制）

**修复**：保持 `thread_id` < 255 字符。要确定性 ID 时用 UUID 或 hash 截断：

```python
import uuid

config = {"configurable": {"thread_id": str(uuid.uuid4())[:255]}}
```

---

### 问题 2：`MemorySaver` 重启后丢数据

**症状**：进程重启后所有 checkpoint 丢失

**修复**：生产环境用持久化 checkpointer：
- `PostgresSaver`：PostgreSQL（支持 async）
- `SqliteSaver`：本地文件存储（开发用）

---

### 问题 3：Checkpoints 无限增长

**症状**：长对话累积大量 checkpoint，导致延迟和存储成本上升

**修复**：定期清理旧 checkpoint 或设置保留策略：

```python
from langgraph.checkpoint.postgres import PostgresSaver

checkpointer = PostgresSaver.from_conn_string("postgresql://...")
checkpointer.setup()  # 创建带索引的表
# 加 cron job 定期删除 N 天前的 checkpoint
```

---

### 问题 4：父图访问子图 state 不一致

**症状**：子图更新 state 后，父图不能立即看到（因为每个子图管理自己的 checkpoint namespace）

**修复**：
- **方案 A**：使用 [Store 共享 state](https://docs.langchain.com/oss/python/langgraph/stores) 让数据跨图边界
- **方案 B**：配置子图写入父图 checkpoint

---

## 七、关键设计哲学

1. **状态显式化** —— State 是 TypedDict，所有变更可追踪
2. **线程隔离** —— 每个 `thread_id` 独立 checkpoint 链，互不干扰
3. **故障恢复天然支持** —— 节点崩溃后重跑，自动从最近 checkpoint 加载
4. **可观测性内置** —— 每个 checkpoint 都可被 LangSmith trace

---

## 八、最佳实践

| 实践 | 说明 |
|------|------|
| ✅ 生产必用 PostgresSaver | 不要用 InMemorySaver 跑生产 |
| ✅ thread_id 用 UUID | 避免冲突 + 控制长度 |
| ✅ 定期清理 checkpoint | 防止存储爆炸 |
| ✅ 配合 LangSmith trace | 每个 checkpoint 都自动 trace |
| ✅ Store 放跨用户共享数据 | 比如"产品目录"等不随会话变的数据 |

---

> **下一篇**：Interrupts（中断机制）—— Human-in-the-Loop 的核心 API
