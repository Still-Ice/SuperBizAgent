# SuperBizAgent

基于 Spring Boot 与 AI Agent 的智能问答与运维助手，面向 OnCall / AIOps 场景。

## 项目简介

本项目提供两套能力：

| 模块 | 作用 |
|------|------|
| **RAG 智能问答** | 文档入库向量库，结合检索与大模型回答问题，支持多轮对话与流式输出 |
| **AIOps 智能运维** | Planner → Executor → Replanner 多 Agent 协作，完成告警分析、日志查询、诊断与报告 |

## 功能特性

- 向量检索 + 多轮会话 + SSE 流式回答
- 告警 / 日志 / 文档等工具调用，自动编排诊断流程
- 会话上下文维护与清理
- 内置 Web 测试页与 REST API
- 密钥通过本地 `.env` 注入，不进入仓库

## 技术栈

| 技术 | 版本 | 说明 |
|------|------|------|
| Java | 17 | 运行语言 |
| Spring Boot | 3.2.0 | 应用框架 |
| Spring AI | 1.1.0 | AI 集成 |
| Spring AI Alibaba | 1.1.0.0-RC2 | DashScope / Agent |
| Milvus | 2.5.x（Compose 镜像） | 向量数据库 |
| Docker Compose | — | 本地依赖（Milvus / MinIO / 可选 CLS MCP） |

## 目录结构

```
SuperBizAgent/
├── src/main/java/org/example/
│   ├── controller/          # HTTP 接口（对话、上传、健康检查）
│   ├── service/             # Chat / RAG / AIOps / 向量服务
│   ├── agent/tool/          # Agent 工具（文档、告警、日志、时间）
│   └── config/              # Milvus、DashScope、上传等配置
├── src/main/resources/
│   ├── static/              # Web 测试界面
│   └── application.yml      # 应用配置（密钥走环境变量）
├── aiops-docs/              # 运维知识库文档
├── vector-database.yml              # Milvus 基础依赖
├── vector-database+cls-mcp.yml      # Milvus + 腾讯云 CLS MCP
├── .env.example / .env-副本         # 密钥模板（可提交）
├── Makefile                 # 一键初始化
└── pom.xml
```

## 环境与密钥

**不要**把真实 API Key、云密钥提交到 Git。仓库里只保留模板：

1. 复制模板为本地文件：

```powershell
Copy-Item .env.example .env
# 或：Copy-Item .env-副本 .env
```

2. 编辑 `.env`，填写真实值，例如：

```env
DASHSCOPE_API_KEY=sk-xxxxxxxx
TENCENTCLOUD_SECRET_ID=AKIDxxxxxxxx   # 仅启用 CLS MCP 时需要
TENCENTCLOUD_SECRET_KEY=xxxxxxxx
```

3. 使用方式：
   - **Docker Compose**：项目根目录存在 `.env` 时会自动读取
   - **本机进程（PowerShell）**：`$env:DASHSCOPE_API_KEY="sk-xxxx"`
   - **本机进程（bash）**：`export DASHSCOPE_API_KEY=sk-xxxx`
   - **团队传递**：用密码管理器 / CI Secrets，不要通过聊天或 Git 明文传递

`application.yml` 中密钥均通过环境变量读取，例如 `${DASHSCOPE_API_KEY}`。

## 快速开始

### 前置条件

- JDK 17+
- Maven 3.8+
- Docker / Docker Compose
- 有效的阿里云 DashScope API Key

### 方式一：Makefile 一键初始化

```bash
# 先配置好 .env 或导出 DASHSCOPE_API_KEY
make init
```

会依次：启动向量库 → 启动 Spring Boot → 上传 `aiops-docs` 文档。

### 方式二：手动启动

```bash
# 1. 启动向量数据库（二选一）
docker compose -f vector-database.yml up -d
# 若需要腾讯云 CLS MCP：
# docker compose -f vector-database+cls-mcp.yml up -d

# 2. 启动应用
mvn clean install
mvn spring-boot:run
```

### 访问入口

| 服务 | 地址 |
|------|------|
| Web / API | http://localhost:9900 |
| Attu（Milvus UI） | http://localhost:8000 |
| CLS MCP SSE（可选） | http://localhost:3000 |

## 主要接口

### 流式对话（推荐）

```bash
POST /api/chat_stream
Content-Type: application/json

{
  "Id": "session-123",
  "Question": "什么是向量数据库？"
}
```

SSE 输出，支持工具调用与多轮对话。

### 普通对话

```bash
POST /api/chat
Content-Type: application/json

{
  "Id": "session-123",
  "Question": "什么是向量数据库？"
}
```

### AIOps 运维诊断

```bash
POST /api/ai_ops
```

SSE 流式输出诊断过程与报告。

### 其它

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/chat/clear` | 清空会话历史 |
| GET | `/api/chat/session/{sessionId}` | 查询会话信息 |
| POST | `/api/upload` | 上传文档并向量化 |
| GET | `/milvus/health` | Milvus 健康检查 |

### 调用示例

```bash
# 上传文档
curl -X POST http://localhost:9900/api/upload \
  -F "file=@document.txt"

# 问答
curl -X POST http://localhost:9900/api/chat \
  -H "Content-Type: application/json" \
  -d "{\"Id\":\"test\",\"Question\":\"什么是向量数据库？\"}"

# 健康检查
curl http://localhost:9900/milvus/health
```

## 核心配置说明

默认监听 `9900`。常用项见 `src/main/resources/application.yml`：

| 配置项 | 含义 | 默认/示例 |
|--------|------|-----------|
| `server.port` | 服务端口 | `9900` |
| `milvus.host` / `port` | 向量库地址 | `localhost:19530` |
| `spring.ai.dashscope.api-key` | 大模型密钥 | `${DASHSCOPE_API_KEY}` |
| `rag.top-k` | 检索条数 | `3` |
| `rag.model` | 对话模型 | `qwen3-max` |
| `document.chunk.max-size` | 分片大小 | `800` |
| `document.chunk.overlap` | 分片重叠 | `100` |

## 常用 Make 命令

| 命令 | 说明 |
|------|------|
| `make init` | 一键初始化（Docker + 服务 + 文档入库） |
| `make up` / `make down` | 启停向量库 Compose |
| `make start` / `make stop` | 启停 Spring Boot |
| `make upload` | 上传 `aiops-docs` |
| `make status` | 查看容器状态 |


