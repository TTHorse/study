# Agent 开发基础

> 走到这里，你已经能写出后端 API、把它部署到公网。这一阶段的目标是**理解 LLM 和 Agent 到底是什么**——剥开"AI 智能"的神秘外衣，看到底下其实就是"LLM + 工具调用 + 循环控制"的组合。这一阶段不追求做出多炫的产品，而是**把每一个核心概念都拆开揉碎，亲手写一遍**。直接上框架等于在黑盒上建高楼——框架会变，原理不会。

---

## 目录

- [1. LLM 基础概念](#1-llm-基础概念)
  - [1.1 LLM 是什么（一句话祛魅）](#11-llm-是什么一句话祛魅)
  - [1.2 Token — 模型读写的最小单位](#12-token--模型读写的最小单位)
  - [1.3 Context Window — 模型一次能看多大](#13-context-window--模型一次能看多大)
  - [1.4 三种角色：System / User / Assistant](#14-三种角色system--user--assistant)
  - [1.5 采样参数：Temperature 等](#15-采样参数temperature-等)
  - [1.6 Streaming — 逐 token 返回](#16-streaming--逐-token-返回)
  - [1.7 Embedding — 把文本变成向量](#17-embedding--把文本变成向量)
- [2. 第一次裸调 API](#2-第一次裸调-api)
  - [2.1 装包与鉴权](#21-装包与鉴权)
  - [2.2 最简调用](#22-最简调用)
  - [2.3 多轮对话（API 是无状态的）](#23-多轮对话api-是无状态的)
  - [2.4 流式输出](#24-流式输出)
- [3. Function Calling — Agent 的基石](#3-function-calling--agent-的基石)
  - [3.1 为什么需要 Function Calling](#31-为什么需要-function-calling)
  - [3.2 工具调用的完整时序](#32-工具调用的完整时序)
  - [3.3 手写 Agent 循环](#33-手写-agent-循环)
  - [3.4 并行工具调用](#34-并行工具调用)
- [4. RAG — 让模型用你的私有数据](#4-rag--让模型用你的私有数据)
  - [4.1 为什么需要 RAG](#41-为什么需要-rag)
  - [4.2 向量检索的原理](#42-向量检索的原理)
  - [4.3 文档分块（chunking）](#43-文档分块chunking)
  - [4.4 从零搭一个最小 RAG](#44-从零搭一个最小-rag)
- [5. Memory — 多轮对话怎么记](#5-memory--多轮对话怎么记)
  - [5.1 短期记忆：历史消息数组](#51-短期记忆历史消息数组)
  - [5.2 上下文窗口溢出](#52-上下文窗口溢出)
  - [5.3 摘要记忆与向量记忆](#53-摘要记忆与向量记忆)
- [6. Agent 框架与生态](#6-agent-框架与生态)
  - [6.1 框架对比](#61-框架对比)
  - [6.2 Vercel AI SDK（前端友好）](#62-vercel-ai-sdk前端友好)
  - [6.3 LangChain / LangGraph](#63-langchain--langgraph)
  - [6.4 MCP（Model Context Protocol）](#64-mcpmodel-context-protocol)
- [7. Agent 进阶方向](#7-agent-进阶方向)
- [附录：常用模型速查](#附录常用模型速查)
- [检验清单](#检验清单)

---

## 1. LLM 基础概念

### 1.1 LLM 是什么（一句话祛魅）

**LLM (Large Language Model)** 本质上就是一个**接收一段文本、输出一段文本**的函数。你给它一串 token，它根据训练学到的概率分布，预测"下一个 token 最可能是什么"，一个一个地吐出来。

它不是数据库，不是搜索引擎，不是计算器。它有的只是"从训练语料里学到的语言规律"。所以：

- 它能"写"代码，但不保证能"跑"——因为它在生成概率最高的文本，不是在执行
- 它"知道"北京是首都，是因为训练语料里出现过太多次，不是因为它查了百科
- 它会"幻觉"（hallucination）——一本正经地编造不存在的事实，因为概率上那些词组合起来很顺

> **生活化类比**：LLM 就像一个**读过几千万本书但没有任何外部工具的学者**——你问他"1999 年世界杯谁赢了"，他可能答对（因为书里见过）；你问他"今天北京天气怎么样"，他只能猜（他看不到今天）；你问他"1234 × 5678 等于多少"，他可能算错（他在生成数字，不是在算）。给他**工具**（搜索引擎、计算器、数据库），他才真正"有用"——这就是 Agent 要解决的问题。

### 1.2 Token — 模型读写的最小单位

LLM 不直接读"字符"也不读"词"，它读 **token**。Token 是模型把文本切成的最小片段：一个常见英文单词可能是 1 个 token，一个生僻词可能被切成 2-3 个 token，一个中文字通常是 1-2 个 token。

```
"Hello, world!"        →  [Hello, ",", world, !]            4 tokens
"你好，世界！"          →  [你, 好, ，, 世界, ！]            约 5 tokens
"unbelievable"         →  [un, bel, iev, able]             4 tokens
```

**为什么要在意 token：**

- **计费按 token 算**：输入多少 token × 单价 + 输出多少 token × 单价。写 prompt 啰嗦 = 烧钱。
- **上下文窗口是 token 数**：你的 prompt + 历史 + 模型输出，加起来不能超过模型的 context window。
- **不要用 `len(string)` 估算**：中文一个字常常是 1-2 token，英文一个单词约 1 token，字符数和 token 数差很多。要用 API 提供的 `count_tokens` 接口或 `tiktoken` 这类分词器估算。

> **生活化类比**：token 就像**给学者看的"卡片"**——你不能直接塞一本书给他，得把书撕成一张张卡片（token），他按卡片读。卡片越多，读得越久、收费越贵。学者一次只能捧住 N 张卡片（context window），超了就放不下。

### 1.3 Context Window — 模型一次能看多大

**Context Window** 是模型一次输入能容纳的最大 token 数。当前主流模型从 128K 到 1M 不等。

| 模型 | Context Window |
|------|----------------|
| Claude Opus 4.8 / Sonnet 4.6 | 1M |
| Claude Haiku 4.5 | 200K |
| GPT-4o | 128K |

**实际能用的远小于标称值：**

- prompt + 历史 + 模型输出都算在 context window 里
- 接近上限时延迟会变高、成本会变贵
- 真正的"长上下文"应用（读完一整本书）要配合 RAG / 摘要，而不是硬塞

> **生活化类比**：context window 就像学者的**书桌大小**——桌子就这么大，一次只能摊开这么多卡片。桌子大（1M token）不代表你要把所有书都摊上去——摊得越多，他翻找越慢、收费越贵。聪明的做法是：只在桌上摊**和当前问题最相关的那几页**，这恰恰是 RAG 在做的事。

### 1.4 三种角色：System / User / Assistant

一次 API 调用的 `messages` 数组里，每条消息有一个 `role`：

| Role | 谁在说话 | 作用 |
|------|---------|------|
| **system** | 你（开发者） | 给模型设定角色、行为边界、输出格式。放在对话最前面，权重最高 |
| **user** | 终端用户 | 提问、下指令 |
| **assistant** | 模型 | 模型之前的回复（多轮对话时回传给它，让它"记得"自己说过什么） |

```python
messages = [
    {"role": "system",    "content": "你是一个简洁的编程助手，回答不超过 3 句话。"},
    {"role": "user",      "content": "Python 里怎么反转字符串？"},
    {"role": "assistant", "content": "用切片：s[::-1]"},
    {"role": "user",      "content": "那列表呢？"},
]
```

**关键点：**

- **system prompt 是稳定层**——尽量不变，可以做 prompt caching 省钱
- **历史消息是增长层**——每轮追加，越聊越长
- **当前用户输入是变化层**——每轮都不同

> **生活化类比**：system prompt 就像**给学者的"岗位说明书"**——你是哪个部门的、怎么说话、什么能说什么不能说。user/assistant 交替就像**你和学者的对话记录**——你问一句、他答一句，下一轮你把整段对话记录再递给他，他才知道上下文。这也是为什么 API 是无状态的——学者没有记忆，全靠你每次把"对话记录"完整递过去。

### 1.5 采样参数：Temperature 等

> ⚠️ **重要**：Claude 4.6+ 的 Opus/Sonnet 模型（包括 Opus 4.8）**已经移除了 `temperature`、`top_p`、`top_k` 参数**，传入会直接 400 报错。这些参数在老模型和某些其他厂商模型上还存在，理解概念即可，新代码不要依赖。

| 参数 | 作用 | 取值 |
|------|------|------|
| **temperature** | 控制随机性。0 = 几乎确定，1 = 比较放飞 | 0.0 – 1.0 |
| **top_p** | nucleus sampling，从概率前 p 的候选里挑 | 0.0 – 1.0 |
| **top_k** | 只从前 k 个候选里挑 | 整数 |

**经验：**

- 需要稳定/可重现（代码生成、JSON 输出、分类）→ 低 temperature（0~0.3）
- 需要创意（写故事、起名字、brainstorm）→ 高 temperature（0.7~1.0）
- Claude 4.6+ 改用 **effort** 参数（`output_config: {effort: "low"|"medium"|"high"|"max"}`）控制思考深度，以及用 **prompting** 替代 temperature 控制风格

> **生活化类比**：temperature 就像**学者喝酒的量**——0 度是清醒状态，问什么答什么，稳但无聊；1 度是微醺，答案花样百出，有趣但可能跑偏。需要他审合同就别让他喝；需要他起名字就让他喝两口。新模型把酒收了，改成"思考时长"旋钮（effort）——让他想久一点，答案也会更细。

### 1.6 Streaming — 逐 token 返回

默认 API 调用是"等模型把整段写完一次性返回"——长答案要等十几秒，用户体验很差。**Streaming（流式输出）** 让模型一边生成一边把 token 推给你，前端可以一边渲染，像 ChatGPT 那种"打字机"效果。

```
非流式：   用户提问 ──→ [等 8 秒] ──→ 整段答案
流式：     用户提问 ──→ "你" "好" "，" "我" "是" ... ──→ 答案完
```

技术上是通过 **SSE (Server-Sent Events)** 推送，每个 token 一个 `data:` 事件。

> **生活化类比**：非流式就像**写信**——写完整封才寄出，收到时要等很久。流式就像**打电话**——对方一边说一边传到你耳朵里，你一边听一边理解。对长答案来说，流式让"首字延迟"从十几秒降到几百毫秒，体感完全不同。

### 1.7 Embedding — 把文本变成向量

**Embedding** 是把一段文本映射成一串数字（向量），这串数字代表了文本的"语义"。语义相近的文本，向量也相近。

```
"我喜欢吃苹果"     →  [0.12, -0.34, 0.88, ..., 0.05]   (1536 维)
"我爱吃水果"       →  [0.10, -0.31, 0.85, ..., 0.07]   (语义近，向量近)
"今天股市大跌"     →  [-0.45, 0.22, -0.11, ...]        (语义远，向量远)
```

怎么衡量"向量近"？常用**余弦相似度**——两个向量方向越一致，相似度越高（最大为 1）。

**Embedding 是 RAG、语义搜索、推荐系统的基础**——把"文本相似度"问题变成"向量数学"问题，计算机才能批量处理。

> **生活化类比**：embedding 就像给每段文本发一个**GPS 坐标**——语义相近的文本坐标也近。"我喜欢吃苹果"和"我爱吃水果"在地图上挨着；"今天股市大跌"在地图另一头。你问"附近有什么吃的"，系统就在你的坐标周围找最近的几个点——这就是语义搜索。传统关键词搜索是"字面匹配"（搜"苹果"只能找到含"苹果"两字的），embedding 是"语义匹配"（搜"苹果"也能找到"水果"，因为它俩在地图上挨着）。

---

## 2. 第一次裸调 API

不用任何框架，手写 HTTP 调用，理解最底层的请求/响应结构。我们以 Anthropic Claude API 为例（Python SDK），概念在其他厂商（OpenAI 等）完全通用。

### 2.1 装包与鉴权

```bash
# 装官方 SDK
pip install anthropic

# 配置 API Key（不要写死在代码里！）
export ANTHROPIC_API_KEY="sk-ant-..."
```

> **安全提醒**：API Key 就是你的钱包，泄露了别人就能花你的钱。规则：① 永远放环境变量 / `.env`，不提交到 git；② `.gitignore` 里加 `.env`；③ 用 `os.environ["ANTHROPIC_API_KEY"]` 读，不要硬编码。Phase 3 部署时讲过，生产环境用 Vault / Secret Manager。

### 2.2 最简调用

```python
import anthropic

client = anthropic.Anthropic()  # 自动从环境变量读 key

response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "用一句话解释什么是闭包"}
    ],
)

# response.content 是一个 block 列表，最常见的是 text block
for block in response.content:
    if block.type == "text":
        print(block.text)
```

**逐行解释：**

- `model`：用哪个模型。`claude-opus-4-8` 是 Opus 系列最新款；练习时可以用 `claude-haiku-4-5`，便宜快很多。
- `max_tokens`：模型最多输出多少 token。**必填**，不填会报错。设小了答案会被截断，设大了不影响实际输出（只影响上限和计费上限）。
- `messages`：对话数组，第一条必须是 `user`。
- `response.content`：是一个**列表**，因为模型一次回复可能包含多个 block（文本块、工具调用块、思考块等）。要按 `block.type` 判断。

**响应里的其他有用字段：**

```python
print(response.stop_reason)   # "end_turn"=正常结束, "max_tokens"=被截断, "tool_use"=要调工具
print(response.usage.input_tokens)   # 输入 token 数（计费依据）
print(response.usage.output_tokens)  # 输出 token 数（计费依据）
```

### 2.3 多轮对话（API 是无状态的）

**关键认知：所有 LLM API 都是无状态的。** 服务器不记得你上一轮问了什么。要实现"多轮对话"，你必须在每次请求里把**完整历史**重新发一遍。

```python
class Chat:
    def __init__(self, system="你是一个有帮助的助手。"):
        self.client = anthropic.Anthropic()
        self.model = "claude-opus-4-8"
        self.system = system
        self.history = []   # 维护历史消息

    def ask(self, user_input: str) -> str:
        # 1. 把用户输入追加到历史
        self.history.append({"role": "user", "content": user_input})

        # 2. 把完整历史发给模型
        response = self.client.messages.create(
            model=self.model,
            max_tokens=1024,
            system=self.system,
            messages=self.history,   # ← 完整历史，每轮都全发
        )

        # 3. 取出回复
        assistant_text = next(
            (b.text for b in response.content if b.type == "text"), ""
        )

        # 4. 把回复也追加到历史（下一轮要带上）
        self.history.append({"role": "assistant", "content": assistant_text})
        return assistant_text


chat = Chat()
print(chat.ask("我叫 Sam。"))           # 你好 Sam！
print(chat.ask("我姓什么？"))            # 你姓...（模型能答对，因为历史里有）
print(chat.ask("给我讲个笑话。"))
```

**为什么这样做能"记住"：** 不是模型有记忆，而是你把之前的对话记录原样塞进了 prompt，模型只是"读"到了上下文。

> **生活化类比**：这就像你和学者每次见面都要**把之前所有的对话记录打印出来递给他**——他根本不记得你是谁，但他读完记录就能接上话。这就是为什么长对话会越来越贵——记录越来越长，每次都要全发，token 越来越多。Memory 那一节会讲怎么对付这个问题。

### 2.4 流式输出

把 `create` 换成 `stream`，逐 token 拿到输出：

```python
with client.messages.stream(
    model="claude-opus-4-8",
    max_tokens=1024,
    messages=[{"role": "user", "content": "写一首关于秋天的诗"}],
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)   # 逐字打印，打字机效果

# 流完后还能拿到完整消息对象（包含 usage 等）
final = stream.get_final_message()
print(f"\n用了 {final.usage.output_tokens} 个输出 token")
```

**前端联调要点：** 后端流式返回时，HTTP 响应是 `text/event-stream`（SSE）。前端用 `fetch` + `ReadableStream` 或 `EventSource` 接收，每收到一个 chunk 就 append 到页面上。下一阶段实战项目会完整实现这套。

> **生活化类比**：`stream` 就像**接水龙头**——你拿个桶（`stream` 上下文）对着龙头，水（token）一边流你一边接（`for text in stream.text_stream`）。接完之后桶里还能看到这桶水的总表（`get_final_message().usage`）。非流式是"等桶装满了一次性递给你"，流式是"一边接一边用"。

---

## 3. Function Calling — Agent 的基石

### 3.1 为什么需要 Function Calling

LLM 本身只能生成文本，不能"做事"：它不能查数据库、不能调 API、不能执行代码。如果你问"北京今天天气如何"，它要么编一个（幻觉），要么说"我不知道"。

**Function Calling（也叫 Tool Use）** 解决的就是这个：让模型**声明它想调哪个函数、参数是什么**，由你的代码真正去执行，然后把结果返回给模型，模型再基于结果生成最终回复。

```
没有 Function Calling：
  用户："北京天气如何？"  →  模型："北京今天晴，25℃"  （编的）

有 Function Calling：
  用户："北京天气如何？"
    →  模型："我想调 get_weather(city='北京')"   （只输出"调用意图"，不执行）
    →  你的代码：真正调天气 API，拿到 "晴 25℃"
    →  把结果塞回模型
    →  模型："北京今天晴，气温 25℃。"
```

**模型不执行任何代码，它只输出一个"我想调用什么"的 JSON。** 执行权永远在你手里。这是安全设计，也是 Agent 的核心机制。

> **生活化类比**：模型就像**坐在办公室的顾问**，他不能自己跑出去查资料，但他可以**填一张"请帮我查这个"的申请单**（tool_use）递给你。你拿着单子去查（执行函数），把结果写在单子背面递回给他（tool_result），他看完结果再给你写最终答复。这个"递单子—查—回单子—答复"的循环，就是 Agent 的工作方式。

### 3.2 工具调用的完整时序

```
1. 你发请求：messages=[用户问题]  +  tools=[工具定义...]
                    ↓
2. 模型返回：content=[{"type":"tool_use", "name":"get_weather",
                       "input":{"city":"北京"}, "id":"toolu_xxx"}]
   stop_reason="tool_use"   ← 关键信号：模型想调工具，没结束
                    ↓
3. 你的代码：解析 tool_use，真的去调 get_weather("北京")，拿到结果 "晴 25℃"
                    ↓
4. 你再发请求：messages=[
     原始用户问题,
     {"role":"assistant", "content": 模型上次的回复(含 tool_use block)},
     {"role":"user", "content": [
        {"type":"tool_result", "tool_use_id":"toolu_xxx", "content":"晴 25℃"}
     ]}
   ]
                    ↓
5. 模型基于结果返回最终文本回复，stop_reason="end_turn"
```

**两个高频踩坑点：**

1. **必须把模型上次的回复（含 tool_use block）原样回传**——模型要靠它知道自己"之前想调什么"。你不能只发 tool_result。
2. **tool_result 里的 `tool_use_id` 必须和 tool_use 里的 `id` 对上**——模型靠 ID 配对"哪个结果对应哪次调用"。多个工具同时调时尤其重要。

### 3.3 手写 Agent 循环

把上面的时序写成代码，就是一个最小的 Agent。**这一节是整个 Phase 4 的核心——做完它你才算真正理解了 Agent。**

```python
import anthropic
import json

client = anthropic.Anthropic()

# 1. 定义模型能用的工具（JSON Schema 描述参数）
tools = [
    {
        "name": "get_weather",
        "description": "获取指定城市的当前天气。当用户问天气时调用。",
        "input_schema": {
            "type": "object",
            "properties": {
                "city": {"type": "string", "description": "城市名，如 北京"},
            },
            "required": ["city"],
        },
    },
    {
        "name": "calculate",
        "description": "做四则运算。当用户问数学计算时调用。",
        "input_schema": {
            "type": "object",
            "properties": {
                "expression": {"type": "string", "description": "数学表达式，如 2+3*4"},
            },
            "required": ["expression"],
        },
    },
]


# 2. 真正执行工具的函数（你的代码，不是模型）
def execute_tool(name: str, args: dict) -> str:
    if name == "get_weather":
        # 真实场景里这里调天气 API，这里用假数据演示
        return f"{args['city']} 今天晴，25℃"
    if name == "calculate":
        # 注意：生产环境不要直接 eval，这里仅演示
        try:
            return str(eval(args["expression"]))
        except Exception as e:
            return f"计算失败: {e}"
    return f"未知工具: {name}"


# 3. Agent 循环
def run_agent(user_input: str):
    messages = [{"role": "user", "content": user_input}]

    while True:
        response = client.messages.create(
            model="claude-opus-4-8",
            max_tokens=1024,
            tools=tools,
            messages=messages,
        )

        # 把模型这次回复追加到历史（含 tool_use block）
        messages.append({"role": "assistant", "content": response.content})

        # 如果模型没要调工具，说明它给出了最终答复，结束
        if response.stop_reason != "tool_use":
            final_text = next(
                (b.text for b in response.content if b.type == "text"), ""
            )
            return final_text

        # 模型要调工具——可能有多个，逐个执行
        tool_results = []
        for block in response.content:
            if block.type == "tool_use":
                print(f"  → 调用工具 {block.name}({block.input})")
                result = execute_tool(block.name, block.input)
                print(f"  ← 结果: {result}")
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,   # ← ID 必须对上
                    "content": result,
                })

        # 把工具结果作为 user 消息发回去
        messages.append({"role": "user", "content": tool_results})


# 试一下
print(run_agent("北京天气怎么样？2 加 3 等于几？"))
```

**运行时会发生什么：**

```
  → 调用工具 get_weather({'city': '北京'})
  ← 结果: 北京 今天晴，25℃
  → 调用工具 calculate({'expression': '2+3'})
  ← 结果: 5
最终回复: 北京今天晴，气温 25℃。2 加 3 等于 5。
```

模型可能**一轮调用多个工具**，也可能**调完一个工具拿到结果后，再决定调下一个**（比如先查天气，发现下雨，再调"发提醒邮件"的工具）——这就是 **Agent 循环**：思考 → 调工具 → 拿结果 → 再思考 → 再调工具 → ... → 最终回复。

> **做完这个，你才真正理解了 Agent。** 后面所有 Agent 框架（LangChain、Vercel AI SDK、OpenAI Agents SDK）封装的都是这个循环，只是加了日志、重试、并发、流式等工程化能力。框架是脚手架，这个循环是地基。

### 3.4 并行工具调用

模型一次回复里可以包含**多个 tool_use block**（并行调用）。正确做法：把它们**全部执行完，结果放在同一个 user 消息里**一次性返回，不要拆成多条消息。

```python
# ✅ 正确：所有 tool_result 放在一个 user 消息里
tool_results = []
for block in response.content:  # 可能有多个 tool_use
    if block.type == "tool_use":
        result = execute_tool(block.name, block.input)
        tool_results.append({
            "type": "tool_result",
            "tool_use_id": block.id,
            "content": result,
        })
messages.append({"role": "user", "content": tool_results})

# ❌ 错误：每个 tool_result 单独一条 user 消息
# 会让模型学到"不要并行调用"，渐渐退化
```

**失败的工具也要返回结果**——把 `is_error: true` 设上，告诉模型这次调用失败了，它会换个思路。不要直接吞掉失败的工具调用。

```python
tool_results.append({
    "type": "tool_result",
    "tool_use_id": block.id,
    "content": "调用失败：网络超时",
    "is_error": True,
})
```

---

## 4. RAG — 让模型用你的私有数据

### 4.1 为什么需要 RAG

LLM 的知识来自训练数据，有三个硬伤：

1. **不知道你的私有数据**——你公司的内部文档、数据库、知识库，模型见都没见过
2. **训练有截止日期**——最近发生的事它不知道
3. **会幻觉**——你问"我们公司的请假政策"，它可能编一个听起来很合理的

**RAG (Retrieval-Augmented Generation，检索增强生成)** 的思路很朴素：

> 既然模型不知道你的数据，那就在问模型之前，**先从你的数据里检索出和问题最相关的几段，塞进 prompt 里**，让模型基于这几段回答。

```
用户："公司年假怎么算？"
      ↓
[检索] 从公司文档库里找出"年假政策.pdf"里最相关的一段："入职满 1 年 5 天，满 3 年 10 天..."
      ↓
[增强] 拼 prompt：
   system: "基于以下资料回答用户问题。资料里没有就说不知道。"
   context: [检索到的那段]
   user: "公司年假怎么算？"
      ↓
[生成] 模型："根据公司政策，入职满 1 年有 5 天年假，满 3 年 10 天。"
```

**RAG 的价值：**

- 模型基于**真实资料**回答，幻觉大幅减少
- 资料可以随时更新（不用重新训练模型）
- 可以溯源——回答来自哪段文档，能给出处

> **生活化类比**：RAG 就是**开卷考试**。闭卷考试（直接问 LLM）学生可能记错、编造；开卷考试（RAG）你把相关页码翻出来摊在他面前，让他照着答，准确率自然高。检索做得好不好，等于"翻页翻得准不准"——翻错页（找错资料）答得再认真也是错的。所以 RAG 的难点不在"生成"，在"检索"。

### 4.2 向量检索的原理

怎么从一大堆文档里"找出和问题最相关的那几段"？关键词搜索不行——用户问"年假"，文档里写的是"带薪休假"，字面不匹配就搜不到。

**向量检索**用 Embedding 解决：

1. **离线阶段**：把文档库每一段都跑一遍 Embedding，得到每段的向量，存进**向量数据库**
2. **查询阶段**：把用户问题也跑一遍 Embedding，得到问题向量
3. **检索**：在向量数据库里找"和问题向量最相近"的几个文档向量——余弦相似度最大的前 K 个
4. 把对应的文档原文捞出来，塞进 prompt

```
文档库                    Embedding              向量数据库
─────────                ────────              ─────────
"年假 5 天..."    ──→    [0.12, ...]    ──→   存进去
"报销流程..."     ──→    [-0.3, ...]    ──→   存进去
"入职手续..."     ──→    [0.45, ...]    ──→   存进去

用户："年假多少？"  ──→   [0.11, ...]   ──→  搜相似  ──→  命中"年假 5 天..."
```

### 4.3 文档分块（chunking）

文档太长，不能整篇做一个向量（向量代表不了那么多元信息，也塞不进 prompt）。要把文档切成一块一块的（chunk），每块单独做 Embedding。

**分块策略：**

| 策略 | 做法 | 适用 |
|------|------|------|
| 固定长度 | 每 N 个字符切一刀 | 简单，可能切断语义 |
| 按段落/标题 | 按 `\n\n` 或 markdown 标题切 | 语义完整，推荐 |
| 递归分块 | 先按大标志切，超长再按小标志切 | 通用，最常用 |
| 重叠分块 | 相邻块有 N 字符重叠 | 避免边界信息丢失 |

**经验值：**

- 块大小 200-500 token（中文 300-800 字）比较常用
- 块太小：上下文不够，答得不全；块太大：一个向量代表太多内容，检索不准，还费 token
- **块的质量比块的大小更重要**——保持语义完整（不要把一句话切成两半）

### 4.4 从零搭一个最小 RAG

用 **ChromaDB**（轻量向量数据库，纯 Python，免部署，自带 Embedding 模型）做一个最简 RAG：

```bash
pip install chromadb anthropic
```

```python
import chromadb
import anthropic

client = anthropic.Anthropic()

# 1. 准备文档库（实际场景里从 PDF/Word/数据库读）
documents = [
    "公司年假政策：入职满 1 年享有 5 天年假，满 3 年 10 天，满 5 年 15 天。",
    "报销流程：员工填写报销单，附发票原件，交部门主管签字后送财务部。",
    "入职手续：新员工需提交身份证复印件、学历证明、银行卡号，到 HR 处办理。",
    "请假需提前在 OA 系统提交申请，3 天以内由直属主管审批，超过 3 天需部门经理审批。",
]

# 2. 建向量库，把文档灌进去
chroma = chromadb.Client()
collection = chroma.create_collection("company_docs")
collection.add(
    documents=documents,
    ids=[f"doc_{i}" for i in range(len(documents))],
    # Chroma 默认用 sentence-transformers 的 all-MiniLM-L6-v2 做 embedding，本地跑，免费
)
# 实际生产用 OpenAI/Voyage 等商业 embedding API 更准，这里用本地的够学习用


# 3. 检索 + 生成
def ask(question: str) -> str:
    # 3.1 检索：拿问题去向量库找最相关的 2 段
    results = collection.query(query_texts=[question], n_results=2)
    context = "\n".join(results["documents"][0])

    # 3.2 增强：把检索到的资料塞进 prompt
    response = client.messages.create(
        model="claude-opus-4-8",
        max_tokens=512,
        system=(
            "你是公司内部助手。只根据下面提供的资料回答用户问题。"
            "如果资料里没有相关信息，明确说'资料里没有这个信息'，不要编造。\n\n"
            f"【资料】\n{context}"
        ),
        messages=[{"role": "user", "content": question}],
    )
    return next(b.text for b in response.content if b.type == "text")


print(ask("年假怎么算？"))        # 基于资料答：满 1 年 5 天...
print(ask("报销要找谁签字？"))     # 基于资料答：部门主管...
print(ask("公司有健身房吗？"))     # 资料里没有这个信息
```

**这就是 RAG 的最小骨架。** 真实项目里还要做：更好的分块、更好的 Embedding 模型、重排序（rerank）、引用溯源、增量更新、多路召回（关键词 + 向量）等。但骨架就是检索→增强→生成这三步。

---

## 5. Memory — 多轮对话怎么记

### 5.1 短期记忆：历史消息数组

最简单的记忆方式就是 2.3 节那种——**把历史消息原样存在数组里，每次请求全发过去**。这种方式叫"短期记忆"或"对话窗口"。

优点：简单、模型能完整看到上下文。缺点：**历史越长，token 越多，越贵越慢，最终撑爆 context window**。

### 5.2 上下文窗口溢出

假设 context window 是 200K token，对话每轮平均 1000 token，聊 200 轮就满了。满了之后：

- 新消息塞不进去，API 报错
- 或者老消息被截断，模型"失忆"

**怎么知道快满了？** 调 `count_tokens` 接口估算，或监控 `response.usage.input_tokens`。

### 5.3 摘要记忆与向量记忆

长对话不能无限堆历史，有三种常见压缩策略：

| 策略 | 做法 | 优点 | 缺点 |
|------|------|------|------|
| **滑窗截断** | 只保留最近 N 轮 | 简单 | 早期信息全丢 |
| **摘要记忆** | 定期让模型把旧历史总结成一段摘要，替换原始消息 | 保留要点 | 摘要会丢细节 |
| **向量记忆** | 把每轮对话存进向量库，每轮按当前问题检索相关历史塞进 prompt | 长期保留 + 按需取 | 实现复杂 |

**实践推荐：** 滑窗 + 摘要组合——保留最近 5-10 轮原文，更早的定期压缩成摘要。这也是 LangChain `ConversationSummaryBufferMemory` 的思路。

> **生活化类比**：记忆管理就像**整理书桌**。书桌（context window）就那么大。每轮对话像往桌上放一张纸条——聊得越久纸条越多，桌面放不下。滑窗就是"把最早的纸条扔掉"——简单但丢信息。摘要就是"把一摞旧纸条总结成一页摘要，然后把原件收起来"——保留要点。向量记忆就是"把旧纸条编进目录放抽屉，需要时按主题翻出来"——最聪明但要建索引。

### 5.4 用 Claude 的 compaction（生产级方案）

Claude 4.6+ 提供了 **Compaction（上下文压缩）** 能力（beta）：当对话接近 context window 时，API 会**自动把早期历史总结成摘要块**，你在后续请求里把摘要块回传即可，不用自己写摘要逻辑。

```python
response = client.beta.messages.create(
    model="claude-opus-4-8",
    max_tokens=1024,
    betas=["compact-2026-01-12"],
    context_management={"edits": [{"type": "compact_20260112"}]},
    messages=messages,
)
# 关键：把 response.content 原样追加到 messages（compaction 块要保留，不能只取文本）
messages.append({"role": "assistant", "content": response.content})
```

这是生产级长对话的省心方案，不用自己造轮子。

---

## 6. Agent 框架与生态

裸写 Agent 循环能让你理解原理，但生产项目里手写循环要处理太多工程细节：重试、超时、并发、流式、工具校验、日志、错误恢复。框架把这些封装好了。**先理解原理，再用框架——否则框架就是黑盒。**

### 6.1 框架对比

| 框架 | 语言 | 特点 | 建议 |
|------|------|------|------|
| **Vercel AI SDK** | TypeScript | 前端生态最友好，多模型，流式开箱即用 | 主力 Node.js 的话首选 |
| **LangChain** | Python/JS | 最成熟、生态最大、抽象层多 | 必知，但别盲目套它的重抽象 |
| **LangGraph** | Python | LangChain 团队出的图式编排，状态机 | 做复杂多步工作流时学 |
| **OpenAI Agents SDK** | Python | 官方轻量框架，简洁 | 入门友好 |
| **Anthropic MCP** | 协议 | 不是框架，是 Agent 与工具的标准化协议 | 生态标准，必须了解 |

### 6.2 Vercel AI SDK（前端友好）

对前端转过来的同学，Vercel AI SDK 几乎零学习成本——TypeScript、和 Next.js 无缝集成、流式渲染封装得极好。

```bash
npm install ai @ai-sdk/anthropic
```

```typescript
import { streamText } from "ai";
import { anthropic } from "@ai-sdk/anthropic";

// 流式生成
const result = streamText({
  model: anthropic("claude-opus-4-8"),
  system: "你是一个简洁的编程助手。",
  messages: [{ role: "user", content: "用一句话解释闭包" }],
});

// 逐 chunk 输出，可以直接 pipe 到前端 Response
for await (const chunk of result.textStream) {
  process.stdout.write(chunk);
}
```

带工具调用的版本（SDK 自动跑循环）：

```typescript
import { generateText, tool } from "ai";
import { anthropic } from "@ai-sdk/anthropic";
import { z } from "zod";

const result = await generateText({
  model: anthropic("claude-opus-4-8"),
  messages: [{ role: "user", content: "北京天气怎么样？" }],
  tools: {
    get_weather: tool({
      description: "获取城市天气",
      parameters: z.object({ city: z.string() }),
      execute: async ({ city }) => `${city} 晴 25℃`,
    }),
  },
});

console.log(result.text);   // SDK 自动跑了"调工具→拿结果→再生成"的循环
```

### 6.3 LangChain / LangGraph

LangChain 的价值在**生态集成**——它预先接好了几百个工具（PDF 加载器、各种数据库、搜索引擎、向量库），拼装起来快。代价是抽象层又厚又容易变，升级常常 break。

**用它的集成，别全盘套它的 Agent 抽象**——很多团队最后都把 Agent 循环换回自己写的，只留它的 loader/retriever。

```python
from langchain_anthropic import ChatAnthropic
from langchain_core.tools import tool

@tool
def get_weather(city: str) -> str:
    """获取城市天气"""
    return f"{city} 晴 25℃"

llm = ChatAnthropic(model="claude-opus-4-8")
llm_with_tools = llm.bind_tools([get_weather])
# LangChain 会帮你跑工具循环
```

**LangGraph** 在此之上提供**图式编排**——把 Agent 工作流定义成节点和边的状态机，适合"先规划→执行→检查→重试"这种复杂流程。

### 6.4 MCP（Model Context Protocol）

**MCP 是 Anthropic 提出的开放协议，让 Agent 通过统一接口连接各种工具和数据源。** 类比：MCP 之于 Agent = USB 之于外设。

**没有 MCP 之前：** 每个 Agent 框架、每个模型厂商，工具接入方式都不一样。你给 LangChain 写的工具，换到 OpenAI Agents SDK 要重写；你给 Claude 写的工具，换到 GPT 也要改。

**有了 MCP：** 你写一个 MCP Server（暴露一组工具/资源），任何实现了 MCP Client 的 Agent（Claude Desktop、Cursor、Cline、各种 IDE）都能用同一套工具，不用改。

```
┌──────────────┐        ┌──────────────┐
│  MCP Client  │◄──────►│  MCP Server  │ ← 你写一次
│ (Agent 端)   │  协议   │ (提供工具)    │
└──────────────┘        └──────────────┘
   Claude Desktop           数据库/文件/API
   Cursor                   一旦写好，所有 Client 都能用
   Cline / IDE
```

**MCP 的核心概念：**

- **MCP Server**：提供工具（tools）、资源（resources）、提示模板（prompts）的服务端
- **MCP Client**：Agent 端，发现并调用 MCP Server 提供的能力
- **传输**：本地用 stdio，远程用 SSE/HTTP

写一个最简 MCP Server（Python）：

```bash
pip install mcp
```

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("my-tools")

@mcp.tool()
def get_weather(city: str) -> str:
    """获取城市天气"""
    return f"{city} 晴 25℃"

@mcp.tool()
def calculate(expression: str) -> str:
    """做四则运算"""
    return str(eval(expression))   # 仅演示，生产别用 eval

if __name__ == "__main__":
    mcp.run()   # 启动 MCP Server，等 Agent 来连
```

写完这个 Server，Claude Desktop、Cursor 都能配置后直接用你的工具——**一次编写，处处可用**，这就是 MCP 的价值。

> **生活化类比**：MCP 之前，每个 Agent 和每个工具之间都要**焊死一根专用线**——N 个 Agent × M 个工具 = N×M 根线，维护地狱。MCP 定义了一个**标准插头**——工具做成 MCP Server（一头是标准插头），Agent 做成 MCP Client（另一头也是标准插头），N+M 根线搞定。USB 之前外设接口五花八门（PS/2、串口、并口），USB 一统天下后随便插——MCP 就是 Agent 世界的 USB。

---

## 7. Agent 进阶方向

掌握基础后，这些是进阶方向，每个都能单开一个坑：

| 方向 | 说明 | 关键词 |
|------|------|--------|
| **多 Agent 协作** | 多个 Agent 各司其职，一个规划、一个执行、一个审查 | multi-agent, supervisor, orchestrator |
| **ReAct / Plan-and-Execute** | Agent 推理模式：ReAct 边想边做，Plan-and-Execute 先规划再执行 | ReAct, Plan-and-Execute, Chain-of-Thought |
| **Evaluation & Observability** | 评估 Agent 输出质量、追踪每一步、监控用量成本 | LangSmith, LangFuse, traces |
| **Human-in-the-loop** | 关键操作（发邮件、删数据）要人工确认，Agent 不能无限自主 | approval gate, interrupt |
| **Prompt Caching** | 把稳定前缀缓存，重复请求大幅降本降延迟 | cache_control, ephemeral breakpoint |
| **Structured Output** | 强制模型输出合法 JSON / 特定 schema，下游能解析 | output_config.format, strict tool use |
| **长上下文管理** | compaction、context editing、记忆系统 | compact, context management, memory |

**学习的优先级建议：**

1. 先把第 3 节的 Agent 循环和第 4 节的 RAG **彻底搞懂**（这是地基）
2. 学 MCP（生态趋势，必学）
3. 做一个小项目把 RAG + Function Calling + 流式串起来（Phase 5）
4. 再看 Evaluation / Observability（项目要上线时必须）
5. 多 Agent / Plan-and-Execute 等模式按需学

---

## 附录：常用模型速查

| 模型 | ID | Context | 适合 | 备注 |
|------|-----|---------|------|------|
| Claude Opus 4.8 | `claude-opus-4-8` | 1M | 最难的任务、长 horizon Agent | 推荐 |
| Claude Sonnet 4.6 | `claude-sonnet-4-6` | 1M | 速度/智能平衡，生产主力 | 性价比高 |
| Claude Haiku 4.5 | `claude-haiku-4-5` | 200K | 简单分类、抽取、省钱 | 练习用这个 |
| GPT-4o | `gpt-4o` | 128K | OpenAI 生态 | 多模态强 |
| text-embedding-3-small | OpenAI | — | Embedding | 便宜够用 |

**几点注意（Claude 4.6+ 系列）：**

- 已移除 `temperature`/`top_p`/`top_k`，传入会 400——用 prompting 或 `effort` 替代
- 不支持 assistant 消息 prefill（在最后一条 assistant 消息里预填文本），要 JSON 输出就用 `output_config.format`
- 思考用 `thinking: {type: "adaptive"}`，不要用已废弃的 `budget_tokens`
- 支持 **MCP connector**、**Tool Search**、**Code Execution** 等服务端工具，详见官方文档

---

## 检验清单

在不看笔记的情况下能解释清楚并动手实现：

- [ ] LLM 本质上在做什么？为什么会"幻觉"？
- [ ] token 是什么？为什么不能用 `len(string)` 估算 token 数？
- [ ] context window 是什么？标称 1M 等于实际能用 1M 吗？
- [ ] system / user / assistant 三种 role 各起什么作用？
- [ ] API 是有状态还是无状态？多轮对话怎么实现"记忆"？
- [ ] streaming 为什么比非流式体验好？技术上靠什么实现？
- [ ] embedding 是什么？余弦相似度衡量的是什么？
- [ ] Function Calling 的完整时序能画出来吗？为什么说"模型不执行代码"？
- [ ] `tool_use_id` 为什么必须对上？并行工具调用的结果应该怎么发回？
- [ ] 能手写一个带工具调用的 Agent 循环（不靠任何框架）吗？
- [ ] RAG 解决什么问题？检索-增强-生成三步各做什么？
- [ ] 为什么要做文档分块？块太大和太小各有什么问题？
- [ ] 长对话上下文溢出怎么处理？滑窗/摘要/向量记忆各有什么取舍？
- [ ] Vercel AI SDK / LangChain / LangGraph / MCP 各解决什么问题？
- [ ] MCP 之前 Agent 接工具的痛点是什么？MCP 怎么解决的？
- [ ] 从零搭一个最小 RAG：文档入库 → 向量检索 → 拼 prompt → 生成，跑通

---

> **下一步**：理解原理后，进入 [Phase 5：实战 — 从零交付一个 Agent 应用](../README.md#phase-5实战--从零交付一个-agent-应用) —— 把 RAG、Function Calling、流式、前端、部署全部串起来，交付一个能演示的智能文档问答应用。
