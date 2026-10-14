# ERP 智能采购助手演示指南

## 1. 演示目标

用 5–8 分钟证明三件事：

1. Agent 能通过 MCP 查询真实 ERP 数据。
2. Agent 能处理库存业务和异常边界。
3. 订单写操作必须经过人工审批，且审批后只执行一次。

## 2. 演示前检查

- Docker Desktop 已启动。
- `.env` 已配置 DeepSeek Key、MongoDB、MCP 和老师授权的 ERP 地址。
- 不在演示屏幕上打开 `.env`，避免泄露密钥。
- 项目目录：`D:\Agent开发\ERP-AGENT-open-source`。
- 仓库默认启用 `review.mode=auto`；若演示所用模型端点不兼容结构化 Grader，可在本地临时改为 `never`，HITL 不受影响。

## 3. Windows 启动顺序

打开四个 PowerShell 终端，均从项目根目录开始。

### 终端 1：MongoDB

```powershell
& ".\.local\mongodb\mongodb-win32-x86_64-windows-8.0.32\bin\mongod.exe" `
  --dbpath ".\.local\mongodb-data" `
  --bind_ip 127.0.0.1 `
  --port 27017
```

用途：保存会话、checkpoint、长期记忆和沙箱映射。

### 终端 2：MCP Server

```powershell
$env:PYTHONUTF8="1"
.\.venv\Scripts\python.exe -m src.mcp_server.server_main
```

用途：把外部 ERP REST API 暴露为 23 个 Agent 工具。

### 终端 3：Agent API

```powershell
$env:PYTHONUTF8="1"
.\.venv\Scripts\python.exe -m uvicorn src.api_view.web_main:app `
  --host 127.0.0.1 `
  --port 8000
```

用途：创建 Agent Session、执行 LangGraph，并通过 SSE 向前端推送事件。

### 终端 4：前端

```powershell
Set-Location .\frontend
npm run dev
```

用途：启动 Next.js 对话界面。

浏览器访问：`http://127.0.0.1:3000`。

## 4. 启动成功信号

| 服务 | 检查方法 | 成功信号 |
|---|---|---|
| MongoDB | 观察终端 | 正在监听 27017 |
| MCP | 观察终端 | Server 运行于 9000 |
| Agent API | 打开 `http://127.0.0.1:8000/health` | `{"status":"ok"}` |
| 前端 | 打开 `http://127.0.0.1:3000` | 显示智能采购助手 |
| Docker | `docker ps --filter "name=erp-sandbox"` | 显示用户或预热沙箱 |

首次对话需要创建 Agent、连接 MCP、恢复沙箱和同步 Skills，通常比后续请求慢。

## 5. 推荐演示脚本

### 第一步：只读查询（约 1 分钟）

发送：

```text
只调用一次 supplier_page，查询供应商列表，只显示前5条，不执行任何新增、修改或删除操作。
```

讲解重点：自然语言被转换为标准 MCP 工具参数，返回真实 ERP 数据。

### 第二步：跨模块查询（约 1 分钟）

发送：

```text
只调用一次 part_by_supplier，查询供应商ID 40提供的零部件，不执行任何写操作。
```

讲解重点：Agent 不直接拼接 HTTP，而是通过 MCP 工具访问 ERP。

### 第三步：库存边界（约 1 分钟，可选）

为避免改变数据，现场优先展示已有测试报告，不重复执行超量出库。说明此前已验证库存为 15 时出库 9999，ERP 返回 603 且库存保持 15。

### 第四步：HITL 审批（约 2–3 分钟）

不要重复创建已经存在的订单号。演示时使用新的唯一订单号，例如带当前日期时间，并明确先等待审批：

```text
请只调用一次 order_create，创建一条课程演示采购订单。使用唯一订单号，零部件ID为71，数量为1，单价为25.50，状态为0。出现人工审批后暂停，等待我选择；不要自动批准，不要重复创建，也不要调用其他工具。
```

在审批卡片出现后先讲解：

- LangGraph 已把待执行动作写入 checkpoint。
- 此时 ERP 尚未创建订单。
- 刷新页面后仍可从历史会话恢复审批。

若允许写入测试数据，再点击“批准执行”；否则点击“拒绝”。

### 第五步：总结（约 30 秒）

“这个演示覆盖了真实 ERP 查询、MCP 工具调用、LangGraph 状态持久化和写操作 HITL。系统不是只生成答案，而是能在权限控制下执行真实业务操作。”

## 6. 面试现场故障处理

| 现象 | 优先检查 |
|---|---|
| 前端打不开 | 3000 端口和 `npm run dev` |
| `/health` 不通 | 8000 端口和 Uvicorn 日志 |
| Agent 初始化很久 | MCP 9000、Docker、首次 Skills 同步 |
| ERP 工具返回 502 | 外部 ERP 地址是否可达 |
| Python 输出乱码或异常 | 是否设置 `PYTHONUTF8=1` |
| 审批卡片未显示 | 刷新页面，重新选择对应历史会话，检查 `/api/chat/{thread_id}/state` |
| Docker 安全模式失败 | 查看是否已回退到基础沙箱；不要宣称完整隔离生效 |

## 7. 演示结束

在四个服务终端分别按 `Ctrl+C`。Docker 用户沙箱是否保留由沙箱生命周期管理器决定；MongoDB 中的会话、记忆和业务测试记录不会因为停止本地服务自动删除。
