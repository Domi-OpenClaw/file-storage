# LangGraph 概览

> **译文说明**：本文档由 Domi（李军召的 AI 助手）从 LangGraph 官方文档英文版（https://docs.langchain.com/oss/python/langgraph/overview）翻译为中文，翻译时间 2026-07-21，原版最近一次更新 2026-07。代码块、API 名称、命令行参数保留英文。专业术语（如 "Checkpointer"、"Pregel"、"Thread"、"HITL"）在首次出现时附英文原文。

---

## 一、LangGraph 是什么？

**LangGraph** 是由 LangChain Inc. 开源的（MIT 协议）**Agent 编排框架与运行时（Orchestration Framework & Runtime）**，用于构建、管理、部署 **长运行、有状态（Long-running, Stateful）** 的 Agent。

**生产客户**：Klarna、Uber、J.P. Morgan、Replit、Elastic 等。

> 一句话定位：**"Build resilient agents"**——构建健壮、可恢复、能长期运行的 AI Agent。

---

## 二、核心定位（与 LangChain 全家桶的关系）

LangGraph 是 LangChain 产品矩阵中的 **"运行时层"**，层级分明：

| 产品 | 角色 | 关键能力 |
|------|------|----------|
| **Deep Agents** | Agent 框架层（高级抽象） | 规划（Planning）、Subagent、文件系统、上下文管理（基于 LangGraph 构建） |
| **LangChain** | Agent 框架层（通用组件） | Model 集成、Tool 抽象、Agent Loop |
| **LangGraph** | **编排运行时层**（核心） | **持久化执行（Durable Execution）、流式输出、人机协同（HITL）、状态管理** |
| **LangSmith** | 可观测性平台 | Trace、Eval、Prompt 管理、部署 |
| **LangSmith Engine** | 智能调优 | 自动检测 Agent trace 中的问题并提议修复，可直接打开 PR |
| **LangSmith Fleet** | 无代码 Agent 构建器 | 模板、集成、流程自动化 |

**关键边界**：LangGraph **不抽象 Prompt、不规定 Agent 架构**，只提供底层编排基础设施。它 **不依赖 LangChain**，但和 LangChain 集成最丝滑。

---

## 三、安装

```bash
# pip
pip install -U langgraph

# uv（推荐，速度更快）
uv add langgraph
```

---

## 四、最小 Hello World 示例

```python
from langgraph.graph import StateGraph, MessagesState, START, END

def mock_llm(state: MessagesState):
    return {"messages": [{"role": "ai", "content": "hello world"}]}

# 1. 定义图结构
graph = StateGraph(MessagesState)
graph.add_node(mock_llm)

# 2. 加边（节点之间的连接）
graph.add_edge(START, "mock_llm")
graph.add_edge("mock_llm", END)

# 3. 编译为可执行对象
graph = graph.compile()

# 4. 调用
graph.invoke({"messages": [{"role": "user", "content": "hi!"}]})
```

**核心抽象**：
- `StateGraph` —— 状态图容器
- `MessagesState` —— 预置的"消息列表"状态 schema
- `START` / `END` —— 图的虚拟起止节点
- `add_node` / `add_edge` —— 节点和边的注册

---

## 五、五大核心能力（Core Benefits）

LangGraph 提供 **任何** 长运行、有状态工作流所需的底层支撑：

### 1. 持久化执行（Persistence / Durable Execution）
- **链接**：[https://docs.langchain.com/oss/python/langgraph/persistence](https://docs.langchain.com/oss/python/langgraph/persistence)
- 构建 **能扛失败、能长期运行** 的 Agent——崩溃后从断点自动恢复，不丢上下文。

### 2. 人机协同（Human-in-the-Loop / HITL）
- **链接**：[https://docs.langchain.com/oss/python/langgraph/interrupts](https://docs.langchain.com/oss/python/langgraph/interrupts)
- 在 **任意执行节点** 暂停，让你审批/修改/补充输入后再继续。

### 3. 全场景记忆（Comprehensive Memory）
- **链接**：[https://docs.langchain.com/oss/python/concepts/memory](https://docs.langchain.com/oss/python/concepts/memory)
- **短期记忆**（当前会话的推理上下文）+ **长期记忆**（跨会话持久化）。

### 4. LangSmith 调试（Debugging with LangSmith）
- **链接**：[https://www.langchain.com/langsmith](https://www.langchain.com/langsmith)
- 可视化 Trace、捕获 State 转换、提供详细运行时指标——专治 Agent 黑盒。

### 5. 生产级部署（Production-Ready Deployment）
- **链接**：[https://docs.langchain.com/langsmith/deployments](https://docs.langchain.com/langsmith/deployments)
- 专为有状态、长运行工作流设计的可扩展部署基础设施。

---

## 六、LangGraph 生态系统

LangGraph 可独立使用，与 LangChain 生态深度集成：

| 工具 | 用途 |
|------|------|
| **LangSmith Observability** | Trace 请求、评估输出、监控部署一站式；本地用 LangGraph 原型，生产用 LangSmith 加可观测 + Eval |
| **LangSmith Deployment** | 一键部署、扩缩容、Studio 可视化原型、团队共享 Agent |
| **LangChain** | Model 集成 + Tool 抽象 + 在 LangGraph 之上构建的 Agent 高级抽象 |

---

## 七、技术灵感与归属

LangGraph 设计灵感来自：
- **Google Pregel**（大规模图计算系统）
- **Apache Beam**（批流统一处理）
- **NetworkX**（图 API 风格参考）

**归属**：LangGraph 由 LangChain Inc.（LangChain 的创造者）开发，但 **无需依赖 LangChain** 即可独立使用。

---

## 八、核心代码库信息

| 项 | 值 |
|----|----|
| **GitHub** | https://github.com/langchain-ai/langgraph |
| **最新版本** | 1.2.9（2026-07-10） |
| **Stars** | 37,750+ |
| **License** | MIT |
| **JS/TS 版** | https://github.com/langchain-ai/langgraphjs |
| **Python 文档** | https://docs.langchain.com/oss/python/langgraph/ |
| **官方论坛** | https://forum.langchain.com |
| **Academy** | https://academy.langchain.com/courses/intro-to-langgraph |

---

## 九、适合谁用？

✅ **适合**：
- 需要 **长运行、有状态** 任务的 Agent 开发
- 需要 **HITL 审批**（金融/医疗/政企场景）
- 需要 **可观测性 + 持久化** 的生产部署
- 多步复杂推理链（研究型 Agent、多 Agent 协作）

❌ **不适合**：
- 简单一次性 LLM 调用（直接用 LangChain 或裸 OpenAI SDK）
- 只想快速搭 Demo 不关心稳定性（用 LangChain agents 即可）

---

> **下一篇**：LangGraph 快速入门（Quickstart）—— 6 步搭建一个 Calculator Agent
