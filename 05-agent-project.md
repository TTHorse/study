# 实战 — 从零交付一个 Agent 应用

> 这是整个学习路线的收尾——把前面四个 Phase 学到的所有东西**串成一个能演示的完整产品**。这一阶段不再讲新概念，而是**走完一个真实项目的全流程**：需求拆解 → 架构设计 → 前后端实现 → 部署上线。做完这个项目，你就有了第一个能写进简历/作品集的 Agent 应用。

---

## 目录

- [1. 项目定义：智能文档问答 Agent](#1-项目定义智能文档问答-agent)
  - [1.1 做什么](#11-做什么)
  - [1.2 为什么选这个项目](#12-为什么选这个项目)
  - [1.3 核心用户故事](#13-核心用户故事)
- [2. 架构设计](#2-架构设计)
  - [2.1 整体架构图](#21-整体架构图)
  - [2.2 技术选型与理由](#22-技术选型与理由)
  - [2.3 数据流设计](#23-数据流设计)
  - [2.4 项目结构](#24-项目结构)
- [3. 后端实现](#3-后端实现)
  - [3.1 项目初始化](#31-项目初始化)
  - [3.2 文档上传与解析](#32-文档上传与解析)
  - [3.3 文档分块与向量化](#33-文档分块与向量化)
  - [3.4 RAG 对话接口（流式）](#34-rag-对话接口流式)
  - [3.5 会话记忆管理](#35-会话记忆管理)
- [4. 前端实现](#4-前端实现)
  - [4.1 项目初始化](#41-项目初始化)
  - [4.2 上传界面](#42-上传界面)
  - [4.3 流式对话界面](#43-流式对话界面)
- [5. 联调与测试](#5-联调与测试)
  - [5.1 本地联调](#51-本地联调)
  - [5.2 端到端测试](#52-端到端测试)
  - [5.3 常见问题排查](#53-常见问题排查)
- [6. 部署上线](#6-部署上线)
  - [6.1 Docker 化](#61-docker-化)
  - [6.2 VPS 部署](#62-vps-部署)
  - [6.3 域名与 HTTPS](#63-域名与-https)
- [7. 进阶优化方向](#7-进阶优化方向)
- [附录：完整操作清单](#附录完整操作清单)
- [检验清单](#检验清单)

---

## 1. 项目定义：智能文档问答 Agent

### 1.1 做什么

做一个**智能文档问答应用**：用户上传 PDF/Word/Markdown 文档，然后可以**针对文档内容用自然语言提问**，Agent 基于文档给出带引用的回答，支持多轮对话和流式输出。

```
┌─────────────────────────────────────────────┐
│  📄 智能文档问答                              │
│  ┌──────────────────────────────────────┐   │
│  │  [拖拽上传 PDF/Word/MD]               │   │
│  │  已上传: 公司手册.pdf ✅               │   │
│  └──────────────────────────────────────┘   │
│  ┌──────────────────────────────────────┐   │
│  │  💬 对话区                             │   │
│  │  我: 年假怎么算？                      │   │
│  │  AI: 根据文档，入职满 1 年 5 天...▌   │   │
│  └──────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

### 1.2 为什么选这个项目

| 涉及知识点 | 在项目里怎么用到 |
|-----------|-----------------|
| Phase 1 网络 | 流式输出靠 SSE，前后端跨域要配 CORS |
| Phase 2 后端 | FastAPI 写 RESTful 接口、文件上传、中间件 |
| Phase 3 部署 | Docker Compose 编排多服务、VPS + Nginx + HTTPS |
| Phase 4 Agent | RAG 检索、Function Calling（可选）、流式、记忆 |

一个项目把四个阶段全覆盖，且**复杂度可控**——不会简单到没东西学，也不会难到做不完。

### 1.3 核心用户故事

1. **上传文档**：用户拖拽一个 PDF 到上传区，看到"已上传"提示
2. **提问**：用户在对话框输入"年假怎么算？"，AI 流式返回基于文档的回答
3. **多轮**：用户追问"那产假呢？"，AI 知道在聊同一个文档
4. **引用**：AI 回答时标注"出自第 3 页"，可溯源
5. **换文档**：用户上传新文档，AI 切换到新文档回答

---

## 2. 架构设计

### 2.1 整体架构图

```
┌──────────────────────────────────────────────────────────────┐
│  浏览器 (React + Vite)                                         │
│    ├─ 上传组件 (拖拽 + 进度)                                    │
│    └─ 对话组件 (SSE 流式渲染 + Markdown)                       │
└────────────┬────────────────────────────┬────────────────────┘
             │ POST /api/documents        │ POST /api/chat (SSE)
             ▼                            ▼
┌──────────────────────────────────────────────────────────────┐
│  后端 (FastAPI, Python)                                        │
│    ├─ /api/documents  上传、解析、分块、向量化                  │
│    ├─ /api/chat       RAG 检索 + 拼 prompt + 流式调 LLM         │
│    └─ /api/sessions   会话记忆管理                              │
└──────┬───────────────┬──────────────────┬────────────────────┘
       │               │                  │
       ▼               ▼                  ▼
┌────────────┐  ┌────────────┐    ┌─────────────────┐
│ PostgreSQL │  │ ChromaDB   │    │ Anthropic API    │
│ 文档元数据  │  │ 向量库      │    │ Claude Opus 4.8  │
│ 会话历史    │  │ (chunks)   │    │ (embedding+LLM)  │
└────────────┘  └────────────┘    └─────────────────┘
```

### 2.2 技术选型与理由

| 层 | 技术 | 为什么选它 |
|----|------|-----------|
| 前端 | React + Vite + TypeScript | 前端背景最熟，Vite 开发体验好 |
| 后端 | FastAPI (Python) | Agent 生态最成熟，异步 + 流式原生支持 |
| 业务数据库 | PostgreSQL | 存文档元数据、会话历史，生产级 |
| 向量数据库 | ChromaDB | 轻量、纯 Python、学习阶段免部署；生产可换 pgvector / Qdrant |
| LLM | Claude Opus 4.8 | Phase 4 主力模型，长上下文、结构化输出强 |
| Embedding | text-embedding-3-small (OpenAI) 或 Voyage-3 | 便宜够用；ChromaDB 本地模型也行（学习用） |
| 文档解析 | PyPDF2 / python-docx / markdown | PDF/Word/MD 文本提取 |
| 部署 | Docker Compose + VPS + Nginx | Phase 3 学过的方案，多服务编排 |

> **生活化类比**：这个架构就像**一家小型咨询公司**。前端是**前台接待**（接客户需求）；FastAPI 是**项目经理**（接需求、分派任务、协调整个流程）；PostgreSQL 是**档案柜**（存客户资料、历史沟通记录）；ChromaDB 是**资料检索室**（按语义找相关档案）；Anthropic API 是**外聘专家**（项目经理把整理好的资料寄给专家，专家写答复寄回来）。用户上传文档 = 客户把材料交给公司；用户提问 = 客户咨询；AI 回答 = 专家基于材料给的答复。

### 2.3 数据流设计

**上传文档流：**

```
1. 前端 POST /api/documents (multipart 文件)
2. 后端接收 → 校验类型/大小 → 存到 ./uploads/
3. 提取文本 (PyPDF2 / python-docx)
4. 分块 (RecursiveCharacterTextSplitter, 500 token, 50 重叠)
5. 每块跑 Embedding → 存进 ChromaDB (collection = 文档ID)
6. 文档元数据 (文件名、chunk 数、上传时间) 存 PostgreSQL
7. 返回 { document_id, chunks: N }
```

**对话流（RAG + 流式）：**

```
1. 前端 POST /api/chat (SSE)  body: { session_id, message }
2. 后端拿 message 做 Embedding → 在 ChromaDB 检索 top-5 相关块
3. 拼 prompt：
     system: "基于以下资料回答。没相关内容就说不知道。"
     + 检索到的 5 个块（带来源标注）
     + 历史对话（最近 N 轮 + 摘要）
     + 当前用户问题
4. 调 Claude API，stream=True
5. 边接收 token 边通过 SSE 推给前端
6. 完整回复存 PostgreSQL 作为会话历史
```

### 2.4 项目结构

```
doc-qa-agent/
├── docker-compose.yml
├── .env.example
├── backend/
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── main.py                 # FastAPI 入口
│   ├── config.py               # 配置/环境变量
│   ├── routers/
│   │   ├── documents.py        # /api/documents
│   │   └── chat.py             # /api/chat
│   ├── services/
│   │   ├── parser.py           # 文档解析
│   │   ├── chunker.py          # 分块
│   │   ├── embedder.py         # 向量化
│   │   ├── retriever.py        # 检索
│   │   └── llm.py              # Claude 调用
│   ├── models/
│   │   └── schemas.py          # Pydantic 模型
│   └── uploads/                # 上传文件（.gitignore）
└── frontend/
    ├── Dockerfile
    ├── package.json
    ├── vite.config.ts
    ├── src/
    │   ├── App.tsx
    │   ├── components/
    │   │   ├── UploadPanel.tsx
    │   │   └── ChatPanel.tsx
    │   ├── api/
    │   │   └── client.ts
    │   └── hooks/
    │       └── useSSE.ts
    └── nginx.conf              # 生产用
```

---

## 3. 后端实现

### 3.1 项目初始化

```bash
mkdir doc-qa-agent && cd doc-qa-agent
mkdir -p backend/{routers,services,models,uploads}
cd backend

# requirements.txt
cat > requirements.txt <<'EOF'
fastapi>=0.110
uvicorn[standard]>=0.27
python-multipart>=0.0.9
anthropic>=0.40
chromadb>=0.4
pypdf>=4.0
python-docx>=1.1
markdown>=3.5
sqlalchemy>=2.0
asyncpg>=0.29
python-dotenv>=1.0
EOF

pip install -r requirements.txt
```

```python
# backend/config.py
import os
from dotenv import load_dotenv

load_dotenv()

class Settings:
    ANTHROPIC_API_KEY = os.environ["ANTHROPIC_API_KEY"]  # 必填
    ANTHROPIC_MODEL = os.getenv("LLM_MODEL", "claude-opus-4-8")
    EMBEDDING_MODEL = os.getenv("EMBEDDING_MODEL", "text-embedding-3-small")
    OPENAI_API_KEY = os.environ.get("OPENAI_API_KEY", "")  # embedding 用
    DATABASE_URL = os.getenv("DATABASE_URL", "postgresql://user:pass@db:5432/docqa")
    UPLOAD_DIR = "./uploads"
    CHUNK_SIZE = 500       # token
    CHUNK_OVERLAP = 50
    TOP_K = 5              # 检索召回数
    MAX_HISTORY_TURNS = 6  # 保留最近几轮原文

settings = Settings()
```

```python
# backend/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from routers import documents, chat
import os

app = FastAPI(title="Doc QA Agent")

# CORS——前端开发期跑在 5173，要允许它跨域
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173"],  # 生产换成你的域名
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

os.makedirs("./uploads", exist_ok=True)

app.include_router(documents.router, prefix="/api")
app.include_router(chat.router, prefix="/api")

@app.get("/health")
def health():
    return {"status": "ok"}
```

### 3.2 文档上传与解析

```python
# backend/services/parser.py
from pypdf import PdfReader
from docx import Document
import markdown
from pathlib import Path

def extract_text(file_path: str) -> str:
    """根据扩展名提取纯文本"""
    ext = Path(file_path).suffix.lower()
    if ext == ".pdf":
        reader = PdfReader(file_path)
        return "\n\n".join(page.extract_text() or "" for page in reader.pages)
    elif ext == ".docx":
        doc = Document(file_path)
        return "\n\n".join(p.text for p in doc.paragraphs if p.text.strip())
    elif ext in (".md", ".markdown"):
        # md 直接取原文（也可以转 HTML 但 LLM 读原文更准）
        return Path(file_path).read_text(encoding="utf-8")
    elif ext == ".txt":
        return Path(file_path).read_text(encoding="utf-8")
    else:
        raise ValueError(f"不支持的文件类型: {ext}")
```

```python
# backend/routers/documents.py
from fastapi import APIRouter, UploadFile, File, HTTPException
from services.parser import extract_text
from services.chunker import chunk_text
from services.embedder import embed_and_store
import uuid, os

router = APIRouter()

ALLOWED = {".pdf", ".docx", ".md", ".markdown", ".txt"}
MAX_SIZE = 10 * 1024 * 1024  # 10MB

@router.post("/documents")
async def upload(file: UploadFile = File(...)):
    # 1. 校验
    ext = os.path.splitext(file.filename)[1].lower()
    if ext not in ALLOWED:
        raise HTTPException(400, f"不支持的类型: {ext}")
    if file.size and file.size > MAX_SIZE:
        raise HTTPException(413, "文件超过 10MB")

    # 2. 存盘
    doc_id = str(uuid.uuid4())
    save_path = f"./uploads/{doc_id}{ext}"
    with open(save_path, "wb") as f:
        f.write(await file.read())

    # 3. 解析 → 分块 → 向量化入库
    try:
        text = extract_text(save_path)
        chunks = chunk_text(text)                # 下一节实现
        embed_and_store(doc_id, chunks)          # 下一节实现
    except Exception as e:
        raise HTTPException(500, f"文档处理失败: {e}")

    return {"document_id": doc_id, "filename": file.filename, "chunks": len(chunks)}
```

### 3.3 文档分块与向量化

```python
# backend/services/chunker.py
import re

def chunk_text(text: str, size: int = 500, overlap: int = 50) -> list[str]:
    """
    递归分块：先按段落切，超长再按句子切，尽量保持语义完整。
    size/overlap 单位是字符（中文场景；英文按 token 估算更准）。
    """
    # 按双换行切段落
    paragraphs = [p.strip() for p in text.split("\n\n") if p.strip()]

    chunks = []
    cur = ""
    for p in paragraphs:
        # 段落本身超长，按句号切
        if len(p) > size:
            if cur:
                chunks.append(cur)
                cur = ""
            sentences = re.split(r"(?<=[。.!?])\s+", p)
            for s in sentences:
                if len(cur) + len(s) > size:
                    if cur:
                        chunks.append(cur)
                    # overlap：保留上一块结尾
                    cur = cur[-overlap:] + s if cur else s
                else:
                    cur += s
        elif len(cur) + len(p) > size:
            chunks.append(cur)
            cur = p
        else:
            cur = cur + "\n\n" + p if cur else p

    if cur:
        chunks.append(cur)
    return chunks
```

```python
# backend/services/embedder.py
import chromadb
from openai import OpenAI
from config import settings

_oai = OpenAI(api_key=settings.OPENAI_API_KEY) if settings.OPENAI_API_KEY else None

# ChromaDB 持久化到磁盘（生产用 Postgres + pgvector 或独立 Qdrant）
_chroma = chromadb.PersistentClient(path="./chroma_data")

def _embed(texts: list[str]) -> list[list[float]]:
    """商业 embedding（更准）；没 key 就用 Chroma 自带的本地模型"""
    if _oai:
        resp = _oai.embeddings.create(
            model=settings.EMBEDDING_MODEL, input=texts
        )
        return [d.embedding for d in resp.data]
    return None  # 让 Chroma 用默认模型自动 embed

def embed_and_store(doc_id: str, chunks: list[str]):
    col = _chroma.get_or_create_collection(name=f"doc_{doc_id}")
    embeddings = _embed(chunks)
    if embeddings:
        col.add(
            documents=chunks,
            embeddings=embeddings,
            ids=[f"{doc_id}_{i}" for i in range(len(chunks))],
            metadatas=[{"doc_id": doc_id, "chunk_index": i} for i in range(len(chunks))],
        )
    else:
        # 没配 OpenAI key，让 Chroma 用本地默认模型
        col.add(
            documents=chunks,
            ids=[f"{doc_id}_{i}" for i in range(len(chunks))],
            metadatas=[{"doc_id": doc_id, "chunk_index": i} for i in range(len(chunks))],
        )
```

```python
# backend/services/retriever.py
from services.embedder import _chroma, _embed
from config import settings

def retrieve(doc_id: str, query: str, top_k: int = settings.TOP_K) -> list[str]:
    col = _chroma.get_or_create_collection(name=f"doc_{doc_id}")
    if not col.count():
        return []
    # 优先用商业 embedding 查；否则让 Chroma 自动 embed query
    q_emb = _embed([query])
    if q_emb:
        res = col.query(query_embeddings=q_emb, n_results=top_k)
    else:
        res = col.query(query_texts=[query], n_results=top_k)
    return res["documents"][0] if res["documents"] else []
```

### 3.4 RAG 对话接口（流式）

这是项目核心——把检索、prompt 拼装、流式调用 LLM、SSE 推送串起来。

```python
# backend/routers/chat.py
from fastapi import APIRouter
from fastapi.responses import StreamingResponse
from pydantic import BaseModel
import anthropic, json
from config import settings
from services.retriever import retrieve
from services.memory import get_history, append_message  # 下一节

router = APIRouter()
client = anthropic.Anthropic()

class ChatRequest(BaseModel):
    session_id: str
    document_id: str
    message: str

SYSTEM_PROMPT = """你是一个文档问答助手。请严格基于下面提供的【参考资料】回答用户问题。

规则：
1. 只用【参考资料】里的信息回答，不要编造。
2. 如果资料里没有相关内容，明确说"资料里没有这个信息"，不要猜。
3. 回答时在关键事实后标注来源，格式：(资料第 N 段)。
4. 用简洁的中文回答。
"""

@router.post("/chat")
async def chat(req: ChatRequest):
    def event_stream():
        # 1. 检索
        chunks = retrieve(req.document_id, req.message)
        context = "\n\n".join(
            f"【资料第 {i+1} 段】{c}" for i, c in enumerate(chunks)
        ) if chunks else "（无相关资料）"

        # 2. 取历史
        history = get_history(req.session_id)

        # 3. 拼 messages
        messages = history + [{"role": "user", "content": req.message}]
        system = SYSTEM_PROMPT + f"\n\n【参考资料】\n{context}"

        # 4. 流式调 LLM
        full_text = ""
        with client.messages.stream(
            model=settings.ANTHROPIC_MODEL,
            max_tokens=1024,
            system=system,
            messages=messages,
        ) as stream:
            for text in stream.text_stream:
                full_text += text
                # SSE 格式：每个 chunk 一个 data: 事件
                yield f"data: {json.dumps({'type':'token','text':text})}\n\n"

        # 5. 存历史
        append_message(req.session_id, "user", req.message)
        append_message(req.session_id, "assistant", full_text)

        # 6. 结束信号
        yield f"data: {json.dumps({'type':'done'})}\n\n"

    return StreamingResponse(
        event_stream(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "X-Accel-Buffering": "no",  # Nginx 不要缓冲
        },
    )
```

### 3.5 会话记忆管理

最简版用内存字典存历史（重启就丢）；生产用 PostgreSQL。这里给一个内存版骨架，DB 版留作练习。

```python
# backend/services/memory.py
from collections import defaultdict
from config import settings

# 内存存储——生产换 Postgres
_store: dict[str, list[dict]] = defaultdict(list)

def get_history(session_id: str) -> list[dict]:
    """返回最近 N 轮历史（滑窗策略）"""
    msgs = _store[session_id]
    # 每轮 2 条（user + assistant），保留最近 MAX_HISTORY_TURNS 轮
    return msgs[-settings.MAX_HISTORY_TURNS * 2:]

def append_message(session_id: str, role: str, content: str):
    _store[session_id].append({"role": role, "content": content})
```

> **进阶练习：** 把内存存储换成 PostgreSQL——建一张 `messages(session_id, role, content, created_at)` 表，`get_history` 查最近 N 条。再进一步：当历史超过 20 轮时，调一次 LLM 把最早 10 轮总结成摘要，替换原文（摘要记忆）。

---

## 4. 前端实现

### 4.1 项目初始化

```bash
cd ../  # 回到项目根目录
npm create vite@latest frontend -- --template react-ts
cd frontend
npm install
npm install react-markdown remark-gfm
```

配置 vite 代理，开发期把 `/api` 转到后端 8000 端口：

```typescript
// frontend/vite.config.ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  server: {
    proxy: {
      "/api": {
        target: "http://localhost:8000",
        changeOrigin: true,
      },
    },
  },
});
```

### 4.2 上传界面

```tsx
// frontend/src/components/UploadPanel.tsx
import { useState, useRef } from "react";

export function UploadPanel({ onUploaded }: { onUploaded: (docId: string, name: string) => void }) {
  const [uploading, setUploading] = useState(false);
  const [fileName, setFileName] = useState<string | null>(null);
  const inputRef = useRef<HTMLInputElement>(null);

  const handleFile = async (file: File) => {
    setUploading(true);
    setFileName(file.name);
    const form = new FormData();
    form.append("file", file);
    try {
      const res = await fetch("/api/documents", { method: "POST", body: form });
      if (!res.ok) throw new Error((await res.text()) || "上传失败");
      const data = await res.json();
      onUploaded(data.document_id, file.name);
    } catch (e) {
      alert(`上传失败: ${e.message}`);
      setFileName(null);
    } finally {
      setUploading(false);
    }
  };

  return (
    <div
      onDragOver={(e) => e.preventDefault()}
      onDrop={(e) => {
        e.preventDefault();
        if (e.dataTransfer.files[0]) handleFile(e.dataTransfer.files[0]);
      }}
      onClick={() => inputRef.current?.click()}
      style={{ border: "2px dashed #888", padding: 24, textAlign: "center", cursor: "pointer" }}
    >
      <input
        ref={inputRef}
        type="file"
        accept=".pdf,.docx,.md,.txt"
        hidden
        onChange={(e) => e.target.files?.[0] && handleFile(e.target.files[0])}
      />
      {uploading ? (
        <p>上传解析中…</p>
      ) : fileName ? (
        <p>✅ {fileName}</p>
      ) : (
        <p>拖拽文件到这里，或点击选择（PDF/Word/MD/TXT，≤10MB）</p>
      )}
    </div>
  );
}
```

### 4.3 流式对话界面

```tsx
// frontend/src/components/ChatPanel.tsx
import { useState, useRef, useEffect } from "react";
import ReactMarkdown from "react-markdown";

type Msg = { role: "user" | "assistant"; content: string };

export function ChatPanel({ docId }: { docId: string | null }) {
  const [msgs, setMsgs] = useState<Msg[]>([]);
  const [input, setInput] = useState("");
  const [streaming, setStreaming] = useState(false);
  const [sessionId] = useState(() => crypto.randomUUID());
  const bottomRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    bottomRef.current?.scrollIntoView({ behavior: "smooth" });
  }, [msgs]);

  const send = async () => {
    if (!input.trim() || !docId || streaming) return;
    const userMsg = input.trim();
    setInput("");
    setMsgs((m) => [...m, { role: "user", content: userMsg }, { role: "assistant", content: "" }]);
    setStreaming(true);

    // 用 fetch 接 SSE——比 EventSource 灵活（能 POST）
    const res = await fetch("/api/chat", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ session_id: sessionId, document_id: docId, message: userMsg }),
    });
    if (!res.body) return;

    const reader = res.body.getReader();
    const decoder = new TextDecoder();
    let buffer = "";

    while (true) {
      const { done, value } = await reader.read();
      if (done) break;
      buffer += decoder.decode(value);
      // SSE：按 \n\n 分事件
      const events = buffer.split("\n\n");
      buffer = events.pop() || "";
      for (const ev of events) {
        const line = ev.trim();
        if (!line.startsWith("data: ")) continue;
        const payload = JSON.parse(line.slice(6));
        if (payload.type === "token") {
          setMsgs((m) => {
            const last = m[m.length - 1];
            return [...m.slice(0, -1), { ...last, content: last.content + payload.text }];
          });
        }
      }
    }
    setStreaming(false);
  };

  return (
    <div style={{ display: "flex", flexDirection: "column", height: "100%" }}>
      <div style={{ flex: 1, overflowY: "auto", padding: 16 }}>
        {msgs.map((m, i) => (
          <div key={i} style={{ textAlign: m.role === "user" ? "right" : "left", margin: "8px 0" }}>
            <div
              style={{
                display: "inline-block",
                padding: "8px 12px",
                borderRadius: 8,
                background: m.role === "user" ? "#dcf8c6" : "#f1f1f1",
                maxWidth: "80%",
                textAlign: "left",
              }}
            >
              {m.role === "assistant" ? (
                <ReactMarkdown>{m.content || "▌"}</ReactMarkdown>
              ) : (
                m.content
              )}
            </div>
          </div>
        ))}
        <div ref={bottomRef} />
      </div>
      <div style={{ display: "flex", padding: 12, borderTop: "1px solid #ddd" }}>
        <input
          value={input}
          onChange={(e) => setInput(e.target.value)}
          onKeyDown={(e) => e.key === "Enter" && send()}
          placeholder={docId ? "问点关于文档的…" : "请先上传文档"}
          disabled={!docId || streaming}
          style={{ flex: 1, marginRight: 8, padding: 8 }}
        />
        <button onClick={send} disabled={!docId || streaming || !input.trim()}>
          发送
        </button>
      </div>
    </div>
  );
}
```

```tsx
// frontend/src/App.tsx
import { useState } from "react";
import { UploadPanel } from "./components/UploadPanel";
import { ChatPanel } from "./components/ChatPanel";

export default function App() {
  const [doc, setDoc] = useState<{ id: string; name: string } | null>(null);

  return (
    <div style={{ maxWidth: 800, margin: "0 auto", padding: 16, height: "100vh", display: "flex", flexDirection: "column" }}>
      <h2>📄 智能文档问答</h2>
      <UploadPanel onUploaded={(id, name) => setDoc({ id, name })} />
      {doc && <p style={{ color: "#666" }}>当前文档: {doc.name}</p>}
      <div style={{ flex: 1, border: "1px solid #ddd", marginTop: 12 }}>
        <ChatPanel docId={doc?.id ?? null} />
      </div>
    </div>
  );
}
```

---

## 5. 联调与测试

### 5.1 本地联调

开两个终端：

```bash
# 终端 1：后端
cd backend
uvicorn main:app --reload --port 8000

# 终端 2：前端
cd frontend
npm run dev   # 跑在 5173，/api 走代理到 8000
```

打开 `http://localhost:5173`，上传一个 PDF 试问。

### 5.2 端到端测试

准备一份测试文档（比如公司手册.pdf），验证：

1. **上传成功**：返回 `chunks: N`（N > 0 说明解析+分块正常）
2. **基础问答**：问"年假怎么算"——AI 基于文档答，不编造
3. **流式效果**：回答逐字出现，不是卡几秒后整段
4. **多轮**：追问"那产假呢"——AI 知道在聊同一文档
5. **拒答**：问文档没的内容——AI 说"资料里没有"
6. **引用标注**：回答带"(资料第 N 段)"
7. **换文档**：上传新文档后切换，AI 用新文档回答

### 5.3 常见问题排查

| 现象 | 排查 |
|------|------|
| 上传报 422 | 检查 `python-multipart` 装了没 |
| 上传后 chunks=0 | PDF 是扫描件（图片）——PyPDF2 提不出文本，要上 OCR（pytesseract） |
| 对话报 401 | `ANTHROPIC_API_KEY` 没设或失效 |
| 流式不流，整段才出 | Nginx/Vite 缓冲了——确认 `X-Accel-Buffering: no`、Vite proxy 没开压缩 |
| AI 不基于文档答 | 检索没召回——`retrieve` 返回空；或 ChromaDB collection 名对不上 |
| CORS 报错 | 后端 `allow_origins` 没加前端的源 |
| AI 回答带"资料第 N 段"但内容对不上 | 分块切得不好——调 chunk_size/overlap，或换更好的 embedding |

---

## 6. 部署上线

### 6.1 Docker 化

```dockerfile
# backend/Dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```dockerfile
# frontend/Dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package.json package-lock.json* ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

```nginx
# frontend/nginx.conf
server {
    listen 80;
    root /usr/share/nginx/html;
    location / { try_files $uri /index.html; }
    location /api/ {
        proxy_pass http://backend:8000;
        proxy_set_header Host $host;
        proxy_buffering off;              # SSE 必须关
        proxy_read_timeout 300s;
    }
}
```

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: docqa
    volumes:
      - pgdata:/var/lib/postgresql/data

  backend:
    build: ./backend
    environment:
      ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY}
      OPENAI_API_KEY: ${OPENAI_API_KEY}
      DATABASE_URL: postgresql://user:pass@db:5432/docqa
    volumes:
      - ./backend/uploads:/app/uploads
      - chroma_data:/app/chroma_data
    depends_on: [db]

  frontend:
    build: ./frontend
    ports: ["80:80"]
    depends_on: [backend]

volumes:
  pgdata:
  chroma_data:
```

```bash
# .env.example
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...
LLM_MODEL=claude-opus-4-8
```

### 6.2 VPS 部署

```bash
# 本地构建并推到服务器（Phase 3 学过的方法）
docker compose build
docker save docqa-frontend docqa-backend | gzip > images.tar.gz
scp images.tar.gz docker-compose.yml .env user@your-server:~/docqa/

# 服务器上
ssh user@your-server
cd ~/docqa
gunzip -c images.tar.gz | docker load
docker compose up -d
docker compose ps          # 三个容器都 Up
curl localhost/health      # 后端通
curl localhost             # 前端首页
```

### 6.3 域名与 HTTPS

```bash
# 服务器装 Nginx + certbot（Phase 3 学过）
sudo apt install nginx certbot python3-certbot-nginx

# 顶层 Nginx 反代到前端容器的 80 端口
sudo tee /etc/nginx/sites-available/docqa <<'EOF'
server {
    server_name docqa.yourdomain.com;
    location / {
        proxy_pass http://localhost:80;   # 前端容器
        proxy_set_header Host $host;
    }
}
EOF
sudo ln -s /etc/nginx/sites-available/docqa /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx

# 申请 HTTPS 证书
sudo certbot --nginx -d docqa.yourdomain.com
```

访问 `https://docqa.yourdomain.com`——你的 Agent 应用上线了。🎉

---

## 7. 进阶优化方向

项目跑通后，这些是把"能演示"升级到"能生产"的方向：

| 方向 | 做什么 | 价值 |
|------|--------|------|
| **引用溯源** | 检索结果带页码/段号 metadata，前端渲染成可点击跳转的引用 | 可信度大增 |
| **重排序 (rerank)** | 向量召回 top-20 后，用 rerank 模型重排取 top-5 | 检索准确度提升明显 |
| **多文档** | 支持上传多份文档，检索时跨文档召回 | 实用性大增 |
| **增量更新** | 文档改动只重新 embed 变化的 chunk | 大文档场景省成本 |
| **观测性** | 接 LangFuse/LangSmith，记录每次检索结果、prompt、token 用量 | 线上排错、成本分析必备 |
| **评估集** | 准备 20-30 个标准问答对，改 prompt/检索参数后回归测 | 防止改 A 坏 B |
| **混合检索** | 关键词 (BM25) + 向量，结果融合 | 专有名词、代码场景更准 |
| **Human-in-loop** | 关键操作（如调"发邮件"工具）加人工确认 | Agent 安全 |
| **多模态** | 支持图片/PDF 里的图表（Claude vision） | 超越纯文本 |

---

## 附录：完整操作清单

### 准备阶段

```
[ ] 注册 Anthropic 拿 API Key
[ ] （可选）注册 OpenAI 拿 embedding Key
[ ] 准备一份测试用 PDF/Word
[ ] 买一台 VPS（Phase 3 的那台就行）+ 一个域名
```

### 后端（2-3 天）

```
[ ] 建 backend/ 项目骨架，装依赖
[ ] 实现 parser（PDF/Word/MD 文本提取）
[ ] 实现 chunker（递归分块）
[ ] 实现 embedder（ChromaDB + embedding API）
[ ] 实现 retriever（向量检索 top-K）
[ ] 实现 /api/documents（上传→解析→分块→入库）
[ ] 实现 /api/chat（SSE 流式 RAG）
[ ] 实现内存版 memory（滑窗历史）
[ ] curl 测通两个接口
```

### 前端（2-3 天）

```
[ ] Vite + React + TS 脚手架
[ ] 配 vite proxy 到 8000
[ ] UploadPanel（拖拽上传 + 进度）
[ ] ChatPanel（fetch + ReadableStream 接 SSE）
[ ] ReactMarkdown 渲染回答
[ ] 多轮对话状态管理
```

### 联调（1 天）

```
[ ] 两个终端起前后端
[ ] 上传测试文档，验证 chunks > 0
[ ] 端到端跑 7 个测试用例（5.2 节）
[ ] 修掉流式缓冲、CORS 等常见坑
```

### 部署（1 天）

```
[ ] 写 backend/frontend Dockerfile
[ ] 写 docker-compose.yml（db + backend + frontend）
[ ] 本地 docker compose up 验证
[ ] 推到 VPS，docker compose up -d
[ ] 配域名 A 记录 → VPS IP
[ ] 顶层 Nginx 反代 + certbot HTTPS
[ ] https://docqa.yourdomain.com 全流程跑通
```

---

## 检验清单

在不看笔记的情况下能解释清楚并动手实现：

- [ ] 这个项目用到了 Phase 1-4 的哪些知识？能逐项对应吗？
- [ ] 上传一个文档，从用户拖拽到 ChromaDB 入库，经过哪些步骤？能画出数据流图吗？
- [ ] 一次对话请求，从用户点"发送"到 AI 第一个字出现在屏幕上，经过哪些步骤？
- [ ] 为什么对话接口用 SSE 而不是普通 JSON？为什么前端用 `fetch` 而不是 `EventSource`？
- [ ] RAG 在这个项目里解决了什么问题？如果不用 RAG 直接问 LLM 会怎样？
- [ ] 检索召回的 chunks 怎么拼进 prompt？为什么强调"资料里没有就说不知道"？
- [ ] 多轮对话怎么实现的？历史消息存在哪？滑窗策略是为了解决什么问题？
- [ ] 文档分块的大小和重叠怎么权衡？切太大/太小各有什么问题？
- [ ] 流式输出在 Nginx 后面有哪些坑？（buffering、timeout）
- [ ] docker-compose 里三个服务怎么通信？数据持久化（uploads、chroma、pg）怎么保证？
- [ ] 上线后怎么监控 Agent 的回答质量、token 成本？（观测性）
- [ ] 从零独立搭一个类似项目，从需求到上线，跑通

---

> **恭喜**：走到这里，你已经从"只会写前端页面"成长为**能独立交付一个全栈 Agent 应用**的工程师。这套能力组合（前端 + 后端 + 部署 + Agent）在当下市场上非常稀缺。接下来持续学习的方向在 [README.md](../README.md#持续学习的资源索引) 的资源索引里。保持动手、保持记录、保持好奇——这才是起点，不是终点。
