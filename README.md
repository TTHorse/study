# 前端转全栈 + Agent 开发 学习路线

> 目标受众：有前端开发经验，希望补齐全栈能力并最终进入 Agent 开发领域的工程师。

---

## 目录

- [Phase 1：网络与传输层基础](#phase-1网络与传输层基础)
- [Phase 2：后端入门与前后端打通](#phase-2后端入门与前后端打通)
- [Phase 3：工程化与项目部署](#phase-3工程化与项目部署)
- [Phase 4：Agent 开发基础](#phase-4agent-开发基础)
- [Phase 5：实战 — 从零交付一个 Agent 应用](#phase-5实战--从零交付一个-agent-应用)
- [持续学习的资源索引](#持续学习的资源索引)

---

## Phase 1：网络与传输层基础

作为前端，你每天都在和 HTTP 打交道，但大概率只停留在"调接口"层面。
这一阶段的目标是**理解数据在网络上到底是怎么流动的**。

### 1.1 HTTP 协议深入

| 主题 | 需要掌握到什么程度 | 建议时间 |
|------|-------------------|---------|
| HTTP 请求/响应结构 | 能徒手写出一个完整的 HTTP 报文 | 1 天 |
| HTTP 方法 (GET/POST/PUT/PATCH/DELETE) | 理解幂等性、安全性，知道什么场景用什么方法 | 0.5 天 |
| HTTP 状态码 | 2xx/3xx/4xx/5xx 分类含义，常见状态码的适用场景 | 0.5 天 |
| HTTP Headers | Content-Type、Authorization、CORS 相关头、缓存头 | 1 天 |
| HTTP/1.1 vs HTTP/2 vs HTTP/3 | 多路复用、头部压缩、QUIC 解决了什么问题 | 1 天 |
| HTTPS 与 TLS 握手 | 对称/非对称加密、证书链、握手过程 | 1.5 天 |

**动手实践：** 用 `curl -v` 访问一个 HTTPS 网站，逐行读懂输出；用 Chrome DevTools Network 面板分析一个完整请求的时序。

### 1.2 TCP/IP 基础

| 主题 | 需要掌握到什么程度 |
|------|-------------------|
| TCP 三次握手 / 四次挥手 | 能画出时序图，解释每个状态转换 |
| TCP 与 UDP 的区别 | 什么时候用 TCP，什么时候用 UDP |
| IP 地址、子网掩码、端口 | 理解 `127.0.0.1:3000` 每个部分的含义 |
| NAT 与内网穿透 | 知道为什么本地起的服务外网访问不到 |

### 1.3 DNS 与域名

| 主题 | 需要掌握到什么程度 |
|------|-------------------|
| DNS 解析流程 | 浏览器 → 本地缓存 → DNS 服务器 → 根域名 → 权威 DNS |
| A 记录 / CNAME / NS 记录 | 知道买域名后怎么配，各记录类型的区别 |
| CDN 原理 | DNS 调度 + 边缘节点缓存 |

---

## Phase 2：后端入门与前后端打通

选一门后端语言深入学习。**推荐 Node.js（对你前端背景最友好）**，
辅以 Python（Agent 生态最成熟），后续 Agent 阶段主要用 Python。

### 2.1 Node.js 后端 (推荐主力)

| 主题 | 做什么 | 建议时间 |
|------|--------|---------|
| Node.js 运行时 | 事件循环、模块系统、异步模型（和浏览器 JS 的异同） | 2 天 |
| Express.js | 从零写 RESTful API：路由、中间件、错误处理 | 3 天 |
| 数据库入门 | SQLite → PostgreSQL，学会写 CRUD、JOIN、索引 | 5 天 |
| ORM (Prisma / Drizzle) | 用 ORM 替代手写 SQL，理解 migration | 2 天 |
| 认证与授权 | JWT 的签发与验证、session vs token、OAuth2.0 流程 | 3 天 |
| 文件上传 | multer、presigned URL、对象存储（S3/OSS） | 1 天 |

**阶段检验：** 写一个带用户注册/登录、支持 CRUD 的 Todo API，用 Postman 测通。

### 2.2 Python 入门 (Agent 预备)

| 主题 | 做什么 |
|------|--------|
| Python 语法速通 | 类型系统、函数、类、async/await、pip/venv |
| FastAPI | 等同于 Express 的角色，写一套 RESTful API |
| 类型注解与 Pydantic | Agent 框架重度依赖，必须掌握 |

### 2.3 前后端联调关键概念

| 概念 | 一句话解释 |
|------|-----------|
| **同源策略与 CORS** | 浏览器禁止跨域请求，后端需要设置 `Access-Control-Allow-Origin` |
| **跨域方案** | CORS（标准方案）、Proxy（开发期用 vite/webpack proxy）、JSONP（历史遗留） |
| **Cookie 跨域** | `withCredentials` + `SameSite` + `Secure`，理解三者关系 |
| **API 设计规范** | RESTful 命名、版本管理、分页、错误码统一格式 |
| **接口文档** | 学会写 OpenAPI/Swagger，FastAPI 可自动生成 |

---

## Phase 3：工程化与项目部署

这是前端同学最大的一块知识盲区。目标：**能把代码变成互联网上可访问的服务**。

### 3.1 部署全景图

先建立整体认知——代码从本地到线上经历了什么：

```
本地开发 → git push → CI/CD 构建 → 产物 → 服务器/容器 → 域名解析 → 用户访问
```

### 3.2 部署方式递进

| 层级 | 方案 | 适合场景 | 先学这个 |
|------|------|---------|---------|
| PaaS 平台部署 | Vercel / Railway / Render | 个人项目、快速验证 | ✅ 从这里开始 |
| 云服务器部署 | 阿里云 ECS / AWS EC2 + Nginx | 有定制需求的项目 | 第二步 |
| 容器化部署 | Docker + Docker Compose | 多服务编排、团队协作 | 第三步 |
| 编排 | Kubernetes (k8s) | 大规模微服务 | 进阶了解即可 |

### 3.3 部署知识拆解

**PaaS 平台阶段 (1-2 天上手)：**
- 在 Vercel 上部署一个前端项目 + 一个 Node API，理解环境变量、构建命令、部署日志
- 看到你的应用有了一个公网 URL（如 `xxx.vercel.app`）

**云服务器阶段 (3-5 天)：**
- 买一台最低配云服务器（阿里云/腾讯云 新用户几十块一年）
- SSH 登录、Linux 基础命令（`ls`/`cd`/`ps`/`top`/`systemctl`/`journalctl`）
- 在服务器上装 Node.js，用 `pm2` 跑你的后端服务
- 配 Nginx 反向代理 + 静态文件服务
- 申请免费 SSL 证书（Let's Encrypt / acme.sh），配 HTTPS

**关键概念：**

| 概念 | 一句话解释 |
|------|-----------|
| **Nginx 反向代理** | Nginx 接收用户请求，转发给后端服务（如 `localhost:3000`），用户不直接接触后端 |
| **SSL/TLS 证书** | 给域名加 HTTPS 小锁；Let's Encrypt 免费，acme.sh 自动续期 |
| **DNS A 记录** | 把域名指向服务器 IP |
| **环境变量** | 敏感配置（数据库密码、API Key）不写代码里，通过环境变量注入 |
| **pm2 / systemd** | 进程守护——服务挂了自动重启，开机自启 |

**Docker 阶段 (3-5 天)：**
- 理解镜像（Image）vs 容器（Container）——镜像是"安装包"，容器是"运行实例"
- 写 Dockerfile 把你的项目打包成镜像
- 用 Docker Compose 编排多服务（前端 + 后端 + 数据库）
- 理解端口映射、数据卷（volume）、网络模式

### 3.4 CI/CD

| 主题 | 做什么 |
|------|--------|
| GitHub Actions | 配置 `.github/workflows/deploy.yml`，push 代码自动构建部署 |
| 典型流程 | lint → test → build → deploy |
| 环境区分 | dev / staging / production 三套环境变量和部署策略 |

---

## Phase 4：Agent 开发基础

进入 Agent 开发前，先建立起对 LLM 和 Agent 的概念框架。

> **Agent 不是魔法**——本质上是 LLM + 工具调用 + 循环控制的组合。
> 先理解原理，再用框架，否则会被框架的黑盒坑。

### 4.1 LLM 基础概念

| 概念 | 一句话解释 |
|------|-----------|
| **Token** | LLM 读写的最小单位，不是"字"也不是"词"，1 token ≈ 0.75 个英文单词 |
| **Context Window** | 模型一次能"看到"的最大 token 数，决定了能塞多少上下文 |
| **Temperature** | 控制输出的随机性，越低越确定，越高越有"创意" |
| **System Prompt** | 给模型设定角色、行为边界的指令，放在对话最前面 |
| **Prompt** | 你发给模型的消息，User/Assistant 交替的对话数组 |
| **Streaming** | 逐 token 返回输出，不等完整结果——用户体验的关键 |
| **Embedding** | 把文本转成向量（一串数字），用来做语义搜索 |

### 4.2 必备概念

| 概念 | 说明 | 重要度 |
|------|------|--------|
| **Function Calling / Tool Use** | 模型不直接执行代码，而是输出"我想调这个函数，参数是这些"的 JSON。你的代码真正执行并返回结果给模型。这是 Agent 的基石。 | ★★★★★ |
| **RAG (检索增强生成)** | 把知识库（文档、数据库）的内容通过向量搜索捞出来，塞进 Prompt 里，让模型基于这些信息回答。解决"模型不知道你的私有数据"问题。 | ★★★★★ |
| **Memory / 对话记忆** | 多轮对话怎么记住之前聊了什么。方案：最简单是 keep 历史消息数组，复杂场景用摘要/向量记忆。 | ★★★★ |
| **Agent 循环** | 思考 → 调工具 → 拿到结果 → 再思考 → 再调工具 → ... → 最终回复。这个循环是 Agent 区别于"一次性回答问题"的核心。 | ★★★★★ |

### 4.3 动手路径（从裸调到框架）

**第一步：原生 API 调通 (1-2 天)**

用 Python + `httpx` 或 Node.js 直接调 OpenAI / Anthropic 的 Chat Completion API。
不用任何 LLM 框架——手写 HTTP 请求，理解 `messages` 数组结构、`stream` 参数、`tools` 参数。

```python
# 示例：一次带 function calling 的调用
import requests

response = requests.post(
    "https://api.openai.com/v1/chat/completions",
    headers={"Authorization": f"Bearer {api_key}"},
    json={
        "model": "gpt-4o",
        "messages": [{"role": "user", "content": "北京今天天气怎么样？"}],
        "tools": [{
            "type": "function",
            "function": {
                "name": "get_weather",
                "description": "获取指定城市的天气",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "city": {"type": "string"}
                    }
                }
            }
        }]
    }
)
```

**第二步：Function Calling 完整闭环 (2-3 天)**

手写一个 Agent 循环：
1. 用户输入问题
2. 发给 LLM，LLM 返回"我想调用 get_weather(city='北京')"
3. 你的代码解析这个调用，真正调用天气 API
4. 把 API 结果追加到 messages 数组
5. 再次发给 LLM，LLM 基于结果生成最终回复
6. 如果 LLM 还想调工具，重复步骤 3-5

**做完这个，你才真正理解了 Agent。**

**第三步：掌握 RAG (2-3 天)**
1. 文档分块（chunking）策略
2. 用 Embedding API 把文本转成向量
3. 向量数据库选型（从 ChromaDB 开始，轻量免部署）
4. 检索 + 增强 Prompt + 生成 = RAG

**第四步：Agent 框架 (3-5 天)**

| 框架 | 特点 | 建议 |
|------|------|------|
| **Vercel AI SDK** | 对前端最友好，TypeScript 生态，支持多模型 | 如果你主力 Node.js |
| **LangChain** | 最成熟，生态最大，但抽象层多 | 必知，但不一定要用 |
| **LangGraph** | LangChain 团队出的 Agent 编排框架 | 做复杂 Agent 工作流时学 |
| **OpenAI Agents SDK** | OpenAI 官方出的轻量 Agent 框架 | 简洁好用，推荐入门 |
| **Anthropic MCP** | 模型上下文协议——Agent 与工具的标准化交互 | 生态标准，必须了解 |

### 4.4 MCP (Model Context Protocol)

MCP 是 Anthropic 提出的开放协议，让 Agent 通过统一接口连接各种工具和数据源。
类比：MCP 之于 Agent = USB 协议之于外设。

核心概念：
- **MCP Server**：提供工具/资源的服务端
- **MCP Client**：Agent 端，发现并调用 MCP Server 提供的工具
- 一个 MCP Server 写好后，可以被任何实现了 MCP Client 的 Agent 使用，避免重复造轮子

### 4.5 Agent 开发进阶方向

| 方向 | 说明 |
|------|------|
| **多 Agent 协作** | 多个 Agent 各司其职（规划 Agent + 执行 Agent + 审查 Agent） |
| **ReAct / Plan-and-Execute** | Agent 推理模式——ReAct 边想边做，Plan-and-Execute 先规划再执行 |
| **Evaluation & Observability** | Agent 行为的评估、追踪和监控（LangSmith / LangFuse） |
| **Human-in-the-loop** | 关键操作需要人工确认，Agent 不能无限制自主执行 |
| **Prompt Caching** | 降低 API 调用成本和延迟，工程化的重要优化手段 |

---

## Phase 5：实战 — 从零交付一个 Agent 应用

做完一个完整项目，把所有知识串起来。

### 建议项目：智能文档问答 Agent

**功能：** 用户上传 PDF/文档，Agent 能回答关于文档内容的问题。

**技术栈：**
- 前端：React / Next.js
- 后端：Python FastAPI（Agent 逻辑）+ Node BFF（可选，纯前端转过来更友好）
- 数据库：PostgreSQL（业务数据）+ ChromaDB（向量库）
- 文件存储：MinIO（本地） / S3（生产）
- 部署：Docker Compose → VPS + Nginx + 域名 + HTTPS

**涉及的知识点：**

| 步骤 | 用到的知识 |
|------|-----------|
| 文件上传界面 | 前端表单、进度条、拖拽上传 |
| 后端接收文件 | multipart 解析、文件校验、对象存储 |
| 文档解析 | PDF/Word 文本提取（PyPDF2 / python-docx） |
| 文本分块与向量化 | chunking 策略 + Embedding API |
| 存入向量数据库 | ChromaDB 写入与查询 |
| 构建 RAG 对话接口 | 用户问题 → 向量检索 → 拼 Prompt → LLM 生成 → 返回 |
| 多轮对话记忆 | 历史消息管理、上下文窗口溢出处理 |
| 前端对话界面 | 流式输出（SSE/WebSocket）、Markdown 渲染 |
| 部署上线 | Docker 打包、VPS 部署、Nginx 反向代理、域名绑定 |
| 监控与日志 | 请求日志、错误追踪、用量统计 |

---

## 持续学习的资源索引

### 网络基础
- 《图解 HTTP》— 入门友好，图文并茂
- 《计算机网络：自顶向下方法》— 大学教材，想深入啃这个
- [MDN HTTP 文档](https://developer.mozilla.org/en-US/docs/Web/HTTP)

### 后端与数据库
- [Node.js 官方教程](https://nodejs.org/en/learn)
- [Express 官方指南](https://expressjs.com/)
- [FastAPI 官方文档](https://fastapi.tiangolo.com/) — 最好的 Python Web 框架文档之一
- [SQL 教程 (SQLZoo)](https://sqlzoo.net/) — 交互式学 SQL

### 部署与运维
- [Nginx 入门指南](https://nginx.org/en/docs/beginners_guide.html)
- [Docker 从入门到实践](https://yeasy.gitbook.io/docker_practice/)
- [GitHub Actions 文档](https://docs.github.com/en/actions)
- Let's Encrypt + acme.sh 官方文档

### Agent 开发
- [Anthropic API 文档](https://docs.anthropic.com/en/docs)
- [OpenAI API 文档](https://platform.openai.com/docs)
- [Vercel AI SDK](https://sdk.vercel.ai/docs)
- [LangChain 文档](https://python.langchain.com/docs/)
- [MCP 官方文档](https://modelcontextprotocol.io/)
- [Anthropic Agent 开发指南](https://docs.anthropic.com/en/docs/agents-and-tools)

### 必看文章与视频
- Andrej Karpathy — [Intro to Large Language Models](https://www.youtube.com/watch?v=zjkBMFhNj_g) (1h 概念入门)
- Lilian Weng — [LLM Powered Autonomous Agents](https://lilianweng.github.io/posts/2023-06-23-agent/) (Agent 综述必读)
- Anthropic — [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)

---

## 学习节奏建议

按每周可投入 10-15 小时计算：

| 阶段 | 周期 | 目标 |
|------|------|------|
| Phase 1 网络基础 | 1-2 周 | 理解完整网络链路 |
| Phase 2 后端入门 | 3-4 周 | 能独立写 CRUD API |
| Phase 3 工程部署 | 2-3 周 | 能把项目部署到公网并配好域名 |
| Phase 4 Agent 开发 | 4-6 周 | 理解 Agent 原理，能手写 Agent 循环 |
| Phase 5 实战 | 3-4 周 | 交付完整可演示的 Agent 应用 |

**总计约 3-4 个月**，可以把整套链路跑通。关键是每个阶段都有可交付的产出物，而不是"看完文档就完"。

---

## 学习原则

1. **做比看重要。** 每个知识点都要有对应的代码/操作记录。
2. **先理解原理，再用框架。** 框架会变，原理不会。直接上框架等于在黑盒上建高楼。
3. **一个问题一个坑地填。** 遇到看不懂的概念不要跳过，层层往下挖。CORS 看不懂就去查同源策略，同源策略看不懂就去查浏览器安全模型——这才是真正的学习。
4. **把学习过程记录在仓库里。** 这个 repo 就是你的知识库，每个 Phase 建一个目录，放笔记和代码。