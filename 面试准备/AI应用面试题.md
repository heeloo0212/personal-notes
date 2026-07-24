---
title: AI 应用面试题（深度版）
tags: [面试, AI, LLM, RAG, Agent, 深度]
created: 2026-07-19
---

# AI 应用面试题（深度版）

> 每题包含：核心原理 → 关键细节/数学 → 常见追问 → 项目映射/代码。面试官追问时可逐层展开。

---

## 一、Transformer 与 LLM 基础

### 1. Transformer 全结构 + 注意力数学

**自注意力（Scaled Dot-Product）**

```
Attention(Q,K,V) = softmax(QKᵀ / √d_k) · V
```

- `√d_k` 缩放：点积方差随维度增长，缩放防止 softmax 进入饱和区（梯度消失）。
- Q/K/V：输入 X 乘三个权重矩阵得到。
- 多头：把 d 拆成 h 个子空间并行做 attention，再 concat 投影回 d，捕捉不同子空间的语义。

**Encoder vs Decoder**

- Encoder（BERT）：双向注意力，看到上下文，适合理解任务（分类/抽取）。
- Decoder（GPT）：自回归（causal mask），只看左侧，适合生成。
- Encoder-Decoder（T5/Whisper）：交叉注意力把 encoder 输出喂给 decoder。

**位置编码**

- 正弦：`sin(pos/10000^(2i/d))`，可外推。
- RoPE（LLaMA/Qwen）：用旋转矩阵编码相对位置，乘法实现，外推性好。
- ALiBi：直接在 attention 加距离偏置，无需位置编码，长序列表现稳。

**LayerNorm vs RMSNorm**

- LayerNorm：减均值再除标准差。
- RMSNorm：只除 RMS（root mean square），省去均值计算，LLaMA/Qwen2 用。
- Pre-Norm（残差在外）：训练稳定，主流大模型用。

**FFN**

- 标准：`Linear → GELU → Linear`。
- SwiGLU（LLaMA）：`Swish(xW) * (xV)`，门控激活，效果更好但参数多。
- MoE：FFN 拆多个 expert，路由器 top-k 选择，稀疏激活降 FLOPs（如 Qwen2-MoE、Mixtral）。

**追问**

- Q：为什么用 softmax 而不是其他？ A：可微、归一化、放大高分。
- Q：注意力复杂度？ A：O(n²d)，长序列瓶颈；优化如 FlashAttention、线性注意力、滑动窗口。

---

### 2. FlashAttention 原理 + KV 缓存

**FlashAttention**

- 标准 attention 中间矩阵 `n×n` 放 HBM，读写开销大。
- FlashAttention 分块（tiling）+ 重计算，在 SRAM 内完成 softmax，避免 n² 矩阵落地，IO-aware。
- 速度提升 2-4x，显存省（不开销 O(n²) 中间态），数值精确（不减精度）。

**KV 缓存**

- 自回归生成时，每生成一个 token 都重算历史 K/V 浪费；缓存已算的 K/V。
- KV 缓存显存：`2 × layers × n_heads × head_dim × seq_len × batch × dtype_bytes`。
- 长上下文 + 大 batch 时 KV 缓存是显存大头 → PagedAttention 优化。

**PagedAttention（vLLM）**

- 把 KV 缓存按页（block）管理，类似 OS 虚拟内存分页。
- 物理块按需分配，逻辑地址连续，避免内部碎片 + 浪费在 padding 上的显存。
- 支持「共享前缀」：多请求共用同一 system prompt 的 KV 页（Copy-on-Write）。

**追问**

- Q：FlashAttention 为什么不省算力？ A：它优化 IO，不是 FLOPs；理论 FLOPs 不变。
- Q：MQA/GQA 是什么？ A：多查询注意力（所有 head 共享 K/V），GQA 分组共享，降 KV 缓存、提吞吐。

---

### 3. vLLM 部署 + Continuous Batching

**Continuous Batching**

- 传统 batch：必须等一整批完成才能下一批，长尾拖慢整体。
- 连续批处理：请求完成后立即换入新请求，迭代级别动态拼 batch，GPU 利用率高。
- 配合 PagedAttention，新请求无缝接入。

**张量并行 + 流水并行**

- TP：矩阵按行/列切分到多卡，每卡算部分，all-reduce 汇总。
- PP：模型按层切到多卡，micro-batch 流水推进。
- 大模型推理 7B/13B 单卡 A100/H100，70B+ 需 TP=4 或 PP。

**量化部署**

- AWQ：激活感知权重量化，保留重要权重 FP16，4bit 几乎不掉点。
- GPTQ：基于二阶 Hessian 信息量化，速度快。
- INT8/INT4 推理用 vLLM/AutoGPTQ/TensorRT-LLM。

**追问**

- Q：vLLM 为什么吞吐高？ A：PagedAttention 省显存 → 更大 batch → 更高吞吐；连续 batch 避免 GPU 空转。
- Q：3B 量化后多少显存？ A：FP16 ~6GB；INT8 ~3GB；INT4 ~1.8GB（加 KV + 激活留 2x 余量）。

---

### 4. Qwen-VL 多模态架构 + 金融风控落地

**架构**

- 视觉编码器：ViT（Vision Transformer），把图片切成 patch 序列。
- 视觉-语言对齐 Adapter：MLP/交叉注意力，把视觉特征对齐到 LLM 词向量空间。
- LLM 主干：Qwen LLM 处理「视觉 token + 文本 token」混合序列，自回归生成。

**Qwen-VL 特点**

- 支持任意宽高比、多图片输入。
- 可定位（输出 bbox 坐标），OCR 能力强。
- Qwen2-VL 用动态分辨率 + M-RoPE（多模态旋转位置编码）。

**金融风控落地**

1. 身份认证：身份证 OCR → 提取姓名/身份证号/地址 → 与申请数据比对。
2. 银行流水：截图识别金额/日期/收支 → 汇总月收入、异常大额检测。
3. 工作证明：图片提取公司/职位/薪资 → 与企业公开信息核验。

**Prompt 工程**

```python
prompt = f"""你是金融风控审核员。从下面图片中提取字段，按 JSON 输出，未识别填 null。
字段：name, id_card, address, issue_date
只输出 JSON，不要其他文字。
"""
```

- 加 JSON Schema 校验 + 失败重试（Temperature=0 + 修复提示）。
- Few-shot：给 2-3 个样例稳定输出格式。
- 长文本截断：流水页数多时分页提取再聚合。

**追问**

- Q：OCR 容错怎么做？ A：置信度阈值 + 关键字段正则校验（身份证 18 位校验位）+ 与库表比对一致性。
- Q：图片分辨率太高怎么办？ A：切片推理（SAHI）或动态分辨率，避免 token 爆炸。

---

### 5. Whisper 架构 + 长音频处理

**架构**

- Encoder-Decoder Transformer。
- 输入：mel 频谱图（80 维 mel + 30s 窗口）。
- 多任务：语音识别（原语）、翻译（→英文）、语言识别、时间戳、VAD。
- 训练：680k 小时多语言弱监督。

**长音频处理**

- 30s 窗口限制 → VAD（Silero VAD）先切静音，再按窗口切。
- 边界重叠：相邻片段重叠 1-2s 防止截断词。
- 拼接：按时间戳合并，处理重复词。
- 后处理：GPT 纠错（标点、专有名词、专业术语）。

**金融风控应用**

- 电话核身录音转文字 → 喂 LLM 判断话术异常（情绪、引导词、矛盾点）。

**追问**

- Q：Whisper 中文方言效果？ A：普通话好，方言差，需 fine-tune 或换 FunASR/Paraformer。
- Q：流式识别？ A：Whisper 非流式；流式用 FunASR online、SenseVoice。

---

## 二、Agent 与 Multi-Agent

### 6. Agent 定义 + ReAct vs Plan-and-Execute vs ReWOO

**Agent = LLM（推理） + 工具调用 + 记忆 + 反思**

- 输入感知 → LLM 推理决策 → 调用工具 → 观察结果 → 继续推理 → ... → 输出。

**ReAct**

```
Thought: 需要查 A 公司资质
Action: search_company["A 公司"]
Observation: 成立于 2018，注册资本 1000 万
Thought: 还需核对法人
Action: ...
```

- 每步推理 + 行动交替，灵活但易跑偏，Token 多。

**Plan-and-Execute**

1. Planner 一次性生成计划（步骤列表）。
2. Executor 逐步执行（每步可内部 ReAct）。
3. 完成后 Re-Plan（必要时调整）。

- 优势：可控、省 Token；劣势：初期规划不准需重规划。

**ReWOO**

- 一次性生成所有子任务 + 依赖关系，并行/串行执行。
- 减少 LLM 中间介入次数。

**选型**

- 探索性、不确定任务 → ReAct。
- 流程固定（风控）→ Plan-and-Execute。
- 大量独立子任务 → ReWOO 并行。

**追问**

- Q：怎么衡量 Agent 成本？ A：总 Token 数 × 单价 + 工具调用次数 + 总延迟；优化靠小模型路由、缓存、并行。

---

### 7. Multi-Agent 协同模式 + 风控平台架构

**协同模式**

| 模式 | 描述 |
|---|---|
| Manager-Worker | 中心调度分发任务 |
| Pipeline | 流水线顺序处理 |
| Debate | 多 Agent 辩论后投票 |
| Group Chat | 多 Agent 群聊，主持人路由 |
| Hierarchical | 多层级联 |

**风控平台架构**

```
            ┌─── Manager Agent ────┐
            │  (调度 + 仲裁 + 聚合)  │
            └──────────┬───────────┘
        ┌─────────────┼─────────────┐
        ▼             ▼             ▼
  身份认证 Agent   银行流水 Agent   工作证明 Agent
  (OCR + 校验)    (金额汇总 + 异常)  (企业核验)
        │             │             │
        └──── Google Search MCP（企业资质）────┘
```

**协同要素**

- 消息总线：Agent 间通过消息通信（`AgentMessage {from, to, content, type}`）。
- 共享状态：BlackBoard / 共享 Memory（材料特征 + 中间结论）。
- 任务路由：Manager 根据材料类型 + Agent 能力路由。
- 冲突仲裁：多 Agent 结论冲突 → 置信度加权 + 规则裁决。
- 结果聚合：Manager 汇总各 Agent 子结论 + 风险依据，输出终审建议。

**输出可解释**

- 每条结论附引用（材料片段 ID + Agent 名），便于人工复核 + 监管审计。

---

### 8. Agent 记忆机制 + 长期记忆

**短期**

- 对话上下文窗口（直接拼 prompt）。
- Scratchpad：工具调用历史记录区。
- 滑动窗口 / 摘要压缩：超长时 summarize 旧对话再拼。

**长期**

- 向量库：embedding + 检索历史对话/用户偏好。
- 知识图谱：实体关系结构化，支持多跳推理。
- 摘要式：定期 rollup，存关键事实而非全量。

**LangChain Memory**

```python
from langchain.memory import (
    ConversationBufferMemory,       # 全量
    ConversationSummaryMemory,      # 摘要
    ConversationSummaryBufferMemory,# 摘要 + 最近 N 条
    ConversationKGMemory,           # 知识图谱
    VectorStoreRetrieverMemory,      # 向量检索
)
```

**追问**

- Q：上下文窗口 200k 还需要长期记忆吗？ A：需要，成本/延迟 + 跨会话 + 海量历史，向量检索更经济。

---

### 9. 防止 Agent 死循环 + 工具调用安全

**循环防护**

- `max_iterations` 总步数上限。
- 单步 `timeout`。
- 重复 Action 检测：连续 N 次相同 Action + 相同参数 → 终止。
- Token 总量上限。
- 工具调用失败 N 次 → 降级（人工 / 默认回复）。

**工具安全**

- 危险操作（写库/转账）需人工确认（Human-in-the-loop）。
- 输入校验 + 沙箱（执行代码用容器）。
- 权限隔离：Agent 只能调授权工具集。
- 审计日志：每次工具调用记录入参/出参/Agent/时间。

**追问**

- Q：Agent 误调删除工具怎么办？ A：白名单 + 二次确认 + 软删除 + 操作日志可回滚。

---

### 10. Function Calling vs MCP vs 自研协议

**Function Calling**

- 模型原生能力：模型输出工具调用结构化 JSON。
- OpenAI/Qwen/DeepSeek 都支持，schema 不同。
- 模型决定调哪个工具、参数。

**MCP**

- Anthropic 开放协议，标准化「模型 ↔ 工具/资源/Prompt 模板」连接。
- Server 暴露 `tools/list`、`resources/list`、`prompts/list`、`tools/call`。
- Client（Claude Desktop、Cursor、自研 Agent）通过 MCP 调用。
- 优势：一个 Server 多 Client 复用，解耦模型与工具实现。

**对比**

| 维度 | Function Calling | MCP |
|---|---|---|
| 层级 | 模型能力 | 传输/发现协议 |
| 复用 | 每个应用重新接 | 一次实现处处调用 |
| 标准 | 各家不同 | 开放统一 |
| 关系 | 可叠加 | MCP 包装工具，模型用 FC 决策 |

---

## 三、MCP 实战

### 11. MCP 协议 + Server 开发

**协议**

- 传输：stdio（本地）、HTTP+SSE（远程）。
- 三类能力：
  - `tools`：模型可调用的函数。
  - `resources`：模型可读的数据/文件。
  - `prompts`：预设的 prompt 模板。
- 生命周期：`initialize` → `initialized` → 交互 → `shutdown`。

**Python SDK 开发**

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("geo-tools")

@mcp.tool()
def geocode(address: str) -> dict:
    """地理编码：地址 -> 坐标"""
    return {"lng": 114.05, "lat": 22.55}

@mcp.tool()
def poi_search(keyword: str, city: str) -> list:
    """POI 检索"""
    return [{"name": "...", "addr": "..."}]

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

**Client 配置**（claude_desktop_config.json）

```json
{
  "mcpServers": {
    "geo": {
      "command": "python",
      "args": ["/path/to/geo_server.py"]
    }
  }
}
```

---

### 12. 高德 MCP + Playwright MCP 在爬虫平台应用

**高德 MCP（地理编码）**

- 已有 Server 或自封装：调高德开放 API。
- 用途：人物地址补全、POI 检索、关系图谱地点维度。

**Playwright MCP（浏览器自动化）**

- MCP 包装 Playwright，暴露 `navigate`、`click`、`fill`、`screenshot`、`evaluate`、`extract` 等工具。
- 用途：JS 渲染页面、登录态、点击翻页、表单提交、反爬对抗。

**爬虫流程（Agent 驱动）**

1. 接收查询目标 → Agent 规划检索策略。
2. 调 Playwright MCP 打开搜索引擎 → 输入查询 → 截图 + 提取结果。
3. 对结果 URL 逐个调 Playwright MCP 抓取详情。
4. 调高德 MCP 补地点信息。
5. 提取人物关系 → 写入图数据库。
6. 失败重试 + 代理切换 + 限速。

**追问**

- Q：Playwright vs Selenium？ A：Playwright 现代化、自动等待、多浏览器、API 简洁；Selenium 生态老但成熟。
- Q：反爬怎么应对？ A：指纹随机化、代理池、限速、真人化操作（鼠标移动）、打码平台兜底。

---

## 四、LangChain / LCEL / FastAPI

### 13. LCEL（LangChain Expression Language）+ 流式

**LCEL 管道**

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

chain = (
    ChatPromptTemplate.from_template("用一句话解释：{topic}")
    | llm
    | StrOutputParser()
)

# 流式
for chunk in chain.stream({"topic": "RAG"}):
    print(chunk, end="", flush=True)

# 异步 + 批处理
await chain.ainvoke({"topic": "RAG"})
chain.batch([{"topic": "a"}, {"topic": "b"}])
```

**LCEL 优势**

- 流式、异步、批处理开箱即用。
- 可组合 `RunnablePassthrough`、`RunnableParallel`、`RunnableLambda`。
- 统一接口，便于观测（LangSmith trace）。

**RunnablePassthrough + 并行检索**

```python
chain = (
    {"context": retriever, "question": RunnablePassthrough()}
    | prompt
    | llm
    | parser
)
```

---

### 14. FastAPI 流式 + SSE + 后台任务

**SSE 流式**

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

app = FastAPI()

@app.post("/chat")
async def chat(req: ChatReq):
    async def gen():
        async for chunk in llm.astream(req.prompt):
            yield f"data: {chunk.content}\n\n"
    return StreamingResponse(gen(), media_type="text/event-stream")
```

- 前端用 `EventSource` 接收，自动重连。
- 比 WebSocket 轻量，单向（服务器推）。

**后台任务**

```python
@app.post("/crawl")
async def crawl(req: CrawlReq, bg: BackgroundTasks):
    job_id = uuid4()
    bg.add_task(run_crawl, job_id, req)
    return {"job_id": job_id}
```

- 长任务异步执行，前端轮询 / WebSocket 推进度。
- 重任务用 Celery / Arq（Redis 队列）独立 worker。

**鉴权**

- OAuth2 + JWT：`OAuth2PasswordBearer` + `python-jose`。
- 依赖注入 `Depends(get_current_user)`。

---

### 14.1 SSE vs WebSocket：区别、选型与 AI 场景落地

**核心区别**

| 维度 | SSE（Server-Sent Events） | WebSocket |
|---|---|---|
| 协议 | HTTP/1.1（或 HTTP/2） | 独立协议 ws:// wss://（基于 HTTP 升级握手） |
| 通信方向 | 单向：服务器 → 客户端 | 全双工：双向实时 |
| 底层连接 | 普通 HTTP 长连接 | 升级后的 TCP 长连接 |
| 数据格式 | 纯文本 `text/event-stream`，按事件帧 | 任意（文本/二进制），自带帧协议 |
| 浏览器 API | `EventSource` | `WebSocket` 对象 |
| 自动重连 | 内置（断线自动重连 + `Last-Event-ID` 续传） | 需手动实现 |
| 代理/防火墙 | 走 HTTP，穿透性好 | 需代理支持 ws 升级 |
| 连接数限制 | HTTP/1.1 同域名 6 连接（HTTP/2 多路复用解除） | 无此限制 |
| 鉴权 | 走 HTTP header/Cookie，简单 | 握手后需自行鉴权（header 仅握手阶段） |
| 服务端成本 | 轻量，复用 HTTP 基础设施 | 需长连接维护、心跳、状态管理 |
| 适用 | 服务器推：LLM 流式输出、通知、行情、日志 | 双向实时：聊天、协作、游戏、多人同步 |

**SSE 协议帧**

```
data: 第一块内容\n
\n
data: 第二块内容\n
\n
event: done\n
data: [DONE]\n
\n
```

- 每条事件以 `\n\n` 分隔；`data:` 是数据行；可加 `event:`/`id:`/`retry:`。
- `id` 用于断线后 `Last-Event-ID` 请求头续传，服务器据此重放。

**FastAPI SSE 实现**

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

app = FastAPI()

@app.post("/chat")
async def chat(req: ChatReq):
    async def gen():
        async for chunk in llm.astream(req.prompt):
            yield f"data: {chunk.content}\n\n"
        yield "data: [DONE]\n\n"
    return StreamingResponse(gen(), media_type="text/event-stream")
```

前端：

```js
const es = new EventSource("/chat?prompt=...");
es.onmessage = e => console.log(e.data);
es.addEventListener("done", () => es.close());
```

**WebSocket 实现（FastAPI）**

```python
from fastapi import FastAPI, WebSocket

app = FastAPI()

@app.websocket("/ws")
async def ws(ws: WebSocket):
    await ws.accept()
    try:
        while True:
            msg = await ws.receive_text()        # 客户端 -> 服务端
            await ws.send_text(f"echo: {msg}")     # 服务端 -> 客户端
    except WebSocketDisconnect:
        pass
```

需自行处理：心跳（ping/pong，默认 20s 超时）、重连、鉴权（握手时校验 token）。

**AI 场景选型**

- **LLM 流式输出 → SSE**：单向推送、HTTP 复用、自动重连、首字时延低，OpenAI/Claude/vLLM 默认用 SSE。
- **多轮对话（用户可中途打断/追问）→ WebSocket**：双向，客户端随时发消息，服务端流式回。
- **实时协同（代码助手、Agent 进度、多人会议纪要）→ WebSocket**：双向 + 低延迟。
- **服务端推送（任务进度、行情、审核结果）→ SSE**：单向足够，简单可靠。

**追问**

- Q：LLM 流式为什么默认用 SSE 而不是 WebSocket？ A：单向即可，复用 HTTP 基础设施（负载均衡、鉴权、限流、CDN），断线自动重连 + 续传，部署简单；WebSocket 双向但需额外维护心跳/重连/状态。
- Q：SSE 在 HTTP/1.1 下的连接数限制？ A：同域名 6 条，多路复用 HTTP/2 解除；生产建议 HTTP/2 + 同域。
- Q：SSE 如何做鉴权？ A：`EventSource` 不支持自定义 header，用 Cookie（同源）或 query token；或用 `fetch` + ReadableStream 自行实现 SSE 客户端以带 header。
- Q：WebSocket 鉴权怎么做？ A：握手阶段（HTTP 升级请求）用 header/cookie/query 带 token 校验；握手后无 HTTP 语义，需在应用层加鉴权消息。
- Q：Nginx 反代 SSE/WebSocket 注意？ A：SSE 关闭 `proxy_buffering` + 设长 `proxy_read_timeout`；WebSocket 加 `proxy_set_header Upgrade $http_upgrade;` + `Connection upgrade` + `proxy_http_version 1.1`。
- Q：SSE vs HTTP 分块传输（chunked）？ A：SSE 是 chunked 之上的应用协议（事件帧 + 自动重连 + event 类型）；裸 chunked 无语义，浏览器需手动解析。

---

### 15. Dify / FastGPT / LlamaIndex / LangChain 对比

| 框架 | 定位 | 优势 | 适用 |
|---|---|---|---|
| Dify | 低代码 AI 应用编排 | 工作流可视化、Agent、RAG、企业级 | 快速搭业务系统 |
| FastGPT | 知识库问答 | 工作流 + 编排、国产化 | 客服/知识库 |
| LlamaIndex | 数据框架 + RAG | 文档加载/索引丰富、代码优先 | 深度定制 |
| LangChain | 通用 AI 应用框架 | 组件全、生态大 | 灵活开发 |

**选型建议**

- 业务快上线、非研发主导 → Dify/FastGPT。
- 深度定制、研究型 → LangChain/LlamaIndex。
- 混合：核心逻辑自研 LangChain，外围用 Dify 编排。

**追问**

- Q：Dify 的工作流和 LangGraph 区别？ A：Dify 可视化拖拽，LangGraph 代码级状态机，控制更精细，适合复杂 Agent 拓扑。

---

## 五、RAG 深度

### 16. RAG 全流程 + 各步优化点

```
文档加载 → 清洗 → 切分 → Embedding → 入向量库 → 索引
查询 → Query 改写 → 检索 → Rerank → 上下文组装 → 生成 → 引用溯源 → 评估
```

**加载**

- PDF：PyMuPDF、Marker（保留结构）、unstructured。
- 表格：Marker/Camelot + Markdown 化。
- 图片：Qwen-VL OCR + 描述。
- 网页：trafilatura。

**清洗**

- 去页眉页脚、目录、水印、空段落。
- 同义词/术语统一。
- 去重（SimHash）。

**切分（下题详述）。**

**Embedding**

- 中文：bge-m3、bge-large-zh、m3e。
- 多语言：bge-m3（稠密 + 稀疏 + ColBERT 三路）。
- 维度：768/1024/1536/3072。

**索引**

- 向量库：Milvus（分布式）、Qdrant（Rust、快）、PGVector（Postgres 扩展，集成简单）、ES（全文+向量混合）。
- 量化：HNSW + PQ/SQ 降内存，召回率换内存。

---

### 17. 切分策略深度

**切分方法**

- 固定大小 + overlap：简单，易割裂语义。
- 递归字符切分（LangChain `RecursiveCharacterTextSplitter`）：按 `["\n\n","\n","。"," "]` 优先级切，保段落完整。
- 语义切分：按句子相似度聚类（SemanticChunker）。
- 结构化：按标题/章节/Markdown header 切分。
- 表格：按行/按 sheet 切；代码按函数。
- QA：按问答对切。

**chunk 大小权衡**

- 太小：单 chunk 信息不全，召回低。
- 太大：噪声多，相关稀释，超 token 限制。
- 经验：256-512 token，overlap 10-20%，结合 rerank 提精度。

**父子块 / Small-to-Big**

- 检索用小块（精确匹配），返回时带父块上下文（更完整）。
- LlamaIndex `HierarchicalNodeParser` + `AutoMergingRetriever`。

**追问**

- Q：chunk 之间重叠的目的？ A：避免切在句子/段落中间导致信息断裂。
- Q：长文档整体理解怎么办？ A：先摘要再切分（Map-Reduce），或层级切分 + 多路检索。

---

### 18. 检索召回优化 + 混合检索 + Rerank

**Query 改写**

- 同义扩展：LLM 生成多个等价 query 并发检索。
- HyDE：让 LLM 先生成假设答案，再用假设答案 embedding 检索。
- Step-back：抽象出更高层问题检索。
- Query 分解：复杂问题拆成子问题分别检索。

**混合检索**

- BM25（关键词，全文）+ 向量（语义）。
- 融合：RRF（Reciprocal Rank Fusion，`1/(k+rank)` 加权）。
- ES 支持 `dismax` 混合。
- Milvus 2.4+ 支持稀疏 + 稠密混合检索。

**Rerank**

- 粗排 top 50-100 → 精排 top 5。
- 模型：bge-reranker、Cohere rerank、jina-reranker。
- 跨编码器（Cross-Encoder）比双塔（双向量）精度高但慢。
- 仅对粗排结果排序，可控成本。

**元数据过滤**

- 时间/类型/权限/来源先过滤，缩小检索范围，提精度降延迟。

**追问**

- Q：向量检索为什么有时不如 BM25？ A：专有名词、代码、ID 类查询，向量相似不准，BM25 关键词匹配更靠谱。
- Q：embedding 模型微调？ A：用领域数据 fine-tune（如 BAAI 的 `bge` 支持对比学习微调），提领域相关性。

---

### 19. RAG 评估 + Ragas

**指标**

| 指标 | 含义 |
|---|---|
| Context Precision | 检索到的上下文相关比例 |
| Context Recall | 应检索到的上下文实际命中比例 |
| Faithfulness | 答案是否忠于上下文（无幻觉） |
| Answer Relevancy | 答案是否切题 |
| Answer Correctness | 与标注答案一致性 |

**工具**

- Ragas：开箱即用，LLM-as-Judge。
- TruLens、LangSmith、Langfuse：trace + 评估。

**评估流程**

1. 构造 golden set（问题 + 答案 + 上下文）。
2. 跑 RAG 系统，记录 retrieved/answer。
3. 跑 Ragas 评分。
4. 分析低分 case，迭代切分/检索/prompt。

**线上监控**

- 用户反馈埋点（赞/踩）。
- 拒答率、引用率、平均检索数。
- 低分 case 回流到评估集，持续改进。

---

## 六、微调与蒸馏

### 20. LoRA 原理 + 选型

**LoRA（Low-Rank Adaptation）**

- 冻结原权重 W，旁路 `ΔW = B·A`，`rank r << d`。
- 训练只更新 `A`、`B`，参数量 `2 × r × d`（约原模型 0.1%）。
- 推理可合并 `W' = W + BA`，无额外延迟。
- 初始化：`A` 高斯，`B` 零，保证训练开始 `ΔW=0` 不破坏原模型。

**为什么低秩有效**

- 微调是「任务适配」，本质低维变化，不需要全秩。
- 论文实验 r=8 即接近全参效果。

**QLoRA**

- 基座 4bit NF4 量化 + LoRA，单卡 24GB 微 7B。
- 反向传播时 LoRA 梯度 fp16，量化基座不动。
- PagedOptimizer 防显存峰值 OOM。

**LoRA 变体**

- DoRA：分解为方向 + 幅度，效果更好。
- LoRA+：A、B 不同学习率。
- rsLoRA：scaling = r/α。
- PiSSA：主成分初始化。

**适用**

- 风格/格式/领域指令适配 → 强。
- 大量新知识记忆 → 弱（应配 RAG）。
- 推理能力提升 → 蒸馏更强模型。

**追问**

- Q：r 怎么选？ A：r=8-64，简单任务小 r，复杂任务大 r；实验调。
- Q：哪些层加 LoRA？ A：q/k/v/o 投影 + FFN 的 gate/up/down；只 attention 也常见。

---

### 21. 微调全流程 + 数据准备

**SFT 流程**

1. 数据：指令-回答对 `{instruction, input, output}`。
2. 格式化：拼成 chat template（`<|im_start|>user ... <|im_end|>`）。
3. 损失：仅对 answer 部分算 loss（mask prompt token）。
4. 训练：transformers `Trainer` 或 LLaMA-Factory。
5. 评估：留出测试集 + 人工/LLM 评估。

**LLaMA-Factory**

- YAML 配置微调，支持 LoRA/QLoRA/DPO。
- 模板：`qwen`、`glm4`、`llama3` 等。
- 可视化 WebUI。

**数据准备**

- 清洗：去重（MinHash）、去噪、脱敏。
- 格式统一：协议解析用「解析这段 Modbus 报文」→「字段含义」。
- 难例挖掘：解析失败 case 单独加权。
- 评估集分层抽样（按协议/设备类型），防数据泄漏。

**追问**

- Q：SFT 与 RLHF 区别？ A：SFT 模仿示范，RLHF 用奖励模型强化偏好（PPO/DPO）。
- Q：过拟合怎么办？ A：少 epoch（1-3）、加正则、留验证集早停、增大样本量。

---

### 22. 知识蒸馏 + DeepSeek 蒸 Qwen2.5-3B CoT

**蒸馏类型**

- 黑盒：用 Teacher 生成输出当 SFT 数据训 Student（response-based，主流）。
- 白盒：logit 级 KL 散度（soft target）+ 中间层特征对齐（feature-based）。

**用 DeepSeek 蒸 Qwen2.5-3B**

1. 构造指令集（IOT 协议、金融风控、推理题）。
2. Teacher（DeepSeek-R1）输出带 CoT 的解答。
3. 过滤：质量过滤（正确性、步骤完整）+ 多样性去重。
4. SFT 训 Student（Qwen2.5-3B）学习模仿 CoT。
5. 评估：CoT 步骤一致性 + 最终答案准确率。

**损失**

- 标准 SFT 损失（最大似然 teacher token）。
- 进阶：加 logit KL（需 teacher 暴露 logits）。

**目的**

- 把大模型的推理能力下沉到小模型，降本提速、可本地部署。

**追问**

- Q：蒸馏 vs 微调 vs RAG？ A：蒸馏是能力迁移（永久内化）；微调注入领域风格；RAG 是外挂知识（可更新）。
- Q：Student 容量不够怎么办？ A：分模块蒸馏、渐进蒸馏、用更大 Student 或多 Student 集成。

---

### 23. Self-Instruction + 数据自动生成

**流程**

1. 种子指令（人工 175 条左右）。
2. LLM 生成新指令 + 输入 + 输出。
3. 过滤：
   - 去重（与已有指令 ROUGE-L < 0.7）。
   - 质量过滤（语言、长度、是否指令）。
4. 微调得到 InstructGPT 风格模型。

**注意**

- 多样性：防止生成同质化指令。
- 质量边界：LLM 生成可能有错，需人工抽样校验。
- 偏见：Teacher 偏见传递到 Student。

**项目映射**：IOT 协议解析微调，用 GPT-4o 根据种子协议示例生成更多「报文 → 解析」对，扩量至 10 万条。

---

### 24. GLM4-9B 微调后协议解析准确率 +40% 验证

**评估设计**

- 测试集：按协议（MQTT/Modbus/CoAP）+ 设备类型分层抽样，覆盖正常/异常报文。
- 指标：
  - 字段级 EM（exact match）。
  - 字段级 F1（部分识别也算分）。
  - 解析成功率（端到端能给出完整解析）。
- 对比：微调前（base）vs 微调后，同测试集。

**线上回测**

- 真实报文回放，统计解析成功率。
- 人工抽查 100 条，标注正确性。
- 失败归因：未覆盖协议变种、报文截断、单位换算错。

**闭环**

- 失败样本回流 → 自动重生成数据 → 下一轮微调。
- 监控：解析失败率超阈值告警，触发数据补充。

**追问**

- Q：微调会忘通用能力吗？ A：会，灾难遗忘；用混入通用数据 + 多任务学习缓解。
- Q：多个协议会不会互相干扰？ A：可能，用 LoRA 多 adapter 隔离或按协议分模型。

---

## 七、计算机视觉

### 25. YOLOv5 / v10 / Co-DETR 原理对比

**YOLOv5**

- Anchor-based，Backbone（CSPDarknet）+ Neck（PANet）+ Head。
- 工程化好：训练脚本/导出/部署齐全，社区资源多。
- 适合：实时检测、工程落地快。

**YOLOv8/v10**

- Anchor-free，decoupled head（分类/回归分支分离）。
- v10：NMS-free，双标签分配（一对多训练 + 一对一推理），端到端无后处理，延迟低。
- v8：Task-Aligned Assigner（TAL），分类+定位对齐打分。

**Co-DETR**

- 基于 DETR 的多协同头训练：多个辅助检测头（ATSS/DETR）共同监督 Encoder 特征，提升特征质量。
- 推理只用一个轻量头，精度高速度可接受。
- 适合：对精度敏感的工业检测（小目标、密集场景）。

**追问**

- Q：Anchor-based vs Anchor-free？ A：前者需先验框设计、超参多；后者直接预测中心点/距离，更简洁。
- Q：NMS 为什么被诟病？ A：串行、阈值敏感、不可微、增加延迟，端到端框架（DETR 系）去 NMS。

---

### 26. 烟火/漏油检测工程化

**数据**

- 实地采集 + 公开数据集（FASDD 烟火、变电站巡检数据）+ 合成（Copy-Paste）。
- 标注 COCO 格式；类别不平衡用 focal loss / 过采样。
- 增强：雾/夜/遮挡/旋转/色相扰动，模拟实际部署环境。

**训练**

- 小目标：高分辨率切片推理（SAHI），大图切小片分别推理再合并。
- 漏油：纹理特征，小目标 + 暗光；用 attention 机制 + 多尺度特征。
- Loss：CIoU/DIoU 提升框回归。

**部署**

- 导出 ONNX → TensorRT 量化（FP16/INT8）加速 3-5x。
- Docker 打包推理服务（FastAPI + TensorRT）。
- 边缘设备：Jetson + TensorRT + DeepStream 流式推理。

**追问**

- Q：误报率高怎么办？ A：阈值调高 + 后处理（面积/持续时间过滤）+ 双模型投票。
- Q：模型更新如何灰度？ A：A/B 测试 + 影子流量 + 监控指标平稳后切流。

---

### 27. COCO/VOC 互转 + 数据格式

**VOC**

- XML per image，字段：folder/filename/size/object（含 bndbox）。
- 坐标 `[xmin,ymin,xmax,ymax]` 绝对像素。

**COCO**

- 单 JSON，字段：images/annotations/categories。
- bbox：`[x,y,w,h]`（左上角 + 宽高），category_id 从 1 开始。
- 含 segmentation、area、iscrowd。

**转换**

```python
from pylabel import Importer
# VOC -> COCO
importer = Importer(xml_path, "voc")
dataset = importer.export("coco.json", "coco")
```

- 注意点：类别 id 映射、坐标系差异（左上 vs 绝对像素需要归一化时除以图宽高）、segmentation 多边形。

---

### 28. Gradio / Streamlit 搭建 Demo

**Gradio Blocks**

```python
import gradio as gr

with gr.Blocks() as demo:
    gr.Markdown("# 漏油检测")
    with gr.Row():
        inp = gr.Image(type="filepath")
        out = gr.Image()
    btn = gr.Button("检测")
    btn.click(detect, inp, out)
    gr.Examples(["sample/a.jpg", "sample/b.jpg"], inp)

demo.queue().launch(server_name="0.0.0.0", share=True)
```

- `share=True` 生成临时公网链接（72h），方便演示。
- `queue` 处理并发；`gr.Camera` 实时摄像头输入。
- 部署：`gradio` + Docker + Nginx 反代。

**Streamlit**：偏数据看板，适合展示分析结果；Gradio 偏模型交互。

---

## 八、Docker / 工程化

### 29. 模型服务 Docker 化 + GPU

**多阶段 Dockerfile**

```dockerfile
FROM python:3.10-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user -r requirements.txt

FROM nvidia/cuda:12.1.1-runtime-ubuntu22.04
COPY --from=builder /root/.local /root/.local
COPY . /app
WORKDIR /app
ENV PATH=/root/.local/bin:$PATH
CMD ["python", "-m", "vllm.entrypoints.openai.api_server", "--model", "Qwen-VL"]
```

- 基础镜像选 `nvidia/cuda` 拿 GPU 运行时；或用 PyTorch 官方镜像。
- 模型权重外挂 volume，避免打进镜像（镜像大 + 更新麻烦）。
- `--gpus all` 启用 GPU，需 `nvidia-container-toolkit`。

**docker-compose**

```yaml
services:
  llm:
    image: my-llm
    volumes:
      - ./models:/models
    ports: ["8000:8000"]
    deploy:
      resources:
        reservations:
          devices:
            - { driver: nvidia, count: 1, capabilities: [gpu] }
  redis: { image: redis }
  milvus: { image: milvusdb/milvus }
```

**K8s**

- Deployment + HPA（按 GPU 利用率/QPS 扩缩）。
- GPU 调度：节点标签 + `nodeSelector` 或 GPU operator。
- 模型权重用 PVC / 对象存储 + initContainer 拉取。

---

### 30. AI 服务限流降级 + 大模型超时处理

**限流层**

- 网关：Gateway + Sentinel / Nginx limit_req。
- 应用：令牌桶（`slowapi`、Sentinel）。
- 模型层：vLLM 队列 + 优先级 + 最大等待。

**降级策略**

- 超时 → fallback 到缓存结果。
- 高负载 → 降级到更小模型（3B 替 70B）。
- 关键路径失败 → 默认安全回复 + 转人工。

**流式输出**

- SSE 分块返回，首字时延低，用户感知快。
- 总超时仍设上限（如 60s），避免长时间占用。

**追问**

- Q：高并发下 GPU 排队怎么控制？ A：vLLM 内置 scheduler + `max_num_seqs`；外层用 Redis 排队 + 限流。
- Q：模型冷启动慢怎么办？ A：常驻服务、预热、模型缓存、避免频繁 reload。

---

## 九、项目实战

### 31. AI 金融风控平台完整架构

```
[前端上传材料] → [FastAPI 网关 + JWT 鉴权]
       │
       ▼
[Manager Agent (调度/仲裁/聚合)]
   ├── 身份认证 Agent (Qwen-VL OCR + 校验)
   ├── 银行流水 Agent (Qwen-VL + 金额汇总 + 异常检测)
   └── 工作证明 Agent (Qwen-VL + Google Search MCP 核验)
       │
       ▼
[风险评分 + 可解释依据] → MySQL + ES 全文检索
```

**能力层**

- Qwen-VL（视觉）：身份证/流水/工作证明图片理解。
- Whisper（语音）：电话核身录音转文字。
- Google Search MCP：企业资质/工商信息核验。
- vLLM：统一推理服务，支持多模型加载。

**存储层**

- 向量库（Milvus）：历史材料语义检索，辅助可比案例。
- MySQL：审核结果 + 风险评分。
- Redis：任务状态 + 限流 + 幂等。

**输出**

- 风险评分（0-100）+ 风险等级 + 各维度子结论 + 每条结论溯源到材料片段。
- 可解释：人工复核时能定位依据，符合监管要求。

---

### 32. 85% 自动化率怎么衡量 + 15% 兜底

**衡量**

- 自动化率 = 无需人工即可出具终审建议的占比。
- 分母：所有申请单；分子：Agent 终审被直接采用的单子。
- 评估周期：按周/月统计，注意样本均衡。

**兜底（15%）**

- 置信度阈值（如综合评分 < 0.8）→ 转人工。
- 疑似造假标记（图片异常、流水逻辑矛盾）→ 人工复核。
- 模型未见过的材料形态（新格式）→ 人工 + 数据回流训练。

**持续提升**

- 复核结果回流形成新训练/提示样本。
- 每月评估自动化率提升，迭代 prompt + 微调。

---

### 33. 虚假材料识别率 90% 实现

**多维度交叉**

- 身份：OCR 与公安/公开信息比对，人脸与身份证照片比对。
- 银行流水：金额走势合理性、收支逻辑、异常大额检测、银行盖章识别。
- 工作证明：企业存续核验（Google Search MCP 查工商）、职位/薪资合理性、印章识别。

**VLM 关注**

- 水印/P 图痕迹、字体异常、章颜色不对、纸张反光、字段错位。
- 多模态：图片 + 文字交叉验证。

**规则 + LLM 双轨**

- 规则硬约束：月收入 > X 但流水不匹配 → 高风险。
- LLM 语义判断：工作描述与行业常识矛盾、收入与职位差距大。
- 输出每条判断依据，便于人工复核。

---

### 34. AI 爬虫平台架构 + 反爬

**架构**

```
[FastAPI 任务入口] → [调度 Agent (LangChain)]
   → [Playwright MCP (浏览器自动化)]
   → [5 大搜索引擎 + 8 个香港网站]
   → [结果解析 + 去重]
   → [高德 MCP 补地理信息]
   → [Neo4j 关系图谱] → [订阅/推送]
```

**反爬**

- Playwright 真实浏览器渲染，绕 JS 检测。
- 指纹随机化：user-agent、viewport、timezone、webgl、canvas。
- 代理池 + 限速 + 指数退避重试。
- 验证码：2Captcha / 打码平台 / 人工兜底队列。
- 行为人性化：鼠标移动、随机停顿。

**稳定性**

- 任务幂等：URL hash 去重。
- 增量爬：按时间/版本号。
- 监控：成功率、平均耗时、被拒次数告警。

---

### 35. 社交关系图谱构建

**数据采集**

- 好友/关注/共同群组/互动（评论点赞）/转发关系。

**图存储**

- Neo4j / NebulaGraph：节点 = 人，边 = 关系（带权重/时间/类型）。
- 节点属性：姓名、手机、ID、平台账号。

**分析**

- 连通子图找关联团伙。
- 中心性（度/介数/接近）定位核心人物。
- 路径搜索查关系链（最短路径、k 短路）。
- 社区发现（Louvain/Label Propagation）。

**LLM 集成**

- 自然语言查询「A 和 B 的关系路径」→ LLM 转 Cypher（Text-to-Cypher）。
- 关系解读：LLM 把图谱路径翻译成自然语言描述。

**追问**

- Q：图谱怎么更新？ A：定时增量 + 全量重建；变更日志 + 版本管理。
- Q：百万级节点性能？ A：NebulaGraph/JanusGraph 分布式；按标签分片；缓存热点节点。

---

### 36. 人工查人 半天→15 分钟 关键优化

- 5 大搜索引擎 + 8 个香港网站并行检索，MCP 统一接口降接入成本。
- LLM 自动选检索词、聚合去重、按可信度排序。
- 关系图谱一图呈现，省去人工逐个网页比对。
- 缓存高频查询结果，二次查询秒级返回。

---

### 37. IOT + LLM 微调开发效率 +60%

**体现**

- 协议解析从手写解析器/查文档 → 自然语言描述报文即得解析结果。
- 自动生成设备接入代码骨架（MQTT 订阅、字段映射、数据校验）。
- 减少协议专家介入，普通工程师可独立接入新设备。
- 错误排查：把异常报文喂模型，给出可能字段含义 + 排查方向。

**效率衡量**

- 接入新设备时间：平均 3 天 → 1 天。
- 重复报文解析工时下降。
- 文档查阅次数下降。

---

### 38. 成本与延迟优化

**成本**

- Token 计量：input + output，按模型单价计。
- 多 Agent 算总调用次数；工具调用回包也算 input。
- 优化：
  - 小模型路由（3B 分类 + 70B 生成）。
  - 缓存高频 query（语义缓存）。
  - RAG 召回裁剪减少上下文 token。
  - prompt 精简（去示例、压缩指令）。

**延迟**

- 首字时延 TTFT：模型加载 + prefill 时间。
- 总时延：prefill + decode × token 数。
- 优化：
  - 并行多 Agent（独立子任务）。
  - 流式输出降感知延迟。
  - 批处理（continuous batching）。
  - KV 缓存复用前缀（vLLM prefix caching）。

---

### 39. AI 应用可观测性

**日志**

- 输入/输出（脱敏）、token 用量、延迟、模型版本、prompt 版本。
- 工具调用入参/出参/耗时。

**Trace**

- LangSmith / Langfuse：Agent 调用链 + 每步耗时 + token。
- OpenTelemetry 接入 APM。

**业务指标**

- 自动化率、识别准确率、人工干预率、幻觉率、拒答率。
- A/B 实验对比新旧 prompt/模型。
- 灰度发布：先小流量，监控平稳后扩量。

---

## 十、开放追问

### 40. 为什么从 Java 转 AI

- 工程化能力（微服务/高并发/消息/缓存/可观测）能把模型真正落地成可靠系统。
- IOT 行业 + 数据，能做垂直领域微调与 RAG，懂业务也懂技术。
- 既懂后端又懂部署，能端到端交付，比纯算法工程师更贴近业务。
- 持续学习：自驱跟进 LLM/Agent/RAG 前沿。

### 41. AI 项目最大技术挑战

- 风控 Agent 输出不稳定 → JSON Schema 校验 + 失败重试 + Prompt 约束 + Few-shot。
- 多 Agent 信息冲突 → 中心仲裁 + 置信度加权 + 规则裁决。
- 微调数据质量差 → 自动化清洗 + 人工抽样 + 难例挖掘 + 数据增强。

### 42. 如何避免 LLM 幻觉

- RAG 提供可溯源上下文 + 提示「不确定时回答不知道」。
- 结构化输出 + Schema 校验。
- 多路投票 / 自我批判（Self-Refine / Self-Check）。
- 关键领域（金融）规则校验兜底，LLM 仅做辅助建议。
- 引用溯源：每条结论标注来源片段，便于复核。

### 43. Agent 发展趋势

- 单 Agent → Multi-Agent 协作（专业分工）。
- 脚本编排 → 自适应规划（动态调整）。
- 工具协议标准化（MCP / Function Call）使生态可复用。
- 端侧小模型 + 云端大模型协同，成本与隐私兼顾。
- 长记忆 + 持续学习 + 个性化。

### 44. 给你保险理赔自动化怎么设计

**需求拆解**：材料种类（报案单/医疗单/凭证）、规则（保单条款）、流程（报案→定损→核赔→支付）。

**架构**：

- FastAPI + Multi-Agent（材料识别 Agent / 规则引擎 Agent / 欺诈检测 Agent）。
- RAG 沉淀历史理赔案例 + 保单条款库。
- 微调小模型做特定单据（医疗发票）OCR 后处理。
- 规则引擎（LiteFlow/Drools）做硬约束（保额、免赔、条款匹配）。

**指标**：自动化率、欺诈识别率、平均处理时长、人工干预率。

**策略**：先规则后模型，灰度上线，复核结果回流训练，持续提升自动化率。
