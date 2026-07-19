---
title: AI 应用面试题
tags: [面试, AI, LLM, RAG, Agent]
created: 2026-07-19
---

# AI 应用方向面试题

> 基于简历技能栈与项目经验定制。每题给出参考答案要点，便于快速复习。

---

## 一、LLM 基础与部署

### 1. 介绍 Transformer 的核心结构？
- Encoder/Decoder 堆叠，多头自注意力 + 前馈网络 + 残差连接 + LayerNorm。
- 注意力：`Q·K^T / √dk` → softmax → × V，捕捉长程依赖。
- 位置编码（正弦/RoPE/ALiBi）补充顺序信息。
- GPT 系列只用 Decoder（自回归）；BERT 只用 Encoder（双向）。

### 2. vLLM 部署 Qwen-VL / Whisper 为什么快？
- PagedAttention：把 KV 缓存按页管理，类似 OS 虚拟内存，显存碎片低、吞吐高。
- Continuous Batching：动态拼 batch，请求可中途加入/退出。
- 配合 `tensor-parallel-size` 多卡张量并行；INT8/INT4 量化（AWQ/GPTQ）降显存。
- Qwen-VL：视觉编码器（ViT）+ Adapter + LLM；Whisper：Encoder-Decoder 多任务。

### 3. Qwen-VL 在金融风控里怎么用？
- 图像理解：识别身份证、银行卡、银行流水截图、工作证明图片，提取结构化字段。
- Prompt 模板化：固定 JSON 输出，配合 JSON Schema 校验。
- 多模态 RAG：图片 + 文本对齐检索；流水金额汇总、异常标记。

### 4. Whisper 语音识别的适用场景与局限？
- 多语言、噪声鲁棒、支持时间戳和翻译；32 秒窗口。
- 中文长音频需 VAD 切分 + 多段拼接；方言/专业术语需 fine-tune 或加 GPT 后处理纠错。
- 风控里可用于电话核身录音转文字，再喂 LLM 做风险判断。

### 5. LLM 推理显存估算？Qwen2.5-3B 部署需要多少？
- 参数 × 字节数：FP16 约 6GB；INT8 约 3GB；INT4 约 1.8GB。
- 加 KV 缓存、激活、上下文长度开销，实际留 2x 余量。
- 3B INT4 单卡 6GB 可跑；推理用 vLLM，训练微调需 A10/3090 以上。

---

## 二、Agent 与多智能体

### 6. 什么是 Agent？与普通 LLM 调用区别？
- Agent = LLM（大脑）+ 规划 + 工具调用 + 记忆，能感知-决策-行动闭环。
- 普通调用是「一问一答」；Agent 可多步、自主选择工具、根据观察结果调整下一步。
- 核心：ReAct（Reasoning + Acting）、Plan-and-Execute、Reflection。

### 7. Multi-Agent 协同架构怎么设计？金融风控的三个 Agent 如何分工？
- 角色拆分：身份认证 Agent（OCR + 真伪校验）、银行流水 Agent（金额汇总 + 异常检测）、工作证明 Agent（企业核验 + 时间逻辑）。
- 编排方式：中心调度（Manager）/ 顺序流水线 / 争论辩论 / 并行投票。
- 协同要素：消息总线、共享状态、任务路由、冲突仲裁、结果聚合。
- 工具复用：Google Search MCP 做企业资质核验，统一检索接口。

### 8. ReAct 和 Plan-and-Execute 区别？什么时候用哪个？
- ReAct：每步 Thought → Action → Observation，灵活但易跑偏，适合探索性任务。
- Plan-and-Execute：先整体规划再执行，可控性强、Token 更省，适合流程固定任务（风控流水线）。
- 实践：风控用 Plan-and-Execute 框定步骤，单步内可嵌 ReAct 应对意外。

### 9. Agent 记忆机制有哪些？
- 短期：对话上下文窗口、scratchpad。
- 长期：向量库（语义检索历史）、知识图谱、摘要压缩（rollup）。
- LangChain `Memory`：`ConversationBufferMemory`、`ConversationSummaryMemory`、`ConversationKGMemory`。

### 10. 如何防止 Agent 死循环/无限调用工具？
- 设置 `max_iterations`、单步超时、总 Token 上限。
- 工具调用结果加校验 + fallback；循环检测（重复 Action 触发终止）。
- 用更小模型做路由/校验，主模型专注生成，降低成本与延迟。

---

## 三、MCP（Model Context Protocol）

### 11. MCP 是什么？相比直接调 Function Calling 优势？
- Anthropic 提出的开放协议，统一「模型 ↔ 外部工具/数据源」连接，类似 USB-C for LLM。
- 优势：工具/资源/提示模板标准化复用；一个 Server 可被多 Client 共享；解耦模型与工具实现。
- 与 Function Calling：FC 是模型能力，MCP 是传输与发现机制，二者可叠加。

### 12. 高德 MCP 和 Playwright MCP 在爬虫平台里怎么用？
- 高德 MCP：地理编码、POI 检索、路径规划；用户画像/关系图谱中地点维度补全。
- Playwright MCP：浏览器自动化，处理 JS 渲染页面、登录态、反爬、点击/翻页。
- 流程：Agent 决策 → 调 Playwright MCP 取页面 → 解析 → 调高德 MCP 补地理信息 → 落库。

### 13. 自定义 MCP Server 的开发流程？
- 实现 `tools/list`、`resources/list`、`prompts/list`、`tools/call` 接口（stdio 或 SSE）。
- Python SDK：`@mcp.tool` 装饰器；声明入参 schema；用 stdio 传输启动。
- 客户端配置 `mcp.json` 注册 server，LangChain/Agent 通过名字调用。

---

## 四、LangChain / FastAPI / 应用框架

### 14. LangChain 的核心抽象？Chain 与 LCEL 区别？
- 抽象：`PromptTemplate`、`LLM`、`Memory`、`Tool`、`Retriever`、`AgentExecutor`、`Chain`。
- LCEL（LangChain Expression Language）：`prompt | llm | parser` 管道式，支持流式、异步、批处理、回溯。
- 推荐用 LCEL 替代旧 Chain，更易组合和观测。

### 15. FastAPI 为什么适合搭 AI 应用？
- 基于 Starlette + Pydantic，异步非阻塞，适合等待 LLM 流式返回。
- `StreamingResponse` + `SSE` 实现打字机效果；WebSocket 做实时交互。
- 自动 OpenAPI 文档，便于前端联调；`BackgroundTasks` 跑异步爬虫任务。

### 16. Dify / FastGPT / LlamaIndex 各自定位？
- Dify：低代码 AI 应用编排，工作流 + Agent + RAG，适合快速搭业务系统。
- FastGPT：偏知识库问答，工作流 + 可视化编排，国产化场景多。
- LlamaIndex：代码优先的数据连接与 RAG 框架，灵活度高，适合深度定制。
- 选型：业务侧快上线用 Dify/FastGPT；研发深度定制用 LangChain/LlamaIndex。

---

## 五、RAG 全流程

### 17. RAG 完整流程？每一步怎么做？
- 文档加载 → 切分 → Embedding → 入向量库 → 检索 → 拼上下文 → 生成 → 后处理。
- 切分：固定 size + overlap，或按语义/标题切分；表格用专门解析（unstructured/Marker）。
- Embedding：bge-m3、text-embedding-3、Qwen embedding。
- 向量库：Milvus / Qdrant / PGVector / Elasticsearch。
- 检索：top-k + rerank（bge-reranker/Cohere）+ 混合检索（BM25 + 向量）。
- 生成：拼 prompt + 引用溯源。

### 18. 切分粒度怎么定？太小太大有什么问题？
- 太小：语义割裂、检索召回率低；太大：噪声多、相关性稀释、超 token 上限。
- 经验：chunk 256–512 token，overlap 10–20%；按段落/标题切更语义连贯。
- 表格、代码、问答对单独策略：表格按行、代码按函数、QA 按对。

### 19. 检索召回率低怎么优化？
- 查询改写：query 扩展 / HyDE 生成假设答案再检索 / 多 query 并发。
- 混合检索：BM25（关键词）+ 向量（语义）RRF 融合。
- Rerank：粗排 top 50 → 精排 top 5。
- 元数据过滤：时间/类型/权限过滤缩小范围。
- 父子块检索 / Small-to-Big：小块检索，返回大块上下文。

### 20. 如何评估 RAG 系统？
- 检索：Recall@k、MRR、NDCG。
- 生成：Faithfulness（是否忠于上下文）、Answer Relevancy、Context Relevancy（Ragas）。
- 端到端：人工标注 + LLM-as-Judge；上线后 A/B + 用户反馈埋点。

---

## 六、微调与蒸馏

### 21. LoRA 原理？为什么轻量？
- 冻结原权重，旁路注入 `B·A` 低秩矩阵（r << d），只训练新增参数。
- 显存大幅下降（约 0.1% 参数），效果接近全参微调。
-QLoRA：4bit 量化基座 + LoRA，单卡可微 7B。
- 适用：风格、领域知识注入；不适合大量新知识记忆（应配 RAG）。

### 22. Self-Instruction 是什么？数据怎么造？
- 用强模型（GPT-4o/DeepSeek）根据种子指令生成新指令→输入→输出，过滤去重。
- 流程：种子 prompt → 生成 → 多样性过滤 → 质量过滤 → 微调数据集。
- 用于低资源场景快速构造垂直领域 SFT 数据。

### 23. 知识蒸馏怎么做？用 DeepSeek 蒸馏 Qwen2.5-3B 的 CoT？
- Teacher（DeepSeek）输出带思维链的答案，Student（Qwen2.5-3B）学习模仿。
- 损失：seq-level 蒸馏（最大似然 teacher 输出），可加 logit 级 KL 散度。
- 目的：把大模型的推理链能力迁移到小模型，降本提速。
- 评估：CoT 正确性 + 推理步骤一致性 + 最终答案准确率。

### 24. 你怎么清洗近 10 万条 IOT 数据用于微调？
- 去重、去噪、脱敏；按协议（MQTT/Modbus）分类标注。
- 统一格式为指令-回答对：「解析这段 Modbus 报文」→「字段含义…」。
- 长样本截断/分段，构造难例（解析失败 case）单独加权。
- 评估集划分：按协议类型分层抽样，避免数据泄露。

### 25. GLM4-9B 微调后协议解析准确率提升 40%，怎么验证？
- 留出测试集（按设备/协议分层），对比微调前后准确率、F1、字段级 EM。
- 真实线上报文回测，统计解析成功率与人工抽查。
- 错误归因：未覆盖协议、报文截断、单位换算，迭代补充数据。

---

## 七、计算机视觉

### 26. YOLOv5 / YOLOv10 / Co-DETR 区别？
- YOLOv5：成熟工程化，Anchor-based，单阶段检测快。
- YOLOv10：NMS-free 双标签分配，端到端、延迟更低。
- Co-DETR：多协同头训练（DETR 变体），精度高但速度慢，适合对精度敏感的工业检测。

### 27. 烟火检测、变电站漏油检测，数据与训练怎么做？
- 数据：实地采集 + 公开数据集 + 数据增强（雾/夜/遮挡），标注 COCO 格式。
- 类别不平衡：focal loss / 过采样 / 合成（Copy-Paste）。
- 漏油：小目标，高分辨率切片推理（SAHI）+ 上下文窗口。
- 部署：TensorRT/ONNX 量化加速；Docker 打包推理服务。

### 28. COCO 和 VOC 格式互转？
- VOC：XML per image，坐标 `[xmin,ymin,xmax,ymax]`。
- COCO：单个 JSON，`[x,y,w,h]` + `image_id` + `category_id` + `segmentation`。
- 工具：`pylabel`、`fiftyone`、自写脚本；注意类别 id 从 0 还是从 1。

### 29. Gradio 搭建 Demo 的最佳实践？
- `gr.Interface` 简单推理；`Blocks` 搭复杂多组件页面。
- 摄像头/图片输入 + 检测结果可视化（`gr.Image` + bboxes overlay）。
- 加 `examples` 提升体验；`queue` + 异步处理并发；`launch(share=True)` 临时公网。

---

## 八、Docker 与工程化

### 30. 模型服务 Docker 化要注意什么？
- 基础镜像选 `nvidia/cuda` 或 `pytorch/pytorch`；多阶段构建减小镜像。
- GPU：`--gpus all` + nvidia-container-runtime。
- 模型权重外挂 volume，避免打进镜像；`docker compose` 管理依赖（Redis/Milvus）。
- 健康检查 + 优雅退出 + 日志挂载；K8s 下用 Deployment + HPA。

### 31. AI 服务怎么限流降级？大模型超时怎么处理？
- 网关层限流（Gateway + Sentinel）；应用层令牌桶；模型层排队 + 优先级。
- 超时：设置 timeout + fallback（缓存上次结果/降级到小模型/提示重试）。
- 流式输出避免一次性超时；用 SSE 分块返回降低首字时延感知。

---

## 九、项目实战与综合

### 32. AI 金融风控平台整体架构怎么设计？
- 接入层：FastAPI + JWT 鉴权；上传材料（PDF/图片/录音）。
- 编排层：Multi-Agent（LangChain），Manager 调度三个子 Agent。
- 能力层：Qwen-VL（视觉）+ Whisper（语音）+ Google Search MCP（核验）+ vLLM 推理。
- 存储层：向量库（材料语义检索）+ MySQL（结果）+ Redis（任务状态/限流）。
- 输出：风险评分 + 可解释依据（每条结论溯源到材料片段）。

### 33. 85% 自动化率怎么衡量？剩余 15% 如何兜底？
- 自动化率 = 无需人工即可出具终审建议的占比。
- 剩余 15%：置信度低于阈值（如 < 0.8）、疑似造假、模型未见过的材料形态 → 转人工复核队列。
- 复核结果回流形成新训练/提示样本，持续提升自动化率。

### 34. 虚假材料识别率 90% 怎么做到的？
- 多维度交叉：身份 OCR 与公安/公开信息比对、银行流水金额与逻辑一致性、工作证明企业存续核验。
- VLM 关注水印/P图痕迹/字体异常；流水金额走势异常检测。
- 规则 + LLM 双轨：规则做硬约束（如月收入 > 阈值但流水不匹配），LLM 做语义判断。

### 35. AI 爬虫平台如何做反爬与稳定性？
- Playwright 真实浏览器渲染，绕过 JS 检测；指纹随机化（user-agent、viewport、时区）。
- 代理池 + 限速 + 失败重试与指数退避；验证码识别（2Captcha/打码平台）或人工兜底。
- 任务幂等：URL hash 去重；增量爬取按时间/版本号。
- 监控：成功率、平均耗时、被拒次数告警。

### 36. 社交平台关系图谱怎么构建？
- 数据采集：好友/关注/共同群组/互动（评论点赞）。
- 图存储：Neo4j / NebulaGraph，节点=人，边=关系（带权重/时间）。
- 分析：连通子图找关联、中心性定位核心人、路径搜索查关系链。
- LLM 辅助：自然语言问答「A 和 B 的关系路径」转 Cypher。

### 37. 人工查人从半天降到 15 分钟，关键优化点？
- 5 大搜索引擎 + 8 个香港网站并行检索；MCP 统一接口降低接入成本。
- LLM 自动选择检索词、聚合去重、按可信度排序。
- 关系图谱一图呈现，省去人工逐个网页比对。

### 38. IOT + LLM 微调后，开发效率提升 60% 体现在哪？
- 协议解析从手写解析器/查文档 → 自然语言描述报文即得解析结果。
- 自动生成设备接入代码骨架（MQTT 订阅、字段映射）。
- 减少 IoT 协议专家介入，普通工程师可独立接入新设备。

### 39. 你怎么衡量 Agent 系统的成本与延迟？
- 成本：input/output token 数 × 单价；多 Agent 要算总调用次数。
- 延迟：首字时延 TTFT + 总时延；多步串联可改并行/缓存中间结果。
- 优化：小模型路由 + 大模型兜底；缓存高频 query；RAG 召回裁剪减少上下文 token。

### 40. 上线一个 AI 应用，你会做哪些可观测性？
- 输入/输出日志（脱敏）+ token 用量 + 延迟分位。
- LangSmith / Langfuse 记录 Agent trace、工具调用链、每步耗时。
- 业务指标：自动化率、识别准确率、人工干预率；异常告警（幻觉/拒答）。
- A/B 实验：新 prompt/模型灰度对比。

---

## 十、可能被追问的开放题

### 41. 为什么从 Java 转 AI？你的优势？
- 工程化能力：微服务、高并发、消息、缓存、可观测性，把 AI 模型真正落地成可靠系统。
- IOT 行业知识 + 数据，能做垂直领域微调与 RAG。
- 既懂后端又懂模型部署，能端到端交付。

### 42. AI 项目里你遇到的最大技术挑战？怎么解决？
- 可选：风控 Agent 输出不稳定 → 加 JSON Schema 校验 + 重试 + Prompt 约束 + 少样本示例。
- 多 Agent 信息冲突 → 设计中心仲裁 + 置信度加权。
- 微调数据质量差 → 自动化清洗 + 人工抽样 + 难例挖掘。

### 43. 如何避免 LLM 幻觉？
- RAG 提供可溯源上下文 + 提示「不确定时回答不知道」。
- 结构化输出 + 校验；多路投票/自我批判（Self-Refine）。
- 关键领域（金融）规则校验兜底，LLM 仅做辅助建议。

### 44. 你怎么看 Agent 的发展趋势？
- 从单 Agent 到 Multi-Agent 协作；从脚本编排到自适应规划。
- 工具协议标准化（MCP/Function Call）使生态可复用。
- 端侧小模型 + 云端大模型协同，成本与隐私兼顾。

### 45. 给你一个新业务（如保险理赔自动化），你怎么设计？
- 需求拆解：材料种类、风控规则、人工流程。
- 架构：FastAPI + Multi-Agent（理赔材料识别 / 规则引擎 / 欺诈检测 Agent）。
- RAG 沉淀历史理赔案例；微调小模型做特定单据 OCR 后处理。
- 指标：自动化率、欺诈识别率、平均处理时长；先规则后模型，灰度上线。
