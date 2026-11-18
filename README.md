# NexusAgent —— 从零手写 Go 智能体平台（教学项目源码）

<div align="center">

**这是我《Go 智能体开发实战》系列视频课程的配套项目源码，跟着视频 + 源码即可完整复现一个生产级 Agent 平台。**

[![Bilibili](https://img.shields.io/badge/B站-课程主页-00A1D6?logo=bilibili&logoColor=white)](https://space.bilibili.com/481791559)
[![Go](https://img.shields.io/badge/Go-1.26+-00ADD8?logo=go&logoColor=white)](https://go.dev)
[![React](https://img.shields.io/badge/React-18-61DAFB?logo=react&logoColor=black)](https://react.dev)
[![License](https://img.shields.io/badge/License-MIT-green.svg)]()

**📺 关注我，不错过每一集更新：**

[![Bilibili](https://img.shields.io/badge/🎬_B站_Space-481791559-fb7299?style=for-the-badge&logo=bilibili&logoColor=white)](https://space.bilibili.com/481791559)

| 简历优化、项目定制、1v1全程陪跑等 |
|:---:|
| <img src="docs/images/vx.png" width="200" alt="微信二维码"/> |
| 扫码加微信 |

</div>

---

NexusAgent 是一个**通用智能体（Agent）平台**，覆盖了 Agent 开发的主线能力：

- **ReAct 智能体**：Function Calling / 推理-行动循环 / 工具调用，含思考过程与决策链可视化
- **RAG 知识库**：文档上传 → ETL 解析 → 分块 → 向量化 → 混合检索（稠密 + 稀疏 + RRF 融合），带检索测试台
- **多智能体协作**：Host/Expert 模式、A2A 远程 Agent 委派
- **工作流引擎**：YAML DSL 编排、条件分支、HITL 人工审批中断/恢复
- **MCP 集成**：作为 MCP Client 接入外部工具生态，支持调试与健康检查
- **长期记忆**：对话自动抽取记忆，跨会话沉淀
- **生产基建**：JWT 双 Token 认证、密钥 AES-GCM 加密、OpenTelemetry 全链路追踪、Prometheus 指标、Redis Streams 异步任务

项目采用**双轨教学**：`agent/naive/` 下有每项能力的 ~500 行手写极简版（看懂原理），生产版基于字节 [eino](https://github.com/cloudwego/eino) 框架工程化实现（学会落地）。
(变更，途中决定不将 eino 和手撕放在一起，后续会单独出一个纯 eino 项目，用于教学。)

## 技术栈

| 层 | 技术 |
|---|---|
| 后端 | Go 1.26+ · Hertz（HTTP）· GORM · PostgreSQL 16 · Redis 7（Streams）· Milvus 2.5（向量）· MinIO（对象存储）· golang-migrate |
| AI/Agent | DeepSeek / Ollama（OpenAI 兼容协议）· eino 框架 · mcp-go · Docker 沙箱（代码执行）· SearXNG（联网搜索） |
| 前端 | React 18.3 · TypeScript（strict）· TailwindCSS 4 · GSAP 3.13 · Zustand 5 · React Router 7 · Vite 7 |
| 可观测 | OpenTelemetry（Jaeger）· Prometheus · Grafana · zap 结构化日志 |

## 快速开始

### 环境要求

- **Go ≥ 1.26**（`go version` 检查）
- **Node.js 20.19+ 或 22.12+**（前端开发与构建）
- **Docker & Docker Compose**（基础设施一键起）

Tailwind CSS 4 的样式最低支持 Safari 16.4、Chrome 111、Firefox 128；需要兼容更旧浏览器时应保留 Tailwind CSS 3.4。详见 [Tailwind CSS 浏览器兼容说明](https://tailwindcss.com/docs/compatibility)。

### ① 启动基础设施（PostgreSQL / Redis / MinIO）

```bash
cd backend
make compose-core          # 起 PostgreSQL/Redis/MinIO/Milvus；make compose-up 额外启动 Ollama/SearXNG/观测栈
```

### ② 数据库迁移 + 初始化管理员

```bash
NX_SEED_ADMIN_PASSWORD='设置一个至少 16 位的管理员密码' make migrate
```

首次迁移会用该密码创建 `admin`。未设置此变量时会生成随机的一次性密码并显示在迁移命令输出中，请安全保存。
升级旧版本时，如果管理员仍使用默认密码 `admin123`，迁移会将其轮换为这里配置的密码；未设置变量时则生成随机密码并显示一次。

### ③ 启动后端 API 服务

```bash
make run                   # 监听 http://localhost:8080，同时启动 A2A 端点 :8082
```

> 如需异步任务（文档 ETL 解析入库），另开一个终端执行 `make run-worker`。

### ④ 启动前端控制台

```bash
cd ../frontend
npm install
npm run dev                # http://localhost:5173，/api 已代理到 localhost:8080
```

### ⑤ 配置模型，开始对话

1. 打开 http://localhost:5173，用迁移时配置或生成的管理员凭据登录
2. 进入 **管理后台 → 模型配置**，给预置的 DeepSeek 供应商填入你的 `API Key`（或切换到 Ollama 本地模型，`config.yaml` 中 `llm.local: true`）
3. 回到 **对话页**，发送消息，即可看到流式输出、思考过程、工具调用决策链

> 完整体验联网搜索和本地模型：`make compose-up` 启动全部基础设施（包括 SearXNG / Ollama / 观测栈）；RAG 所需的 Milvus 已包含在 `compose-core` 中。

## 项目结构

```
agent-golang/
├── README.md                  # 本文件
├── docs/                      # 教学文档（PRD/TD/教学规划/讲义 PDF/学习资料）
├── backend/                   # Go 后端
│   ├── cmd/
│   │   ├── server/            # API 服务入口（依赖装配总装车间）
│   │   ├── worker/            # 异步任务 Worker（文档 ETL / 评估）
│   │   ├── cli/               # 终端对话 Agent（跳过前端直接体验）
│   │   └── migrate/           # 数据库迁移工具（make migrate）
│   ├── configs/config.yaml    # 默认配置（NX_ 环境变量可覆盖）
│   ├── deployments/           # docker-compose / Dockerfile / Prometheus / Grafana
│   ├── migrations/            # SQL 版本化迁移（golang-migrate）
│   └── internal/
│       ├── api/               # HTTP 路由与 Handler（REST + SSE）
│       ├── service/           # 业务编排层
│       ├── domain/            # 领域模型
│       ├── repo/              # 数据访问层（GORM）
│       ├── agent/             # ReAct 循环（naive 手写版 + eino 生产版）
│       ├── rag/               # 检索增强引擎（分块/混合检索/RRF）
│       ├── workflow/          # 工作流引擎（DSL/HITL）
│       ├── memory/            # 长期记忆抽取
│       ├── tool/              # 工具系统（内置/HTTP/沙箱/db_query）
│       ├── llm/               # LLM 网关（多供应商/熔断/重试/流式）
│       ├── mcp/ a2a/          # MCP Client / A2A 远程协作
│       ├── infra/             # PG/Redis/Milvus/MinIO 连接初始化
│       ├── security/          # JWT / AES-GCM 加密
│       ├── observability/     # OTel 追踪 / Prometheus 指标 / 日志
│       └── worker/            # Redis Streams 消费者
└── frontend/                  # React 控制台
    └── src/
        ├── api/               # 类型化接口封装（与后端 Handler 一一对应）
        ├── pages/             # 登录/对话/agents/kbs/workflows/admin
        ├── stores/            # zustand：auth / chat（SSE 状态机）/ ui
        ├── lib/               # request（401 刷新重放）/ sse（POST SSE 解析）/ gsap
        ├── components/        # 布局 + UI 原语 + 决策链时间线/计划卡片
        └── types/             # 领域类型
```

## 常用端口速查

| 服务 | 地址 | 说明 |
|---|---|---|
| 前端控制台 | http://localhost:5173 | 用户名 `admin`；密码由迁移时设置或生成 |
| 后端 API | http://localhost:8080 | REST + SSE |
| A2A 端点 | http://localhost:8082 | `/.well-known/agent.json` |
| MinIO 控制台 | http://localhost:9001 | `minioadmin / minioadmin` |
| Jaeger UI | http://localhost:16686 | 链路追踪（需 obs profile） |
| Grafana | http://localhost:3004 | 监控面板（`admin / admin`） |

## 常见问题

- **`make migrate` 连接数据库失败**：先确认 `make compose-core` 已启动且 PG 健康检查通过，再重试。
- **生产部署**：设置 `NX_ENV=prod`，并提供不同且足够随机的 `NX_JWT__SECRET` 和 `NX_SECRET_KEY`（均至少 32 个字符）；服务会拒绝使用默认密钥或相同密钥启动。
- **对话报模型错误**：未配置 API Key。到「管理后台 → 模型配置」填写，或使用 Ollama 本地模型。
- **端口冲突**：Redis 使用 **6380**（避开本机 6379），Grafana 使用 3004，均可在 `config.yaml` / compose 中调整。
- **想体验 RAG 但不想起 Milvus**：`config.yaml` 中 `milvus.mode: lite` 可用本地嵌入式模式（仅开发用）。

## 学习路线

配合 B 站视频课程食用，按里程碑推进：

**M1 工程骨架 → M2 ReAct 智能体 → M3 RAG 知识库 → M4 MCP/A2A → M5 工作流 → M6 可观测与上线**

每集均遵循「回顾目标 → Demo → 代码走读 → 验证 → 理论 → 总结」结构。

---

<div align="center">

**如果这个项目对你有帮助，请给个 Star ⭐ 支持一下！**

[![Bilibili](https://img.shields.io/badge/🎬_B站_Space-481791559-fb7299?style=for-the-badge&logo=bilibili&logoColor=white)](https://space.bilibili.com/481791559)

<img src="docs/images/vx.png" width="160" alt="微信"/>

</div>
