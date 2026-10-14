# ERP 智能采购助手测试报告

## 1. 测试结论

截至 2026-10-13，项目已完成本地服务联通、Agent 对话、MCP 工具发现、真实 ERP 读写、库存异常保护和订单 HITL 审批测试，可作为面试演示 MVP 使用。

“通过”表示当前环境中实际观察到预期结果，不代表生产级性能、安全或高可用认证。

## 2. 测试环境

| 项目 | 当前环境 |
|---|---|
| 操作系统 | Windows，PowerShell |
| Python | 项目 `.venv`，Python 3.11+ 兼容依赖 |
| 前端 | Next.js 16.2.12、React 19 |
| Agent API | FastAPI/Uvicorn，`127.0.0.1:8000` |
| MCP Server | FastMCP SSE，`127.0.0.1:9000` |
| MongoDB | 项目本地数据目录，`127.0.0.1:27017` |
| 沙箱 | Docker Desktop，`python:3.11-slim` |
| ERP | 课程老师授权的外部 ERP API |

## 3. 服务与构建验证

| 检查项 | 预期结果 | 当前结果 |
|---|---|---|
| FastAPI `/health` | 返回 `status=ok` | 通过 |
| 前端首页 | HTTP 200 | 通过 |
| MCP 工具加载 | 发现 23 个 ERP 工具 | 通过 |
| 前端正式构建 | TypeScript 与 Next.js 构建成功 | 通过 |
| 记忆层脚本测试 | 34/34 | 通过 |
| 记忆集成选定测试 | 7/7 | 通过 |
| Python 完整组合回归 | 见第 6 节 | 未全部完成，不能写成全绿 |

## 4. 真实 ERP 业务测试

### 4.1 只读查询

| 场景 | 使用工具 | 结果 |
|---|---|---|
| 供应商列表 | `supplier_page` | 返回真实供应商数据 |
| 零部件列表 | `part_page` | 返回真实零部件数据 |
| 库存预警 | `inventory_warning` | 返回成功，当前数据为空列表 |
| 采购订单列表 | `order_page` | 返回真实订单及状态 |

### 4.2 供应商状态

- 创建测试供应商成功：ID `40`，编码 `TEST-TENOWE-20261013-01`。
- 将状态从 `1` 更新为 `0`，查询确认成功。
- 将状态从 `0` 恢复为 `1`，查询确认成功。
- 其他字段保持不变。

### 4.3 零部件

- 创建测试零部件成功：ID `71`。
- 名称：`Tenowe课程测试刹车片`。
- 采购价：`25.50`；建议零售价：`39.90`；预警值：`10`。
- 关联供应商 ID：`40`。
- 创建后按名称查询到唯一匹配记录。

### 4.4 库存正常流程

- 对零部件 ID `71` 入库 `20` 件。
- 查询确认当前库存为 `20`。
- 出库 `5` 件。
- 再次查询确认库存从 `20` 变为 `15`。

### 4.5 库存异常流程

- 当前库存为 `15` 时请求出库 `9999` 件。
- ERP 返回 `code=603`、`库存不足`。
- 查询确认最终库存仍为 `15`，没有发生负库存或部分扣减。

### 4.6 采购订单与 HITL

- 请求创建订单 `PO-TEST-TENOWE-20261013-01`。
- Agent 在执行 `order_create` 前暂停并展示完整审批卡片。
- 用户批准后只调用一次 `order_create`。
- 创建成功：订单 ID `1000046`，总金额 `255.00`，状态 `0（待审核）`。
- 订单明细：零部件 ID `71`、数量 `10`、单价 `25.50`、小计 `255.00`。

## 5. HITL 缺陷修复回归

### 原始问题

LangGraph checkpoint 显示会话已经中断，但前端只显示 `order_create` 一直运行，没有审批按钮，输入框也无法继续正常交互。

### 根因

- 子 Agent 中断可能嵌套在 `updates` 流或子图数据内。
- 部分中断只保存在 checkpoint，没有出现在外层流的顶层字段。
- 后端中断载荷、前端 `event.data` 和恢复请求字段不一致。

### 修复验证

| 验证点 | 结果 |
|---|---|
| 新订单请求能进入人工审批 | 通过 |
| 审批卡片显示完整订单参数 | 通过 |
| 页面刷新后重新选择会话仍能恢复审批 | 通过 |
| 点击批准后继续同一 checkpoint | 通过 |
| `order_create` 未重复执行 | 通过 |
| 订单真实创建成功 | 通过 |

## 6. 自动化回归命令

本次实际通过的命令：

```powershell
$env:PYTHONUTF8="1"
.\.venv\Scripts\python.exe -m src.test.test_memory_layer
```

结果：`34/34 passed`。

```powershell
$env:PYTHONUTF8="1"
.\.venv\Scripts\python.exe -m unittest `
  src.test.test_memory_integration.MemoryIntegrationTests.test_sync_graph_and_user_isolation `
  src.test.test_memory_integration.MemoryIntegrationTests.test_missing_thread_does_not_write_shared_unknown `
  src.test.test_memory_integration.MemoryIntegrationTests.test_negation_correction_and_temporary_requests `
  src.test.test_memory_integration.MemoryIntegrationTests.test_old_turn_does_not_overwrite_newer_preference `
  src.test.test_memory_integration.MemoryIntegrationTests.test_env_defaults_are_loaded `
  src.test.test_memory_integration.MemoryIntegrationTests.test_llm_extraction_enabled_and_fallback `
  src.test.test_memory_integration.MemoryIntegrationTests.test_warm_prompt_is_bounded_and_store_failure_safe -q
```

结果：`Ran 7 tests ... OK`。

前端验证：

```powershell
Set-Location .\frontend
npm run build
```

本文件中的自动化测试数量应以本次实际运行结果为准，不沿用旧记录中的数字。

### 本次未能计入通过数的项目

- 完整组合回归在 `test_real_async_graph_keeps_separate_threads_and_refreshes_memory` 长时间未结束后被人工停止；没有把它计为通过或失败。
- `test_sandbox_lifecycle` 共运行 11 项，其中 9 项通过、1 项因当时本机使用 `review.mode=never` 与测试的自动 Review 预期不一致而失败、1 项因受限 Windows 临时目录无权限而报错。
- `test_hybrid_review` 与 `test_jev_router` 在本次环境中未在限定时间内结束，因此未记录通过数。

这些自动化问题不改变第 4、5 节已经观察到的真实 ERP 与 HITL 结果，但说明当前测试套件还不能宣称“全部通过”。

## 7. 已知边界

- 外部 ERP 源码和数据库不在本仓库中，业务数据依赖老师提供的服务。
- 该轮演示时本机使用 `review.mode=never`，自动 Grader 未参与；仓库发布前已恢复默认 `auto`，HITL 始终不受影响。
- Windows Docker Desktop 无法加载当前 seccomp 配置时会回退到基础沙箱模式。
- 未进行压力测试、并发容量测试、渗透测试和公网部署验证。
- 未实现生产级身份认证、RBAC、审计合规与灾备。
- 测试供应商、零部件、库存和订单数据仍保留在课程 ERP 中，清理需另行确认。
