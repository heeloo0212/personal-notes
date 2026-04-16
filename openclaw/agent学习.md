# Agent 学习笔记

> 更新时间：2026-04-16
> 学习方向：LangChain > 1.0 实战项目

---

## 一、官方核心项目

### 1. langchain-ai/langchain
- **地址**：https://github.com/langchain-ai/langchain
- **说明**：LangChain 官方主仓库，v1.x 最新版。核心功能包括 chains、agents、memory、callbacks。
- **安装**：`pip install langchain`
- **快速上手**：
  ```python
  from langchain.chat_models import init_chat_model
  model = init_chat_model("openai:gpt-5.4")
  result = model.invoke("Hello, world!")
  ```

### 2. langchain-ai/langgraph
- **地址**：https://github.com/langchain-ai/langgraph
- **说明**：官方 Agent 流编排框架，用于构建可控、复杂任务的生产级 Agent 工作流。
- **适用场景**：多步骤推理、循环控制、状态管理。

### 3. langchain-ai/deepagents
- **地址**：https://github.com/langchain-ai/deepagents
- **说明**：官方深度 Agent 示例项目，含规划工具、子Agent、文件系统交互，适合学习高级 Agent 架构。

---

## 二、社区热门实战项目

### 4. FlowiseAI/Flowise
- **地址**：https://github.com/FlowiseAI/Flowise
- **语言**：TypeScript
- **说明**：可视化拖拽式 AI Agent 构建平台，大厂生产级使用。学习如何将 LangChain 封装成产品。

### 5. chatchat-space/Langchain-Chatchat
- **地址**：https://github.com/chatchat-space/Langchain-Chatchat
- **语言**：Python
- **说明**：中文！基于 LangChain 与 ChatGLM、Qwen、Llama 等模型的 RAG + Agent 知识库问答项目。可直接部署使用。

### 6. NirDiamant/RAG_Techniques
- **地址**：https://github.com/NirDiamant/RAG_Techniques
- **语言**：Jupyter Notebook
- **说明**：50+ 种高级 RAG 技术教程，每个技术都有详细 Notebook 示例，含 LangChain 实现代码。RAG 进阶必看。

### 7. NirDiamant/GenAI_Agents
- **地址**：https://github.com/NirDiamant/GenAI_Agents
- **语言**：Jupyter Notebook
- **说明**：50+ Agent 教程，从基础对话机器人到复杂多Agent系统全覆盖，代码可复现。

### 8. 1Panel-dev/MaxKB
- **地址**：https://github.com/1Panel-dev/MaxKB
- **语言**：Python
- **说明**：企业级开源智能体平台，支持 RAG、Agent、多模型，生产可用。

### 9. BerriAI/litellm
- **地址**：https://github.com/BerriAI/litellm
- **语言**：Python
- **说明**：统一接口调用 100+ LLM API（OpenAI、Azure、Bedrock、HuggingFace 等），含费用追踪、负载均衡，是 LangChain 项目中调用 LLM 的最佳拍档。

---

## 三、辅助生态项目

| 项目 | 地址 | 说明 |
|------|------|------|
| langfuse/langfuse | https://github.com/langfuse/langfuse | LLM 可观测性平台，追踪、评估、监控 |
| mlflow/mlflow | https://github.com/mlflow/mlflow | AI 工程平台，LangChain 集成支持 |
| cometchat/opik | https://github.com/comet-ml/opik | LLM 调试、评估、监控工具 |

---

## 四、视频教程推荐

> 建议在 YouTube / Bilibili 搜索以下关键词

### YouTube 搜索关键词
- `LangChain 1.0 tutorial real world project 2024 2025`
- `LangChain LangGraph production app deployment`
- `LangChain agent tutorial for beginners`

### Bilibili 搜索关键词
- `LangChain 实战`
- `LangChain RAG 项目`
- `LangChain Agent 教程`

### 推荐频道
| 频道 | 内容特点 |
|------|---------|
| LangChain Official | 官方最新特性讲解 |
| Patrick Lee（GPTBots）| 实战向，RAG + Agent |
| Jay Feng（RAG系列）| 讲解细致，代码可复现 |

---

## 五、学习路径建议

```
第一阶段：官方入门
  langchain-ai/langchain → README + Quickstart

第二阶段：核心场景
  NirDiamant/RAG_Techniques  → 掌握 RAG 全流程
  NirDiamant/GenAI_Agents   → 掌握 Agent 模式

第三阶段：生产级项目
  FlowiseAI/Flowise         → 可视化编排
  langchain-ai/deepagents   → 官方最佳实践
  chatchat-space/Langchain-Chatchat → 中文 RAG 落地参考
```

---

## 六、核心概念速查

### LangChain 组件
- **Models**：支持 OpenAI、Anthropic、HuggingFace 等
- **Prompts**：提示词模板化管理
- **Chains**：将多个组件串联成工作流
- **Agents**：让 LLM 自主决策调用工具
- **Memory**：对话历史与状态管理
- **Callbacks**：钩子机制，记录日志、监控

### LangChain vs LangGraph
- **LangChain**：通用链式调用框架
- **LangGraph**：基于图的精细化流编排，支持循环和状态，适合复杂 Agent

---

> 笔记整理自 GitHub Topics / Search，持续更新中。
