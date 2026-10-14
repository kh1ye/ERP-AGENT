# ERP-Agent 项目完整架构与技术解析

**项目名称：** ERP-Agent｜基于 DeepAgents、LangGraph 与 MCP 的企业智能采购系统

**项目类型：** AI Agent / Multi-Agent / 企业级智能采购系统

**技术方向：** Agent 后端开发、Harness Engineering、MCP 工具集成、长期记忆与工程可靠性

**个人项目仓库：** https://github.com/kh1ye/ERP-AGENT

**上游来源：** https://github.com/wodrake/ERP-AGENT-open-source

---

## 一、项目背景与项目介绍

### 1.1 项目背景

传统 ERP 系统通常需要采购人员在不同业务模块之间切换，手工完成供应商查询、零部件查询、库存分析、报价比较以及采购订单创建等操作。

当采购任务涉及多个业务步骤时，还需要人工进行数据汇总和业务决策。

ERP-Agent 针对这一问题，利用大语言模型的自然语言理解、任务规划和工具调用能力，将复杂 ERP 业务操作组织成可执行的多智能体任务流程。

### 1.2 项目定位

ERP-Agent 不是普通的 AI 聊天机器人，而是一个面向企业采购业务的 Multi-Agent 智能系统。

它采用 **DeepAgents + LangGraph + MCP + FastAPI** 的技术架构，实现：

1. 自然语言驱动 ERP 操作。
2. 主 Agent 与专业子 Agent 协作。
3. 通过 MCP 动态调用企业业务工具。
4. 跨会话状态持久化与长期记忆。
5. 高风险业务操作的人工审批。
6. 动态执行审查和异常处理。
7. Docker 沙箱隔离代码执行。
8. Skills 技能加载和任务执行指导。
9. 前端实时展示执行过程和结果。

### 1.3 典型业务场景

用户输入：

> 查询某种零部件的库存，如果库存不足 100 件，请比较可用供应商的报价，推荐采购方案，并准备创建采购订单。

系统需要完成：

1. 理解用户采购需求。
2. 判断需要调用哪些 ERP 工具。
3. 查询零部件与库存。
4. 查询和比较供应商。
5. 生成采购建议。
6. 准备采购订单。
7. 请求用户审批。
8. 审批通过后执行相应工具调用。
9. 将结果通过 SSE 返回前端。
10. 保存相关执行状态与记忆。

**项目核心目标：** 将传统 ERP 中分散的业务操作，组织成由 Agent 规划、工具执行、系统审查和人工审批共同控制的智能业务流程。

---

## 二、项目整体系统架构

### 2.1 系统分层

整个系统可分为六个逻辑层次：

| 层级 | 技术 | 核心职责 |
|---|---|---|
| 前端展示层 | Next.js、React、TypeScript | 聊天窗口、历史消息、审批和流式结果 |
| Web API 层 | FastAPI、SSE | 请求接收、会话管理、Agent 调用 |
| Agent 编排层 | DeepAgents、LangGraph、DeepSeek | 任务规划、子 Agent 委派、状态管理 |
| Harness 运行时层 | Middleware、Review、Skills、HITL | 执行治理、审查、审批、异常处理 |
| MCP 工具层 | FastMCP、langchain-mcp-adapters | 动态发现和调用 ERP 工具 |
| 基础设施层 | MongoDB、Docker、外部 Java ERP | 持久化、沙箱和业务数据 |

### 2.2 总体调用关系

```text
                     用户
                      │
                      ▼
           Next.js + React 前端
                      │
                  HTTP / SSE
                      │
                      ▼
                 FastAPI
               localhost:8000
                      │
                      ▼
                Agent Loader
                      │
                      ▼
           DeepAgents / LangGraph
                      │
                Main Agent
                      │
             ┌────────┴────────┐
             ▼                 ▼
       采购分析 Agent      订单管理 Agent
             │                 │
             └────────┬────────┘
                      │
       ┌──────────────┼──────────────┐
       ▼              ▼              ▼
  MCP Tools       Memory Store   Harness Runtime
       │              │              │
       ▼              ▼              ▼
  MCP Server       MongoDB       Review / HITL
 localhost:9000    :27017        Skills / Sandbox
       │
       ▼
  Java ERP REST API
 localhost:8081
       │
       ▼
  供应商 / 零部件 / 订单 / 库存
```

注意：

- FastAPI 提供 Agent 应用 API。
- MCP Server 负责封装业务工具。
- Java ERP 服务负责实际业务操作。
- MongoDB 负责执行状态和长期记忆。
- Docker Sandbox 负责隔离代码及文件执行。
- Harness 运行时贯穿 Agent 执行过程，不是一个单独的线性调用阶段。

---

## 三、项目技术栈

| 类型 | 技术 | 具体用途 |
|---|---|---|
| 编程语言 | Python | Agent 和后端开发 |
| 后端框架 | FastAPI | REST API、SSE 流式响应 |
| Agent 框架 | DeepAgents | 主 Agent 与 SubAgent 委派 |
| 状态图框架 | LangGraph | 状态管理、中断与恢复 |
| LLM | DeepSeek | 规划、推理、工具选择 |
| MCP 框架 | FastMCP | 实现 MCP Server |
| MCP 客户端 | langchain-mcp-adapters | 发现、加载并调用 MCP Tool |
| 数据库 | MongoDB | 会话、记忆、Checkpoint |
| 数据库驱动 | Motor、PyMongo | MongoDB 读写 |
| 容器技术 | Docker SDK | 沙箱生命周期管理 |
| 前端 | Next.js 16、React 19 | 用户交互 |
| 前端语言 | TypeScript | 类型安全开发 |
| 数据处理 | Pandas | 业务数据处理 |
| 图表 | Matplotlib | 图表生成 |
| 配置 | YAML、.env | 子 Agent 与服务配置 |

---

## 四、FastAPI 后端架构

### 4.1 核心职责

FastAPI 是整个 Agent 应用的 Web 服务入口。

主要负责：

- 接收用户聊天请求。
- 验证和管理请求上下文。
- 根据用户及会话获取 Agent。
- 调用 LangGraph 的执行接口。
- 转换 Agent 执行事件。
- 通过 SSE 推送实时消息。
- 处理 Agent Interrupt 和 Resume。
- 查询会话历史及执行状态。
- 提供健康检查和文件下载接口。

### 4.2 主要代码位置

```text
src/api_view/
├── web_main.py
├── web_config.py
├── agent_loader.py
├── mongodb_store.py
└── api/
    ├── chat.py
    └── history.py
```

| 文件 | 职责 |
|---|---|
| `web_main.py` | 创建 FastAPI 应用、注册路由和生命周期管理 |
| `web_config.py` | Web 服务和数据库配置 |
| `agent_loader.py` | Agent 实例及用户会话管理 |
| `mongodb_store.py` | MongoDB Store 的具体实现 |
| `api/chat.py` | 对话、流式输出、中断与恢复 |
| `api/history.py` | 会话历史查询及管理 |

### 4.3 核心 API

| 方法 | 接口 | 功能 |
|---|---|---|
| POST | `/api/chat/stream` | 发起流式 Agent 对话 |
| POST | `/api/chat/{thread_id}/resume` | 恢复被中断的 Agent |
| GET | `/api/chat/{thread_id}/state` | 查询 Agent 状态 |
| GET | `/api/chat/{thread_id}/history` | 查询会话消息 |
| GET | `/api/history` | 查询用户会话列表 |
| DELETE | `/api/history/{thread_id}` | 删除会话 |
| GET | `/api/download/{filename}` | 下载生成文件 |
| GET | `/health` | 服务健康检查 |

### 4.4 SSE 流式输出原理

系统使用 LangGraph 的异步流式执行接口获取事件。

核心逻辑：

```python
async for namespace, chunk_type, chunk in session.agent.astream(
    ...
):
    # 根据事件类型进行转换
    # 将事件通过 SSE 发送给前端
    ...
```

基本流程：

```text
用户发送请求
      ↓
FastAPI 接收
      ↓
Agent 执行
      ↓
LangGraph 持续产生事件
      ↓
后端转换为 SSE
      ↓
前端实时展示
```

**重点：** SSE 主要用于服务端向浏览器持续推送执行结果，而不是让浏览器直接控制模型内部执行。

---

## 五、Agent 核心编排架构

### 5.1 核心文件

`src/agent/main_agent.py`

关键函数：

`create_main_agent()`

该文件相当于整个 Agent 系统的组装中心。

### 5.2 Agent 创建流程

```text
create_main_agent()
       │
       ├── 初始化 DeepSeek 模型
       │
       ├── 初始化 Sandbox
       │
       ├── 构建 CompositeBackend
       │
       ├── 加载 MCP 工具
       │
       ├── 加载自定义工具
       │
       ├── 读取 SubAgent YAML
       │
       ├── 初始化 Middleware
       │
       ├── 初始化 Checkpointer / Store
       │
       └── create_deep_agent()
                    │
                    ▼
               Agent 实例
```

### 5.3 关键代码

```python
agent = create_deep_agent(
    model=llm,
    tools=all_tools,
    system_prompt=system_prompt,
    middleware=middlewares,
    subagents=subagents if subagents else None,
    skills=None,
    memory=[str(MEMORY_DIR / "AGENTS.md")],
    backend=composite_backend,
    interrupt_on=INTERRUPT_ON_TOOLS,
    checkpointer=checkpointer,
    store=store,
    name="procurement-main-agent",
)
```

### 5.4 参数解释

| 参数 | 作用 |
|---|---|
| `model` | 指定 LLM |
| `tools` | 传入 Agent 可调用的工具 |
| `system_prompt` | 定义 Agent 角色和行为规范 |
| `middleware` | 插入执行控制和处理逻辑 |
| `subagents` | 配置专业子 Agent |
| `skills` | 框架内置 Skills 配置 |
| `memory` | 加载 `AGENTS.md` 等持久指令 |
| `backend` | 文件操作与执行后端 |
| `interrupt_on` | 需要人工确认的工具 |
| `checkpointer` | 执行状态持久化 |
| `store` | 跨会话数据持久化 |
| `name` | Agent 名称 |

需要区分：

`memory=AGENTS.md` 主要用于加载 Agent 指令，不等于完整的长期用户记忆系统。

长期记忆由 MongoDB Store 和项目自定义 Memory 模块管理。

---

## 六、Multi-Agent 多智能体架构

### 6.1 为什么使用 Multi-Agent？

单 Agent 虽然也能够使用多个工具，但复杂业务可能包含大量中间数据、工具日志和不同职责。

若所有任务都由一个 Agent 处理，容易出现：

- 上下文不断膨胀。
- 业务职责混杂。
- 工具选择范围过大。
- 中间结果干扰最终决策。
- 维护和测试困难。

因此项目采用主 Agent 与专业子 Agent 协作的模式。

### 6.2 Main Agent

主 Agent 负责：

1. 理解用户的自然语言需求。
2. 制定整体任务计划。
3. 判断使用普通工具还是委派子 Agent。
4. 管理子任务返回结果。
5. 协调审批及执行结果。
6. 汇总最终回复。

### 6.3 Procurement Analyst

采购分析子 Agent 主要负责：

- 查询零部件与供应商。
- 分析库存情况。
- 比较供应商价格。
- 形成采购建议。
- 生成分析报告与图表。

### 6.4 Procurement Order

订单管理子 Agent 主要负责：

- 查询采购订单。
- 创建采购订单。
- 修改订单。
- 跟踪订单状态。
- 根据高风险操作要求进行审批。

### 6.5 YAML 声明式配置

专业 Agent 使用 YAML 定义其职责和可访问工具。

示意：

```yaml
name: procurement-order

description: >
  采购订单专家，负责采购订单的创建、
  修改、状态跟踪和人工审批。

tools:
  - order_create
  - order_update
  - order_get
  - order_page
  - request_order_info
```

随后通过代码加载配置：

```python
load_subagent_configs()
resolve_subagent_tools(...)
```

### 6.6 SubAgent 与 LangGraph 的关系

LangGraph 提供状态图、状态传递、中断恢复等底层能力。

DeepAgents 在其基础上封装了主 Agent 和子 Agent 的高层委派机制。

这里的 `subagents` 参数以及 `task` 工具属于 DeepAgents 的高层接口，而不是 LangGraph 核心库直接提供的同名 API。

采用子 Agent 的主要价值：

- 明确职责分工。
- 限制专业 Agent 的工具集合。
- 隔离中间任务上下文。
- 减少主 Agent 的无关信息。
- 方便独立维护和扩展。

---

## 七、MCP 工具架构

### 7.1 MCP 的作用

MCP（Model Context Protocol）是一套让 AI 应用连接外部工具与资源的标准协议。

ERP-Agent 利用 MCP 将订单、零部件、供应商、库存等业务能力封装为 Agent 能够调用的工具。

### 7.2 系统中的 MCP 调用链

```text
Main Agent / SubAgent
          │
          ▼
LangChain Tool
          │
          ▼
MultiServerMCPClient
          │
          ▼
MCP Server localhost:9000
          │
          ▼
FastMCP 工具函数
          │
          ▼
ERP HTTP Client
          │
          ▼
Java ERP REST API
localhost:8081（默认）
          │
          ▼
ERP 业务系统
```

### 7.3 MCP Client

关键文件：

`src/agent/tools/mcp_client.py`

关键代码：

```python
client = MultiServerMCPClient(
    {
        "erp": {
            "url": MCP_SSE_URL,
            "transport": "sse",
        }
    }
)

tools = await client.get_tools()
```

`MultiServerMCPClient` 支持配置多个 MCP Server。当前项目在此处主要连接本地 ERP MCP 服务。

`get_tools()` 动态加载服务端公开的工具，并将其转换为 Agent 可调用的工具对象。

### 7.4 MCP 连接治理

项目实现：

| 机制 | 作用 |
|---|---|
| 最多 3 次连接尝试 | 避免短暂连接失败直接终止 |
| 指数退避 | 在连续失败时增加等待间隔 |
| 300 秒缓存 | 降低重复获取工具列表的开销 |
| 缓存失效 | 服务变化后可强制刷新 |
| 失败返回空列表 | MCP 不可用时允许部分 Agent 功能继续运行 |

主要配置：

```python
MAX_RETRIES = 3
RETRY_BASE_DELAY = 1.0
RETRY_BACKOFF_FACTOR = 2.0
CACHE_TTL = 300
```

注意：连接重试和工具列表缓存不代表每一次业务工具调用都会自动安全重试。订单写操作如果发生异常，还需要考虑幂等性和执行结果确认。

### 7.5 MCP Server

关键文件：

`src/mcp_server/server_main.py`

主要负责：

- 创建 FastMCP 服务。
- 注册 ERP 业务工具。
- 暴露 SSE 接口。
- 接收来自 MCP Client 的调用请求。

### 7.6 ERP 工具分类

项目封装了 23 个 ERP 工具。

| 工具分类 | 数量 | 功能 |
|---|---:|---|
| 供应商管理 | 5 | 供应商查询及管理 |
| 零部件管理 | 5 | 零部件查询及管理 |
| 采购订单 | 7 | 订单创建、查询、更新等 |
| 库存管理 | 6 | 库存查询、预警、入库等 |
| **总计** | **23** | |

### 7.7 MCP 与普通 REST API 的区别

REST API 通常用于普通应用之间的业务通信。

MCP 则进一步提供面向 AI 应用的标准化工具描述、发现和调用机制。

项目的关系是：

**Agent 使用 MCP 调用工具，MCP 工具内部再使用 REST API 访问 Java ERP。**

因此 MCP 并没有替代 ERP 原有的业务逻辑，而是增加了 Agent 可调用的工具接口层。

---

## 八、Harness Engineering 运行时架构

### 8.1 Harness 的核心思想

Harness 指在模型推理之外，围绕 Agent 执行过程提供规划、工具管理、状态控制、约束、审查、恢复等工程能力。

项目通过 Harness 实现对复杂任务执行过程的治理。

### 8.2 四个逻辑阶段

```text
Planning
    ↓
Executing
    ↓
Review
    ↓
Result
```

- Planning：理解目标并准备执行计划。
- Executing：调用工具、执行子任务。
- Review：依据审查策略验证执行结果。
- Result：形成面向用户的结果。

这些属于逻辑阶段，不表示每个请求都一定依次进入四个独立的 LangGraph 节点。

### 8.3 Middleware 中间件

Middleware 可以在 Agent 和模型、工具执行流程中插入控制逻辑。

项目包含多类中间件：

| 中间件 | 主要作用 |
|---|---|
| `HarnessPhaseMiddleware` | 运行阶段及审查策略管理 |
| `ContextInjectionMiddleware` | 注入用户上下文 |
| `WarmMemoryMiddleware` | 注入温记忆 |
| `ProgressiveSkillsMiddleware` | 按需披露 Skills |
| `ToolsSummarizationMiddleware` | 工具调用过程与摘要管理 |
| `MemoryUpdateMiddleware` | 用户偏好更新 |
| `MemoryConsolidationMiddleware` | 情节归档与遗忘 |
| `SandboxHealthMiddleware` | 沙箱健康检查 |
| `SandboxCircuitBreakerMiddleware` | 沙箱异常熔断 |
| `SafeRubricMiddleware` | 执行结果评审 |
| `ReviewExecutionGate` | 根据执行结果升级审查 |
| `ModelCallLimitMiddleware` | 限制模型调用次数 |
| `ToolCallLimitMiddleware` | 限制工具调用次数 |

### 8.4 Hybrid Review Router

项目采用规则和 LLM 结合的动态审查机制。

示意流程：

```text
用户请求
    │
    ▼
判断任务是否明确、简单
    │
    ├── 是 → 可跳过额外审查
    │
    └── 否
         │
         ▼
     规则是否能判定
         │
         ├── 是 → 规则路由
         │
         └── 否 → LLM 辅助判断
                       │
                       ▼
                 按策略执行 Review
```

这种方法试图在审查成本和系统可靠性之间取得平衡。

此外，工具异常、业务写操作及其他执行信号可能影响最终审查级别。

### 8.5 Review 与 HITL 的区别

Review 主要用于系统内部的结果质量检查。

HITL（Human-in-the-Loop）用于需要用户参与决策、确认或批准的业务操作。

例如：

- 检查供应商比较结果是否符合约束：Review。
- 在创建真实采购订单前请求用户确认：HITL。

两者可以同时使用，但不能互相替代。

---

## 九、长期记忆架构

### 9.1 为什么需要长期记忆？

普通 Agent 即使具备会话历史，也可能缺少对跨会话用户偏好、长期事件和可复用经验的持续管理。

ERP-Agent 使用分层记忆策略，让不同类型的信息通过不同路径被使用。

### 9.2 HOT / WARM / COLD

| 记忆层级 | 内容 | 访问策略 |
|---|---|---|
| HOT | 当前对话、任务计划、工具结果 | 当前执行上下文 |
| WARM | 稳定偏好、近期情节摘要 | 调用模型前主动注入 |
| COLD | 长期情节、历史事实、经验 | 按需检索 |

### 9.3 HOT 热记忆

HOT 主要与当前执行状态相关。

通过 LangGraph State 和 Checkpointer 管理当前会话、执行状态和恢复信息。

### 9.4 WARM 温记忆

WARM 保存能够在后续任务中持续发挥作用的用户信息。

例如：

> 用户习惯优先选择信用评级高的供应商。

后续执行供应商推荐任务时，系统可以主动将该偏好注入 Agent 上下文。

项目对温记忆注入设置了字符量限制，以控制上下文大小。

### 9.5 COLD 冷记忆

COLD 用于保存不适合每次全部注入 Prompt 的长期信息。

Agent 可以按需使用 `read_memory` 等工具检索。

当前代码的相关排序逻辑结合了 BM25、时间衰减、访问频次、置信度等因素。

这不同于直接使用 Milvus 或 Qdrant 进行向量相似度检索。

### 9.6 主要文件

| 文件 | 职责 |
|---|---|
| `memory/keeper.py` | 统一记忆管理 |
| `memory/types.py` | 记忆数据结构 |
| `memory/extractor.py` | 偏好提取和情节总结 |
| `memory/scoring.py` | 检索和记忆评分 |
| `middlewares/warm_memory.py` | 温记忆注入 |
| `tools/memory_tools.py` | 记忆读写工具 |
| `mongodb_store.py` | MongoDB Store 实现 |

### 9.7 MongoDB Checkpointer 和 Store

Checkpointer 主要负责保存执行状态，支持 Agent 中断与恢复。

Store 主要负责跨会话的持久化信息存储。

二者解决的问题不同：

**Checkpointer 偏执行过程，Store 偏长期信息。**

---

## 十、Skills 技能系统

### 10.1 Skills 是什么？

Skill 是一种可复用的任务知识与操作规范。

与 MCP Tool 不同：

- MCP Tool 负责实际执行某项操作。
- Skill 负责告诉 Agent 某类任务应该如何完成。

例如：

MCP Tool 负责查询供应商报价。

供应商分析 Skill 负责说明如何比较供应商、如何形成评分和输出报告。

### 10.2 渐进式加载

项目采用 Skills Progressive Disclosure 思想。

```text
加载 Skills Catalog
         ↓
提供技能名称和描述
         ↓
Agent 选择所需技能
         ↓
read_skill()
         ↓
读取完整技能说明
         ↓
按需读取相关资源
```

这样无需在每个请求中将全部 Skill 内容放进 System Prompt。

### 10.3 工程价值

- 降低初始上下文占用。
- 减少无关业务指令干扰。
- 让技能知识能够独立维护。
- 支持增加新业务技能。
- 便于复用已有操作规范。

仓库相关专项测试记录了基础技能提示词片段的字符量显著降低，但该结果不能直接等同于完整对话 Token 成本下降相同比例。

---

## 十一、Sandbox 沙箱执行架构

### 11.1 为什么需要沙箱？

在业务分析中，Agent 可能需要：

- 执行 Python 代码。
- 处理 CSV 或 Excel 数据。
- 生成图表。
- 创建和编辑文件。
- 执行自动化脚本。

如果直接使用宿主机环境执行不可信代码，可能产生安全风险。

### 11.2 项目沙箱架构

```text
Agent
   │
   ▼
CompositeBackend
   │
   ▼
SandboxBackendProxy
   │
   ▼
SandboxManager
   │
   ▼
Docker Sandbox
   │
   ▼
执行代码 / 文件操作
   │
   ▼
返回结果
```

### 11.3 核心组件

| 文件 | 职责 |
|---|---|
| `sandbox_setup.py` | 配置和创建容器 |
| `sandbox_manager.py` | 沙箱生命周期管理 |
| `sandbox_proxy.py` | 沙箱后端引用与替换 |
| `custom_opensandbox.py` | Docker 容器操作封装 |
| `sandbox_holder.py` | 请求范围内的沙箱上下文 |
| `seccomp.json` | 系统调用限制配置 |

### 11.4 工程机制

系统设计涉及：

- 沙箱创建。
- 沙箱预热。
- 健康检查。
- 故障重建。
- 熔断与恢复。
- 用户执行环境管理。
- 资源限制和部分权限控制。

### 11.5 当前需要注意的问题

Docker 不可用时，项目可能回退到 LocalShellBackend。

LocalShell 不是等价于 Docker 容器的隔离环境。

同时，默认使用 Docker bridge 网络也不意味着容器完全断网。

因此该模块应被理解为：

**项目提供 Docker 沙箱执行与容错能力，但实际安全边界取决于部署配置和是否启用了回退路径。**

---

## 十二、完整采购请求的执行流程

以下以一个典型的库存补货任务为例。

用户输入：

> 帮我查询某零部件的库存，如果不足 100 件，就推荐供应商，并创建补货订单。

### 第一步：前端发送请求

前端将用户消息发送到：

`POST /api/chat/stream`

### 第二步：FastAPI 接收请求

后端处理会话标识和用户上下文，通过 Agent Loader 获取对应的 Agent Session。

### 第三步：恢复状态

如果属于已有会话，通过对应的 `thread_id` 恢复 LangGraph 执行状态。

### 第四步：主 Agent 制定计划

DeepSeek 判断任务需要查询库存、零部件和供应商，并确定是否委派采购分析子 Agent。

### 第五步：MCP 查询库存和供应商

Agent 调用相应 MCP Tool。

```text
Agent
  ↓
MCP Client
  ↓
MCP Server
  ↓
ERP REST API
  ↓
库存 / 供应商数据
```

### 第六步：子 Agent 生成采购分析

采购分析子 Agent 汇总数据，比较供应商和库存缺口。

### 第七步：订单管理子 Agent 准备订单

主 Agent 根据结果调用订单管理子 Agent。

如果缺少必要字段，可以请求用户补充。

### 第八步：HITL 人工审批

当订单创建等高风险操作触发 `interrupt_on` 时，LangGraph 暂停执行。

前端展示审批信息。

用户可以依据系统支持的方式批准、拒绝或要求修改。

### 第九步：Agent 恢复执行

用户审批后调用：

`POST /api/chat/{thread_id}/resume`

通过 `Command(resume=...)` 恢复原有 Agent 执行流程。

### 第十步：执行 Review

根据执行信号和 Review Policy 判断是否需要审查。

### 第十一步：返回最终结果

FastAPI 将执行结果转换为 SSE 事件。

前端实时展示处理状态和最终结果。

### 第十二步：持久化

MongoDB 保存会话执行状态和相关信息。

Memory Middleware 按规则更新用户偏好或归档历史情节。

**注意：** 这里描述的是有条件的业务调用路径，并不是每个请求都必须触发所有子 Agent、所有工具和人工审批。

---

## 十三、关键源码学习清单

以下是建议优先阅读的核心文件，而非仓库全部文件的逐一枚举。

| 顺序 | 文件 | 学习目标 |
|---|---|---|
| 1 | `src/api_view/web_main.py` | 理解 FastAPI 启动 |
| 2 | `src/api_view/api/chat.py` | 理解聊天请求与 SSE |
| 3 | `src/api_view/agent_loader.py` | 理解 Agent 的创建与管理 |
| 4 | `src/agent/main_agent.py` | 理解整个 Agent 组装过程 |
| 5 | `src/agent/config.py` | 理解模型和基础配置 |
| 6 | `src/agent/subagents/loader.py` | 理解子 Agent 动态加载 |
| 7 | `src/agent/subagents/configs/*.yaml` | 理解职责和工具声明 |
| 8 | `src/agent/tools/mcp_client.py` | 理解 MCP 客户端 |
| 9 | `src/mcp_server/server_main.py` | 理解 MCP 服务端 |
| 10 | `src/mcp_server/tools/*.py` | 理解 ERP 工具封装 |
| 11 | `src/mcp_server/http_base.py` | 理解 REST API 转发 |
| 12 | `src/agent/harness.py` | 理解运行阶段 |
| 13 | `src/agent/review_policy.py` | 理解动态审查 |
| 14 | `src/agent/middlewares/review_gate.py` | 理解审查升级 |
| 15 | `src/agent/memory/keeper.py` | 理解记忆治理 |
| 16 | `src/agent/progressive_skills.py` | 理解 Skills 渐进加载 |
| 17 | `src/agent/backends/sandbox_manager.py` | 理解沙箱生命周期 |
| 18 | `frontend/src/hooks/useChat.ts` | 理解前端聊天状态 |
| 19 | `frontend/src/hooks/useSSE.ts` | 理解 SSE 接收 |
| 20 | `frontend/src/lib/api.ts` | 理解前后端请求封装 |

### 最重要的三个文件

**第一：`chat.py`**

弄清楚用户请求如何进入系统，以及 Agent 执行结果如何返回前端。

**第二：`main_agent.py`**

弄清楚整个 Agent 如何由模型、工具、子 Agent、中间件、Memory 和 Sandbox 组装而成。

**第三：`mcp_client.py`**

弄清楚 Agent 如何连接 MCP Server，以及如何加载和调用 ERP 业务工具。

---

## 十四、项目评测和结果

仓库记录了 Review Router、记忆、采购业务验收等测试结果。

| 指标 | 仓库记录结果 | 说明 |
|---|---|---|
| Review Router 标签一致率 | 93.46%（100/107） | 模糊请求测试 |
| 应审查召回率 | 88.33%（53/60） | 路由模块 |
| Review 路由 P95 | 约 1.13 秒 | 路由延迟 |
| 采购业务自动验收 | 36/36 | 模拟依赖下的自动测试 |
| 温记忆注入上限 | 4000 字符 | 上下文控制配置 |

这些结果来自开源项目的现有记录，不应直接解释为完整 Agent 在真实 ERP 生产环境下的成功率。

如果用于简历，需要先确认自己是否完成了相关代码开发、实验复现或测试改进。

---

## 十五、项目目前的技术边界

| 问题 | 说明 |
|---|---|
| 外部 ERP 依赖 | Python Agent 仓库不包含完整 Java ERP 业务后端 |
| 沙箱回退 | Docker 不可用时可能使用 LocalShell |
| MCP 可靠性 | 连接重试、缓存不等于所有工具操作都具备幂等保护 |
| Memory 扩展性 | 长期记忆检索和大规模数据治理仍可优化 |
| 身份与权限 | 示例环境不能直接认定具有生产级用户鉴权 |
| 端到端评测 | 模拟环境测试不能替代真实 ERP 集成测试 |
| Review 有效性 | 模块路由指标不能直接当作整体任务成功率 |

---

## 十六、适用于面试的项目介绍

ERP-Agent 是一个面向企业采购管理场景的多智能体系统，基于 DeepAgents 和 LangGraph 构建，主要解决传统 ERP 中跨多个业务模块手工查询、分析和操作的问题。

整体采用前后端分离架构，前端使用 Next.js，后端使用 FastAPI，并通过 SSE 返回 Agent 的流式执行结果。

核心 Agent 采用主 Agent 与采购分析、订单管理两个专业子 Agent 的分层架构，通过任务委派与独立上下文完成复杂采购任务。

在工具调用方面，项目基于 FastMCP 封装 23 个 ERP 业务工具，通过 MCP Client 动态发现工具并访问外部 Java ERP 服务。

在 Agent 运行可靠性方面，项目引入 Harness Engineering 机制，整合任务规划、Middleware、动态 Review、HITL 人工审批以及 Docker Sandbox。

在状态和记忆方面，项目通过 MongoDB Checkpointer 和 Store 实现执行状态持久化与跨会话长期记忆，并通过 HOT/WARM/COLD 分层访问策略管理不同类型的记忆信息。

此外，系统支持 Skills 渐进式加载，减少无关技能内容对模型上下文的占用，并提供 Review Router 和采购业务测试，用于评估关键模块的可靠性。

**面试注意：** 上述内容是系统整体架构介绍。谈及个人工作时，必须明确区分开源仓库原有功能与本人实际参与开发、复现和改进的部分。

---

## 十七、推荐的源码学习顺序

### 第一阶段：请求链路

阅读：

1. `web_main.py`
2. `chat.py`
3. `agent_loader.py`

目标：理解前端请求如何到达 Agent，以及 SSE 如何工作。

### 第二阶段：Agent 组装

阅读：

1. `main_agent.py`
2. `config.py`
3. `subagents/loader.py`
4. 子 Agent YAML 配置

目标：理解模型、工具、专业 Agent 如何组合。

### 第三阶段：MCP 工具链

阅读：

1. `mcp_client.py`
2. `server_main.py`
3. ERP Tools
4. `http_base.py`

目标：理解 AI Agent 如何真正操作 ERP。

### 第四阶段：Harness 与可靠性

阅读：

1. `harness.py`
2. Middleware 相关文件
3. `review_policy.py`
4. `review_gate.py`

目标：理解系统如何控制工具调用、审查任务和处理异常。

### 第五阶段：状态与记忆

阅读：

1. `mongodb_store.py`
2. `memory/keeper.py`
3. `memory/extractor.py`
4. `memory/scoring.py`

目标：理解 Checkpointer、Store、用户偏好和长期记忆。

### 第六阶段：工程能力

阅读：

1. Sandbox 相关代码
2. Skills 相关代码
3. 评测与自动测试代码

目标：理解部署、安全、容错、性能和评测机制。

---

## 十八、总结

ERP-Agent 最重要的架构关系可以概括为：

**FastAPI 接收请求 → DeepAgents / LangGraph 编排任务 → MCP 调用 ERP → Middleware / Review / HITL 控制执行 → MongoDB 保存状态 → SSE 返回结果。**

从技术学习角度，应该重点掌握：

- **Agent 编排：** Main Agent、SubAgent、Task Delegation。
- **工具调用：** MCP Client、MCP Server、Tool Calling。
- **执行控制：** LangGraph State、Interrupt、Resume。
- **工程可靠性：** Middleware、Review、重试、熔断。
- **长期记忆：** MongoDB Checkpointer、Store、HOT/WARM/COLD。
- **上下文工程：** SubAgent 隔离、Skills 渐进式加载。
- **安全执行：** Docker Sandbox、HITL。
- **前后端交互：** FastAPI、SSE、Next.js。

最终学习目标不是简单地把项目运行起来，而是能够清楚解释每一个核心模块为什么存在、如何实现、与其他模块怎样交互，以及在异常情况下系统如何处理。

---

*文档说明：本整理基于此前的项目架构解析及公开仓库代码，属于架构学习文档，不是 ZIP 压缩包内全部文件的逐文件代码审计。*
