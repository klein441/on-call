<div align="center">

# 🚨 SuperBizAgent · On-Call

**企业级智能对话 & AIOps 运维助手**

> 基于 LangGraph Plan-Execute-Replan 模式，融合 RAG 知识库检索与 MCP 工具调用，实现全自动故障诊断与根因分析。

[![Python](https://img.shields.io/badge/Python-3.10+-3776AB?style=flat-square&logo=python&logoColor=white)](https://www.python.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.109+-009688?style=flat-square&logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com/)
[![LangChain](https://img.shields.io/badge/LangChain-latest-1C3C3C?style=flat-square&logo=langchain&logoColor=white)](https://www.langchain.com/)
[![LangGraph](https://img.shields.io/badge/LangGraph-latest-FF6B35?style=flat-square)](https://langchain-ai.github.io/langgraph/)
[![Milvus](https://img.shields.io/badge/Milvus-Vector_DB-00A1EA?style=flat-square)](https://milvus.io/)
[![License](https://img.shields.io/badge/License-MIT-green?style=flat-square)](LICENSE)

</div>

---

## 目录

- [核心特性](#-核心特性)
- [技术架构](#-技术架构)
- [快速开始](#-快速开始)
- [API 接口](#-api-接口)
- [项目结构](#-项目结构)
- [配置说明](#️-配置说明)
- [AIOps 诊断流程](#-aiops-诊断流程)
- [开发指南](#-开发指南)
- [常见问题](#-常见问题)
- [参考资源](#-参考资源)

---

## ✨ 核心特性

| 特性 | 说明 |
|------|------|
| 🤖 **智能对话** | LangChain 多轮对话，支持流式输出（SSE） |
| 📚 **RAG 问答** | 向量检索增强，文档上传后自动建立索引、实时更新知识库 |
| 🔧 **AIOps 诊断** | Plan → Execute → Replan 三段式自动故障诊断与根因分析 |
| 🌐 **Web 界面** | 现代化 UI，支持普通问答 / 流式对话 / 智能运维三种模式 |
| 🔌 **MCP 集成** | 通过 Model Context Protocol 接入日志查询与监控数据工具 |

---

## 🏗 技术架构

```
┌─────────────────────────────────────────────────────────┐
│                     Web UI / API Client                  │
└───────────────────────┬─────────────────────────────────┘
                        │ HTTP / SSE
┌───────────────────────▼─────────────────────────────────┐
│                   FastAPI  (port 9900)                   │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────┐  │
│  │  /api/chat  │  │ /api/aiops   │  │  /api/upload   │  │
│  └──────┬──────┘  └──────┬───────┘  └───────┬────────┘  │
│         │                │                  │            │
│  ┌──────▼──────┐  ┌──────▼───────┐  ┌───────▼────────┐  │
│  │ RAG Agent   │  │ AIOps Agent  │  │  Doc Splitter  │  │
│  │ (LangGraph) │  │ Plan-Execute │  │  + Embedding   │  │
│  └──────┬──────┘  └──────┬───────┘  └───────┬────────┘  │
└─────────┼────────────────┼──────────────────┼───────────┘
          │                │                  │
   ┌──────▼──────┐  ┌──────▼───────┐  ┌───────▼────────┐
   │   Milvus    │  │  MCP Servers │  │   DashScope    │
   │ Vector DB   │  │ CLS/Monitor  │  │  (通义千问)     │
   └─────────────┘  └──────────────┘  └────────────────┘
```

**技术栈一览**

| 层级 | 技术 |
|------|------|
| 框架 | FastAPI + LangChain + LangGraph |
| LLM | 阿里云 DashScope（通义千问） |
| 向量库 | Milvus |
| 工具协议 | MCP (Model Context Protocol) |
| 日志 | Loguru（按天轮转） |
| 包管理 | uv |

---

## 🚀 快速开始

### 环境要求

- Python **3.10+**
- Docker Desktop（运行 Milvus）
- 阿里云 DashScope API Key → [获取地址](https://dashscope.aliyun.com/)

### Linux / macOS

```bash
# 1. 克隆项目
git clone https://github.com/klein441/on-call.git
cd on-call

# 2. 安装依赖（推荐 uv）
pip install uv
uv venv && source .venv/bin/activate
uv pip install -e .

# 3. 填写配置
cp .env.example .env   # 如有示例文件；或直接编辑 .env
vim .env               # 填入 DASHSCOPE_API_KEY

# 4. 一键初始化（启动 Docker + 服务 + 上传文档）
make init

# 5. 启动
make start
```

### Windows（推荐使用一键脚本）

```powershell
# 启动所有服务
.\start-windows.bat

# 停止所有服务
.\stop-windows.bat
```

<details>
<summary>Windows 手动启动步骤（点击展开）</summary>

```powershell
# 1. 克隆并进入项目
git clone https://github.com/klein441/on-call.git
cd on-call

# 2. 创建虚拟环境并安装依赖
pip install uv
uv venv
.venv\Scripts\activate
uv pip install -e .

# 3. 填写配置
notepad .env   # 填入 DASHSCOPE_API_KEY

# 4. 启动 Milvus（需先启动 Docker Desktop）
docker compose -f vector-database.yml up -d
timeout /t 10

# 5. 启动 MCP 服务（各开一个新窗口）
python mcp_servers/cls_server.py
python mcp_servers/monitor_server.py

# 6. 启动主服务
python -m uvicorn app.main:app --host 0.0.0.0 --port 9900

# 7. 上传知识库文档
timeout /t 5
python -c "import requests,os,time; [requests.post('http://localhost:9900/api/upload',files={'file':open(f'aiops-docs/{f}','rb')}) or time.sleep(1) for f in os.listdir('aiops-docs') if f.endswith('.md')]"
```

</details>

### 访问服务

| 服务 | 地址 |
|------|------|
| Web 界面 | http://localhost:9900 |
| API 文档（Swagger） | http://localhost:9900/docs |

---

## 📡 API 接口

| 功能 | 方法 | 路径 | 说明 |
|------|------|------|------|
| 普通对话 | `POST` | `/api/chat` | 一次性返回 |
| 流式对话 | `POST` | `/api/chat_stream` | SSE 流式输出 |
| AIOps 诊断 | `POST` | `/api/aiops` | 自动故障诊断（流式） |
| 文件上传 | `POST` | `/api/upload` | 上传并索引文档 |
| 健康检查 | `GET` | `/api/health` | 服务状态检查 |

**示例请求**

```bash
# 普通对话
curl -X POST http://localhost:9900/api/chat \
  -H "Content-Type: application/json" \
  -d '{"Id":"session-123","Question":"你好"}'

# 流式对话
curl -X POST http://localhost:9900/api/chat_stream \
  -H "Content-Type: application/json" \
  -d '{"Id":"session-123","Question":"你好"}' --no-buffer

# AIOps 诊断
curl -X POST http://localhost:9900/api/aiops \
  -H "Content-Type: application/json" \
  -d '{"session_id":"session-123"}' --no-buffer
```

---

## 📁 项目结构

```
on-call/
├── app/
│   ├── main.py                      # FastAPI 入口
│   ├── config.py                    # 配置管理（环境变量 + MCP）
│   ├── api/                         # 路由层
│   │   ├── chat.py                  # 对话接口
│   │   ├── aiops.py                 # AIOps 接口
│   │   ├── file.py                  # 文件上传
│   │   └── health.py                # 健康检查
│   ├── services/                    # 业务服务层
│   │   ├── rag_agent_service.py     # RAG Agent（LangGraph）
│   │   ├── aiops_service.py         # AIOps 服务
│   │   ├── vector_store_manager.py  # 向量存储管理
│   │   ├── vector_embedding_service.py
│   │   ├── vector_index_service.py
│   │   ├── vector_search_service.py
│   │   └── document_splitter_service.py
│   ├── agent/
│   │   ├── mcp_client.py            # MCP 客户端
│   │   └── aiops/                   # AIOps 核心
│   │       ├── planner.py           # 计划制定
│   │       ├── executor.py          # 步骤执行
│   │       ├── replanner.py         # 动态重规划
│   │       └── state.py             # 状态定义
│   ├── models/                      # 数据模型
│   ├── tools/                       # Agent 工具集
│   ├── core/                        # LLM 工厂 + Milvus 客户端
│   └── utils/                       # 日志工具（Loguru）
├── mcp_servers/
│   ├── cls_server.py                # CLS 日志查询服务
│   └── monitor_server.py            # 监控数据服务
├── aiops-docs/                      # 运维知识库（Markdown）
├── static/                          # Web 前端
├── start-windows.bat                # Windows 一键启动
├── stop-windows.bat                 # Windows 一键停止
├── vector-database.yml              # Milvus Docker Compose
├── Makefile                         # Linux/macOS 快捷命令
└── pyproject.toml                   # 项目依赖配置
```

---

## ⚙️ 配置说明

复制并编辑 `.env` 文件：

```bash
# ── LLM（必填）─────────────────────────────────────────────
# API Key 管理：https://bailian.console.aliyun.com/#/api-key
DASHSCOPE_API_KEY=your-api-key-here
DASHSCOPE_API_BASE=https://dashscope.aliyuncs.com/compatible-mode/v1
DASHSCOPE_MODEL=qwen-max

# ── Milvus ────────────────────────────────────────────────
MILVUS_HOST=localhost
MILVUS_PORT=19530

# ── RAG ───────────────────────────────────────────────────
RAG_TOP_K=3
CHUNK_MAX_SIZE=800
CHUNK_OVERLAP=100
```

---

## 🎯 AIOps 诊断流程

```
告警触发
    │
    ▼
┌─────────┐     生成 4-6 个诊断步骤
│ Planner │ ──────────────────────────►
└─────────┘
    │
    ▼
┌──────────┐    调用 MCP 工具
│ Executor │ ── 日志查询 / 监控数据 ──►
└──────────┘
    │
    ▼
┌────────────┐   继续 / 调整计划 / 生成报告
│ Replanner  │ ──────────────────────────►
└────────────┘
    │
    ▼
诊断报告（根因分析 + 运维建议）
```

**AIOps 特性清单**

- ✅ 自动制定多步诊断计划
- ✅ 智能工具调用（日志 + 监控）
- ✅ 动态重规划（根据执行结果调整）
- ✅ 全程流式输出，实时可见
- ✅ 结构化诊断报告

---

## 📝 开发指南

```bash
# 项目管理（Linux/macOS）
make init        # 一键初始化
make start       # 启动所有服务
make stop        # 停止所有服务
make restart     # 重启服务

# 依赖管理
make install-dev # 安装开发依赖
make sync        # 同步依赖

# 代码质量
make format      # 格式化代码
make lint        # 代码检查
```

---

## 🐛 常见问题

<details>
<summary><strong>Windows：<code>make</code> 命令不可用</strong></summary>

使用批处理脚本代替：

```powershell
.\start-windows.bat   # 启动
.\stop-windows.bat    # 停止
```

</details>

<details>
<summary><strong>Windows：PowerShell 执行策略限制</strong></summary>

```powershell
# 临时允许（管理员权限）
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope Process
```

</details>

<details>
<summary><strong>端口被占用</strong></summary>

```powershell
# Windows
netstat -ano | findstr :9900
taskkill /F /PID <PID>
```

```bash
# Linux/macOS
lsof -i :9900
kill -9 <PID>
```

</details>

<details>
<summary><strong>Milvus 连接失败</strong></summary>

```bash
# 确认 Docker Desktop 已启动，然后
docker compose -f vector-database.yml restart
docker ps | grep milvus
```

</details>

<details>
<summary><strong>API Key 错误</strong></summary>

```bash
# Linux/macOS
grep DASHSCOPE_API_KEY .env

# Windows
type .env | findstr DASHSCOPE_API_KEY
```

</details>

<details>
<summary><strong>查看服务日志</strong></summary>

```powershell
# Windows（最新日志文件末 50 行）
Get-ChildItem logs\*.log | Sort-Object LastWriteTime -Descending |
  Select-Object -First 1 | Get-Content -Tail 50
```

```bash
# Linux/macOS
tail -f logs/app_$(date +%Y-%m-%d).log
```

</details>

---

## 📚 参考资源

- [FastAPI 文档](https://fastapi.tiangolo.com/)
- [LangChain 文档](https://python.langchain.com/)
- [LangGraph Plan-Execute 教程](https://langchain-ai.github.io/langgraph/tutorials/plan-and-execute/)
- [阿里云 DashScope](https://dashscope.aliyun.com/)
- [MCP 协议](https://modelcontextprotocol.io/)
- [Milvus 文档](https://milvus.io/docs)

---

## 📄 许可证

MIT License · author: chief
