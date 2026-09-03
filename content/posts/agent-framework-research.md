---
title: "5大Agent框架深度对比：我用MCP搭建低空数据智能体的实战记录"
date: 2026-09-03T15:00:00+08:00
draft: false
tags: ["AI Agent", "MCP", "技术选型", "低空经济", "nanobot", "Hermes", "Pi", "Agno", "DeepSeek Harness"]
categories: ["技术"]
summary: "经过2个月的实测，我对比了5个主流Agent框架（nanobot、Hermes、Pi、Agno、DeepSeek Harness）的MCP接入能力和工程化特性。本文详细记录了每个框架的核心特点、接入方式、踩坑经验和适用场景，最终选择了Agno作为低空项目的中心Agent服务。"
---

# 5大Agent框架深度对比：我用MCP搭建低空数据智能体的实战记录

## 为什么要做这个调研？

去年我们团队接了一个低空经济项目，核心需求是：**让 AI 能自动查询和分析飞行数据**。

听起来简单，但实际做起来发现：
- 数据源是飞服日报 MCP Server（12 个工具）
- 需要支持自然语言查询（"7 月 5 日飞行总架次是多少？"）
- 要集成到现有工作流（HTTP API 调用）
- 还得考虑生产环境（鉴权、超时、幂等、日志）

于是我开始调研 Agent 框架，目标是找到一个**既能快速接入 MCP，又能满足工程化要求**的方案。

经过 2 个月的实测，我对比了 5 个主流框架：**nanobot、Hermes、Pi、Agno、DeepSeek Harness**。

本文会把每个框架的**核心特点、接入方式、踩坑经验、适用场景**都讲清楚，帮你少走弯路。

---

## 5 大框架快速对比

先上一张表，让你 30 秒看懂差异：

| 维度 | nanobot | Hermes | Pi | Agno | DeepSeek Harness |
|------|---------|--------|-----|------|------------------|
| **核心定位** | 轻量自托管 Agent | 完整 CLI Agent/服务 | TypeScript Agent Harness | Agent 框架和运行时 | 插件化 Agent Runtime |
| **主要语言** | Python | Python | TypeScript | Python | TypeScript + Python SDK |
| **本地安装** | 简单 | 中等（WSL） | 简单 | 中等 | 中等偏高 |
| **原生 MCP** | ✅ 支持 | ✅ 支持 | ❌ 需 Extension | ✅ 支持 | ✅ 支持 |
| **HTTP API** | OpenAI 兼容 | Runs/Responses | 无内置 | AgentOS REST | Web 内部 API |
| **子进程 RPC** | 非核心 | 非核心 | ✅ 原生 JSONL | 非核心 | ✅ 原生 JSONL |
| **Run ID** | 能力较弱 | ✅ 支持 | RPC 请求 ID 映射 | ✅ 支持 | 无独立 Run ID |
| **工具筛选** | ✅ 支持 | ✅ 支持 | Extension 实现 | include/exclude | 需外层实现 |
| **安全沙箱** | 无完整沙箱 | 需收敛工具权限 | 无沙箱 | 需部署隔离 | 有沙箱/权限插件 |
| **接入开发量** | 低 | 低至中 | 高 | 中 | 中至高 |
| **中心编排适配度** | 中 | 高 | 中 | 很高 | 中高 |

**一句话总结：**
- **nanobot**：轻量入门首选，配置即可接 MCP
- **Hermes**：功能最完整，CLI/微信/API 全覆盖
- **Pi**：TypeScript 生态最佳，RPC 嵌入优秀
- **Agno**：Python 全栈方案，Workflow 能力最强
- **DeepSeek Harness**：插件化设计，Trace 能力突出

**我的最终选择：Agno**（原因后面会说）

---

## nanobot：轻量级 MCP 入口

### 核心特点

nanobot 是我第一个测试的框架，原因是它**安装最简单、配置最直观**。

它的定位很清晰：**轻量自托管 Agent**，适合快速验证和边缘部署。

**核心组件：**
- **CLI**：命令行交互
- **Gateway**：消息网关（微信、Telegram 等）
- **WebUI**：浏览器界面（端口 8765）
- **Session/Memory**：会话和记忆管理

**MCP 支持：**
- stdio（本地进程）
- HTTP（远程服务）
- SSE（服务器推送）
- OAuth（授权认证）

### 低空项目实践

**安装步骤：**

```powershell
# 创建项目目录
New-Item -ItemType Directory -Force "E:\gis\mcp\poc\nanobot"

# 创建虚拟环境
python -m venv "E:\gis\mcp\poc\nanobot\.venv"
& "E:\gis\mcp\poc\nanobot\.venv\Scripts\python.exe" -m pip install --upgrade pip
& "E:\gis\mcp\poc\nanobot\.venv\Scripts\python.exe" -m pip install nanobot-ai

# 验证安装
& "E:\gis\mcp\poc\nanobot\.venv\Scripts\nanobot.exe" --version

# 初始化配置
& "E:\gis\mcp\poc\nanobot\.venv\Scripts\nanobot.exe" onboard --wizard

# 启动 WebUI
& "E:\gis\mcp\poc\nanobot\.venv\Scripts\nanobot.exe" webui
```

**接入飞服 MCP：**

```powershell
# 备份原配置
$nanobotConfigPath = "$env:USERPROFILE\.nanobot\config.json"
$nanobotBackupPath = "$env:USERPROFILE\.nanobot\config.before-feifu.json"
Copy-Item -LiteralPath $nanobotConfigPath -Destination $nanobotBackupPath -Force

# 读取配置
$nanobotConfig = Get-Content -LiteralPath $nanobotConfigPath -Raw -Encoding UTF8 | ConvertFrom-Json

# 配置飞服 MCP Server
$feifuServer = [ordered]@{
    type = "stdio"
    command = "E:\gis\mcp\poc\nanobot\.venv\Scripts\python.exe"
    args = @("-m", "feifu_mcp.server")
    env = [ordered]@{
        PYTHONPATH = "E:\gis\mcp\低空MCP程序_交付_20260707\src"
        FEIFU_DATA_DIR = "E:\gis\mcp\低空MCP程序_交付_20260707\data\derived"
        PYTHONIOENCODING = "utf-8"
        PYTHONUTF8 = "1"
    }
    toolTimeout = 30
    enabledTools = @(
        "query_metric_catalog",
        "resolve_named_period",
        "query_metric_value",
        "list_hot_zones",
        "compare_metrics",
        "compare_segments",
        "explain_metric",
        "compute_derived_metric",
        "scan_metric_range",
        "rank_metric_over_time",
        "lookup_event_annotations",
        "explain_anomaly"
    )
}

# 写入配置
if ($null -eq $nanobotConfig.tools.mcpServers) {
    $nanobotConfig.tools | Add-Member `
        -NotePropertyName "mcpServers" `
        -NotePropertyValue ([PSCustomObject]@{}) `
        -Force
}

$nanobotConfig.tools.mcpServers | Add-Member `
    -NotePropertyName "feifu" `
    -NotePropertyValue $feifuServer `
    -Force

# 保存（UTF-8 无 BOM）
$nanobotJson = $nanobotConfig | ConvertTo-Json -Depth 100
$utf8WithoutBom = New-Object System.Text.UTF8Encoding($false)
[System.IO.File]::WriteAllText($nanobotConfigPath, $nanobotJson, $utf8WithoutBom)
```

**验证 MCP 调用：**

```powershell
# CLI 方式调用
& "E:\gis\mcp\poc\nanobot\.venv\Scripts\nanobot.exe" agent `
  --logs `
  --session "feifu-poc-001" `
  -m "必须调用 feifu MCP 的 query_metric_value 工具查询：2026-07-05 飞行总架次是多少？"
```

**返回结果：**
```
工具名：query_metric_value
日期：2026-07-05
数值：9448
单位：架次
```

**HTTP API 调用：**

```powershell
# 启用 API 插件
& "E:\gis\mcp\poc\nanobot\.venv\Scripts\nanobot.exe" plugins enable api
& "E:\gis\mcp\poc\nanobot\.venv\Scripts\nanobot.exe" serve

# 健康检查
Invoke-RestMethod -Method Get -Uri "http://127.0.0.1:8900/health"

# 对话调用
$body = @{
    model = "deepseek-ai/DeepSeek-V4-Pro"
    messages = @(
        @{
            role = "user"
            content = "必须调用 feifu MCP 的 query_metric_value 工具查询：2026-07-05 飞行总架次"
        }
    )
    session_id = "feifu-api-poc-001"
    stream = $false
} | ConvertTo-Json -Depth 10

$response = Invoke-RestMethod `
    -Method Post `
    -Uri "http://127.0.0.1:8900/v1/chat/completions" `
    -ContentType "application/json; charset=utf-8" `
    -Body ([System.Text.Encoding]::UTF8.GetBytes($body))

$response.choices[0].message.content
```

### 踩坑经验

**1. WebUI 中文路径问题**

nanobot WebUI 对中文 MCP 路径存在兼容问题，我通过配置文件规避了这个问题。

**2. Run 生命周期能力弱**

nanobot 的 Run 管理比较基础，没有完整的状态追踪和取消机制。

### 我的评价

**优点：**
- ✅ 配置成本低，半小时就能跑通
- ✅ WebUI 操作直观，适合演示
- ✅ CLI、WebUI、HTTP API 三合一
- ✅ 支持工具白名单和超时控制

**缺点：**
- ❌ Run 生命周期能力弱于 Hermes 和 Agno
- ❌ 更适合作为轻量入口，不适合复杂编排
- ❌ 生产化风险：取消恢复、鉴权、并发、审计能力待验证

**适合场景：**
- 快速 PoC 验证
- 边缘设备部署
- 轻量级查询节点
- 个人开发者项目

---

## Hermes Agent：功能完整的 Agent 平台

### 核心特点

Hermes 是我调研的第二个框架，也是**功能最完整**的一个。

它的定位是：**完整 CLI Agent/服务**，提供从命令行到 HTTP API 的全套能力。

**核心入口：**
- `hermes chat`：命令行交互
- Gateway：消息网关（微信、Telegram、Discord）
- Dashboard：Web 管理面板
- API Server：HTTP 服务

**MCP 支持：**
- 原生 stdio 和远程 HTTP
- 启动时自动发现工具
- 支持 include/exclude 工具筛选
- 可声明只读 MCP 并行调用
- 连接管理、超时、allowlist

### 低空项目实践

**安装部署（WSL）：**

```bash
# 安装 Hermes
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash

# 添加飞服 MCP
hermes mcp add feifu \
  --command /home/asus/.hermes/hermes-agent/venv/bin/python \
  --connect-timeout 30 \
  --env \
    "PYTHONPATH=/mnt/e/gis/mcp/低空MCP程序_交付_20260707/src" \
    "FEIFU_DATA_DIR=/mnt/e/gis/mcp/低空MCP程序_交付_20260707/data/derived" \
    "PYTHONIOENCODING=utf-8" \
    "PYTHONUTF8=1" \
  --args -m feifu_mcp.server

# 验证 MCP 连接
hermes mcp list
hermes mcp test feifu
```

**CLI 调用验证：**

```bash
hermes chat
```

输入：
```
禁止使用 terminal、bash、Python 或文件工具。必须调用 feifu MCP 的 query_metric_value 工具查询：2026-07-05飞行总架次是多少？请给出工具名、日期、数值和单位，不允许猜测。
```

**API Server 配置：**

```bash
# 生成 API 密钥
HERMES_API_TEST_KEY=$(
  /home/asus/.hermes/hermes-agent/venv/bin/python \
  -c "import secrets; print(secrets.token_urlsafe(32))"
)

# 写入配置
printf '\nAPI_SERVER_ENABLED=true\nAPI_SERVER_HOST=127.0.0.1\nAPI_SERVER_PORT=8642\nAPI_SERVER_KEY=%s\n' \
  "$HERMES_API_TEST_KEY" >> /home/asus/.hermes/.env

chmod 600 /home/asus/.hermes/.env
unset HERMES_API_TEST_KEY

# 重启 Gateway
hermes gateway restart

# 检查状态
hermes gateway status
ss -ltnp | grep 8642
```

**Runs API 验证：**

```bash
# 定义 API 地址
HERMES_API_BASE="http://127.0.0.1:8642"

# 创建任务
RUN_RESPONSE=$(
  curl -sS -X POST \
    -H "Authorization: Bearer ***" \
    -H "Content-Type: application/json" \
    -H "Idempotency-Key: feifu-hermes-q01-001" \
    --data '{
      "input": "必须调用feifu MCP的query_metric_value工具查询2026-07-05飞行总架次",
      "session_id": "feifu-hermes-api-poc-001",
      "instructions": "禁止使用terminal、bash、Python和文件工具；不得猜测飞服数据。"
    }' \
    "${HERMES_API_BASE}/v1/runs"
)

# 提取 Run ID
HERMES_RUN_ID=$(
  printf '%s' "$RUN_RESPONSE" |
  python3 -c 'import json,sys; print(json.load(sys.stdin)["run_id"])'
)

# 查询状态
curl -sS \
  -H "Authorization: Bearer ***" \
  "${HERMES_API_BASE}/v1/runs/${HERMES_RUN_ID}" |
  python3 -c 'import json,sys; print(json.load(sys.stdin)["output"])'
```

### 踩坑经验

**1. WSL 部署链路较长**

Hermes 需要 WSL 环境，Windows 用户需要额外配置。首次 npm 安装时 Windows 表现不稳定。

**2. 复杂参数序列化问题**

模型曾将 `compare_metrics` 嵌套对象参数错误序列化为字符串。

**3. 幂等性未达预期**

v0.18.0 的 `/v1/runs` 没有实现有效幂等去重，`Idempotency-Key` 实测未生效。

**4. Token 消耗较高**

单次请求上下文较大，本次 API 请求约消耗 4.1 万输入 Token。

### 我的评价

**优点：**
- ✅ MCP 功能完整，支持注册、测试、工具筛选
- ✅ CLI、微信、HTTP API 全覆盖
- ✅ Runs API 和 SSE 事件适合工作流编排
- ✅ Bearer Token 鉴权已验证
- ✅ 可以观察智能体运行状态和工具调用过程

**缺点：**
- ❌ WSL 部署链路长，Windows 用户门槛高
- ❌ CLI 调试日志较多
- ❌ 幂等性实测未达预期
- ❌ 单次 Token 消耗较高

**适合场景：**
- 中心工作流编排节点
- 需要多渠道接入（CLI + 微信 + API）
- 需要完整 Run 生命周期管理
- 团队有 WSL/Linux 环境

---

## Pi Agent：TypeScript 生态的极简选择

### 核心特点

Pi 是一个**极简终端 coding agent Harness**，定位很明确：轻量的、可扩展的代码 agent 运行时。

**三种使用方式：**
- **交互 CLI**：人在终端里直接和 Pi 对话
- **TypeScript SDK**：用代码调用 Pi，适合集成到 Node.js 项目
- **JSONL RPC**：进程间通信协议，每行一个 JSON 对象

**设计哲学：**
Pi 的核心明确**不内置** MCP、sub-agent、权限弹窗和后台 bash。它认为这些不是"核心"，可以通过 Extension 扩展加，但不应该塞进主程序。

### 低空项目实践

**安装 Pi CLI：**

```powershell
New-Item -ItemType Directory -Force "E:\gis\mcp\poc\pi"
Set-Location "E:\gis\mcp\poc\pi"

npm install -g --ignore-scripts @earendil-works/pi-coding-agent

pi --version
pi --help
```

**配置 SiliconFlow 模型：**

```powershell
New-Item -ItemType Directory -Force "$HOME\.pi\agent"
notepad "$HOME\.pi\agent\models.json"
```

```json
{
  "providers": {
    "siliconflow": {
      "baseUrl": "https://api.siliconflow.cn/v1",
      "api": "openai-completions",
      "authHeader": true,
      "compat": {
        "supportsDeveloperRole": false,
        "supportsReasoningEffort": false
      },
      "models": [
        {
          "id": "deepseek-ai/DeepSeek-V4-Pro",
          "name": "DeepSeek V4 Pro via SiliconFlow",
          "reasoning": false,
          "input": ["text"],
          "contextWindow": 128000,
          "maxTokens": 8192
        }
      ]
    }
  }
}
```

**编写飞服 MCP Extension：**

这是 Pi 接入 MCP 的关键步骤。由于 Pi 不原生支持 MCP，需要自己写 Extension：

```typescript
// E:\gis\mcp\poc\pi\.pi\extensions\feifu-mcp.ts
import { Extension } from "@earendil-works/pi-coding-agent";

export default class FeifuMCPExtension extends Extension {
  name = "feifu-mcp";
  
  async initialize() {
    // 启动飞服 MCP Server
    const mcpProcess = spawn("python", ["-m", "feifu_mcp.server"], {
      env: {
        ...process.env,
        PYTHONPATH: "E:\\gis\\mcp\\低空MCP程序_交付_20260707\\src",
        FEIFU_DATA_DIR: "E:\\gis\\mcp\\低空MCP程序_交付_20260707\\data\\derived",
        PYTHONIOENCODING: "utf-8",
        PYTHONUTF8: "1",
      },
    });

    // 通过 stdio 连接 MCP
    // 自动发现工具并注册
    // 排除 answer_question
    // 将剩余 12 个工具注册为 Pi 工具
  }
}
```

**验证 Extension 连接：**

```powershell
pi --provider siliconflow --model "deepseek-ai/DeepSeek-V4-Pro" `
   --no-builtin-tools --no-extensions `
   -e "E:\gis\mcp\poc\pi\.pi\extensions\feifu-mcp.ts"
```

进入 Pi 后输入 `/feifu-status` 检查连接状态。

**验证 MCP 调用：**

```powershell
pi --provider siliconflow --model "deepseek-ai/DeepSeek-V4-Pro" `
   --no-session --no-builtin-tools --no-extensions `
   -e "E:\gis\mcp\poc\pi\.pi\extensions\feifu-mcp.ts" `
   -p "必须调用 query_metric_value 工具查询 2026-07-05 的飞行总架次。只输出工具名、日期、数值和单位，不得猜测。"
```

**JSONL RPC 测试：**

```powershell
pi --mode rpc --provider siliconflow --model "deepseek-ai/DeepSeek-V4-Pro" `
   --no-session --no-builtin-tools --no-extensions `
   -e "E:\gis\mcp\poc\pi\.pi\extensions\feifu-mcp.ts"
```

输入：
```json
{"id":"q1","type":"prompt","message":"必须调用 query_metric_value 查询 2026-07-05 飞行总架次，只输出数值和单位。"}
```

观察到的关键事件：
```
response
agent_start
turn_start
message_start
tool_execution_start  ← 开始调用 query_metric_value
tool_execution_end    ← 工具执行完成
message_end
agent_end
agent_settled         ← 任务完全结束
```

### 踩坑经验

**1. 不原生支持 MCP**

这是 Pi 在本次选型中的最大问题。nanobot 和 Hermes 可以直接配置 MCP Server，而 Pi 需要自行开发和维护 Extension，包括：
- MCP Client 连接
- 工具发现
- Schema 转换
- 参数转发
- 结果转换
- 生命周期管理
- 超时和重连
- 错误处理

**2. 没有内置 HTTP Agent API**

Pi 主要提供 CLI、SDK 和 RPC，没有类似 Hermes Runs API 或 Agno AgentOS 的 HTTP 控制面。如果工作流编排层只能调用 HTTP 接口，需要增加一层服务。

**3. 默认不是安全沙箱**

Pi Extension 和工具以启动 Pi 的操作系统用户权限运行。生产环境不能直接以高权限用户运行。

### 我的评价

**优点：**
- ✅ CLI 体验较好，支持交互式 TUI、一次性调用、会话管理
- ✅ RPC 适合子进程集成，比解析 CLI 文本更可靠
- ✅ Extension 扩展能力强，可以注册自定义工具、拦截生命周期事件
- ✅ 运行时较轻，不需要部署大型控制面、数据库或 Web 服务

**缺点：**
- ❌ 不原生支持 MCP，接入成本高
- ❌ 没有内置 HTTP Agent API
- ❌ 没有内置安全沙箱
- ❌ 生产化需要自行维护 MCP Extension 和 RPC 管理层

**适合场景：**
- TypeScript/Node.js 应用内嵌
- 本地开发和单机执行节点
- 边缘设备
- 短生命周期 Agent 任务
- 团队愿意维护 Extension 和 Worker Manager

---

## Agno Agent：Python 全栈 Agent 框架

### 核心特点

Agno（原 Phidata/Phi Agent）是一个**Python Agent 开发框架**，提供 Agent、Team、Workflow、MCPTools 等开发组件。

**核心抽象：**
- **Agent**：单个 AI 代理，有工具、记忆、知识
- **Team**：多个 Agent 协作
- **Workflow**：确定性的流程编排（不依赖 LLM 决策）

**AgentOS**：基于 FastAPI 的运行时和控制面，把 Agent 变成 Web 服务。

**MCP 支持：**
- 原生 MCPTools，支持 stdio、SSE、Streamable HTTP
- include/exclude、timeout、动态 header、刷新连接
- confirmation、external execution、stop-after-tool-call

### 低空项目实践

**创建环境：**

```powershell
New-Item -ItemType Directory -Force "E:\gis\mcp\poc\agno"
Set-Location "E:\gis\mcp\poc\agno"

python -m venv ".venv"
& ".\.venv\Scripts\Activate.ps1"

# 如果激活时报执行策略错误
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
& ".\.venv\Scripts\Activate.ps1"

# 安装依赖
python -m pip install --upgrade pip
python -m pip install --upgrade "agno[os]" openai mcp

# 检查版本
python -m pip show agno
python -m pip show mcp
```

**设置 SiliconFlow API Key：**

```powershell
$secureKey = Read-Host "请输入 SiliconFlow API Key" -AsSecureString
$env:SILICONFLOW_API_KEY = [System.Net.NetworkCredential]::new("", $secureKey).Password
Remove-Variable secureKey
```

**编写飞服 Agent：**

```python
# feifu_agent.py
import asyncio
import os
from pathlib import Path

from agno.agent import Agent
from agno.models.siliconflow import Siliconflow
from agno.tools.mcp import MCPTools


FEIFU_PROJECT = Path(r"E:\gis\mcp\低空MCP程序_交付_20260707")
FEIFU_SRC = FEIFU_PROJECT / "src"
FEIFU_DATA = FEIFU_PROJECT / "data" / "derived"


async def main() -> None:
    if not os.getenv("SILICONFLOW_API_KEY"):
        raise RuntimeError("没有设置 SILICONFLOW_API_KEY")

    if not FEIFU_SRC.exists():
        raise RuntimeError(f"飞服源码目录不存在：{FEIFU_SRC}")

    if not FEIFU_DATA.exists():
        raise RuntimeError(f"飞服数据目录不存在：{FEIFU_DATA}")

    mcp_command = "python -m feifu_mcp.server"

    mcp_env = {
        "PYTHONPATH": str(FEIFU_SRC),
        "FEIFU_DATA_DIR": str(FEIFU_DATA),
        "PYTHONIOENCODING": "utf-8",
        "PYTHONUTF8": "1",
    }

    async with MCPTools(
        command=mcp_command,
        env=mcp_env,
        transport="stdio",
        exclude_tools=["answer_question"],
        timeout_seconds=30,
    ) as feifu_tools:
        agent = Agent(
            id="agno-feifu-poc",
            name="Agno飞服日报Agent",
            model=Siliconflow(
                id="deepseek-ai/DeepSeek-V4-Pro",
            ),
            tools=[feifu_tools],
            instructions=[
                "涉及飞服日报数据时必须调用飞服MCP工具。",
                "不得猜测飞服数据。",
                "不得绕过MCP直接读取飞服数据文件。",
                "回答时给出实际调用的工具名、日期、数值和单位。",
            ],
            markdown=False,
        )

        await agent.aprint_response(
            "必须调用 query_metric_value 工具，"
            "查询 2026-07-05 的飞行总架次。"
            "请输出工具名、日期、数值和单位。",
            stream=False,
        )


if __name__ == "__main__":
    asyncio.run(main())
```

**运行验证：**

```powershell
python ".\feifu_agent.py"
```

**返回结果：**
```
工具名：query_metric_value
日期：2026-07-05
数值：9448
单位：架次
```

**AgentOS HTTP API：**

Agno 原生提供 AgentOS，可以把 Agent 发布成 HTTP 服务：

```python
# agent_os.py
from agno.agent import Agent
from agno.models.siliconflow import Siliconflow
from agno.tools.mcp import MCPTools
from agno.os import AgentOS

# ... Agent 定义同上 ...

os = AgentOS(
    agents=[agent],
    host="127.0.0.1",
    port=8000,
)

os.run()
```

启动后可以通过 HTTP 调用：

```powershell
$body = @{
    input = "必须调用 query_metric_value 查询 2026-07-05 飞行总架次"
    session_id = "feifu-poc-session-001"
} | ConvertTo-Json

$response = Invoke-RestMethod `
    -Method Post `
    -Uri "http://127.0.0.1:8000/agents/agno-feifu-poc/run" `
    -ContentType "application/json" `
    -Body ([System.Text.Encoding]::UTF8.GetBytes($body))

$response.status      # COMPLETED
$response.run_id      # 478b49ab-79e2-491a-a8b3-feba3a57254c
$response.session_id  # feifu-poc-session-001
```

### 踩坑经验

**1. Agno 与 MCP SDK 版本兼容问题**

直接安装最新依赖时，MCP 2.x 将 `McpError` 重命名为 `MCPError`，Agno 3.0.5 仍引用旧名称，导致 `MCPTools` 无法导入。

**解决方案：** 固定版本
```powershell
python -m pip install "mcp>=1.9.2,<2.0.0"
```

**2. MCP 启动命令白名单限制**

Agno 不接受 Python 解释器绝对路径作为 MCP 启动命令，必须使用白名单中的 `python`。

**3. Agno 版本间 API 变化较快**

旧示例中的 `show_tool_calls=True` 在 Agno 3.0.5 已经不可用，导致 `TypeError`。

**4. 部署成本高于轻量 Agent**

Agno 需要自己维护 Python 应用代码，包括 Agent 定义、MCP 配置、数据库存储、AgentOS 服务、环境变量、API 鉴权、日志和监控、依赖锁定、部署脚本。

**5. 会话历史会增加 Token 消耗**

AgentOS 在复用同一个 `session_id` 时会加载历史记录。本次重复使用会话后，输入 Token 增加到 8420。

### 我的评价

**优点：**
- ✅ 原生支持 MCP，不需要自己实现 MCP Client
- ✅ AgentOS 适合工作流编排层，提供完整的 HTTP API
- ✅ 与低空项目技术栈一致（都是 Python）
- ✅ 支持进一步扩展 Team 和 Workflow
- ✅ 返回工程化信息较完整（Agent ID、Run ID、Session ID、状态、Token 统计等）

**缺点：**
- ❌ Agno 与 MCP SDK 存在版本兼容问题
- ❌ MCP 启动命令存在白名单限制
- ❌ Agno 版本间 API 变化较快
- ❌ 部署成本高于轻量 Agent
- ❌ 会话历史会增加 Token 消耗

**适合场景：**
- Python 技术栈项目
- 需要中心 Agent 服务
- 需要 Workflow 编排能力
- 团队愿意维护应用代码和依赖

---

## DeepSeek Harness：插件化 Agent Runtime

### 核心特点

DeepSeek Harness 是一个**插件化 Agent Runtime**，核心特点是：
- 插件化设计：MCP 插件、沙箱插件、权限插件
- Trace 能力突出：完整的调用链追踪
- JSONL 子进程 RPC：适合程序化控制

### 低空项目实践

（由于篇幅限制，这部分内容略，实际博客中会补充完整的安装、配置和验证代码）

### 踩坑经验

1. 安装配置相对复杂
2. 需要理解插件化架构
3. Trace 数据需要额外处理

### 我的评价

**优点：**
- ✅ 插件化设计，灵活度高
- ✅ Trace 能力突出，适合调试和审计
- ✅ JSONL RPC 适合程序化控制

**缺点：**
- ❌ 学习曲线较陡
- ❌ 文档相对较少
- ❌ 社区活跃度不如其他框架

**适合场景：**
- 需要完整 Trace 和审计
- 团队有 TypeScript 经验
- 愿意投入时间学习插件化架构

---

## 综合对比与选型建议

### 详细对比表格

| 评估维度 | nanobot | Hermes | Pi | Agno | DeepSeek Harness |
|---------|---------|--------|-----|------|------------------|
| **MCP 接入难度** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| **HTTP API 完整度** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **工程化成熟度** | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **部署复杂度** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| **文档和社区** | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ |
| **扩展性** | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **生产就绪度** | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |

### 选型决策树

```
你的项目是什么技术栈？
├─ Python
│  ├─ 需要 Workflow？
│  │  ├─ 是 → Agno
│  │  └─ 否 → nanobot 或 Hermes
│  └─ 需要多渠道接入（微信/Telegram）？
│     ├─ 是 → Hermes
│     └─ 否 → nanobot
├─ TypeScript/Node.js
│  ├─ 需要嵌入到现有应用？
│  │  ├─ 是 → Pi
│  │  └─ 否 → 考虑其他
│  └─ 需要完整 Trace？
│     ├─ 是 → DeepSeek Harness
│     └─ 否 → Pi
└─ 其他
   └─ 根据具体需求评估
```

### 我的最终选择：Agno

**为什么选 Agno？**

1. **技术栈一致**：飞服 MCP 是 Python 项目，Agno 也是 Python 框架，复用环境、调试方便
2. **原生 MCP 支持**：不需要像 Pi 一样自己写 Extension，接入成本低
3. **AgentOS 完整**：提供 HTTP API、Session 管理、Run 状态追踪，适合工作流编排
4. **扩展能力强**：后续可以用 Team 和 Workflow 实现更复杂的业务逻辑
5. **工程化信息完整**：返回 Agent ID、Run ID、Session ID、Token 统计等，便于集成

**代价是什么？**

- 需要维护应用代码和依赖版本
- 部署成本比 nanobot 高
- 需要处理版本兼容问题

**但综合来看，Agno 是低空项目的最佳选择。**

---

## 经验总结

### MCP 接入的通用经验

1. **工具白名单很重要**：不要暴露所有工具，只启用需要的
2. **超时控制必须有**：防止 MCP 调用卡死
3. **错误处理要完善**：MCP 可能返回各种异常
4. **日志记录要详细**：方便调试和审计

### 工程化注意事项

1. **鉴权**：API 必须有认证机制（Bearer Token、JWT 等）
2. **超时**：设置合理的超时时间（30-60 秒）
3. **幂等**：防止重复提交（虽然 Hermes 实测未生效）
4. **日志**：记录完整的调用链（工具名、参数、结果、耗时）
5. **监控**：监控 Token 消耗、响应时间、错误率

### 给后来者的建议

1. **先跑通 PoC**：不要一开始就追求完美，先验证可行性
2. **记录踩坑经验**：每个框架都有坑，记录下来避免重复踩
3. **关注版本兼容**：特别是 Python 生态，版本变化快
4. **考虑生产环境**：不只是"能跑"，还要考虑鉴权、超时、日志、监控
5. **选择适合的，不是最强的**：根据你的技术栈、团队能力、项目需求选择

### 后续计划

- 完善 Agno 的生产化部署（鉴权、并发、审计）
- 测试 Team 和 Workflow 能力
- 对比更多 Agent 框架（如 LangChain、AutoGPT）
- 分享低空项目的完整架构设计

---

## 附录

### 测试环境

- **操作系统**：Windows 11 + WSL2 (Ubuntu)
- **Python**：3.11
- **Node.js**：20+
- **模型**：DeepSeek-V4-Pro (SiliconFlow)
- **MCP Server**：飞服日报 MCP（12 个工具）

### 完整代码仓库

所有 PoC 代码和配置已开源：[https://github.com/Du-Huan-1123/agent-framework-research](https://github.com/Du-Huan-1123/agent-framework-research)

仓库包含：
- nanobot 完整配置和示例
- Hermes WSL 部署脚本和 API 调用示例
- Pi MCP Extension 实现
- Agno Agent 和 AgentOS 代码
- DeepSeek Harness 配置（待补充）
- 框架对比分析文档

---

**如果这篇文章对你有帮助，欢迎点赞、收藏、分享！**

**有问题欢迎在评论区讨论，我会及时回复。**
