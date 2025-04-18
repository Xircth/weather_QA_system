# 📝 兴之助系统技术方案文档

---

## 一、🎯 需求拆解

### **1.1 核心需求**
- 基于本地GPU(AMD或其他，通过 `**device_map='auto'**`)部署大模型服务。
- 集成函数调用能力（如天气查询）。
- 构建Langchain风格的中间件处理业务逻辑和工具调用。
- 设计 **Streamlit** 前端界面进行交互。
- 基于简历或其他文档构建 **RAG** 知识库。

### **1.2 技术约束**
- **硬件**：支持本地CPU/GPU推理。
- **模型大小**：优先选用轻量级模型（<7B）。
- **技术栈**：`Langchine+Transformers+LoRA+RAG+Embedding+Streamlit`
- **开发风格**：Python面向对象，代码简洁高效。
- **系统定位**：功能演示，具备良好的扩展性。

---

## 二、🤖 模型选择

### **2.1 大模型选择**

| 模型                  | 参数量 | ROCm支持 | 中文能力 | 推理速度 | 最终选择 |
| ----------------------- | ------ | -------- | -------- | -------- | -------- |
| **Qwen2.5-0.5B-Instruct** | 0.5B   | ✅       | 良好     | 极快     | ✅       |
| Qwen2.5-1.5B-Instruct | 1.5B   | ✅       | 优秀     | 极快     | 备选     |
| Qwen2.5-7B-Instruct   | 7B     | ✅       | 优秀     | 快       | 备选     |
| OpenChat-3.5          | 7B     | ✅       | 优秀     | 快       | 备选     |

**选择理由**：
> - **Qwen2.5-0.5B-Instruct** 作为最终选择，因其参数量小（0.5B），硬件要求低，推理速度快，适合本地部署。
> - 该模型通过专家模型蒸馏，保留了较好的中文理解和生成能力，并兼容 `transformers` 库。
> - 使用 `**device_map="auto"**` 结合 `**accelerate**` 库可自动利用可用硬件（CPU/GPU），无需强制指定CUDA。
> - 指令模型包含对话模板，易于集成到聊天应用中。

### **2.2 嵌入模型选择**

| 模型                  | 维度 | 多语言   | 推理速度 | 最终选择 |
| --------------------- | ---- | -------- | -------- | -------- |
| **bge-small-zh-v1.5**     | 384  | 中文优先 | 极快     | ✅       |
| text2vec-base-chinese | 768  | 中文优先 | 快       | 备选     |
| m3e-small             | 384  | 中英双语 | 极快     | 备选     |

**选择理由**：
> - **bge-small-zh-v1.5** 作为最终选择，因其维度低（384），资源占用少，检索速度快。
> - 该模型在中文检索任务上表现优异，与**RAG**技术兼容性好。

---

## 三、🔧 模型微调

### **3.1 LoRA方案设计**
- 项目包含独立的模型微调脚本（`scripts/finetune.py`），采用 **LoRA**（Low-Rank Adaptation）技术对基础大模型进行微调。
- 微调目标主要是增强模型在特定任务上的表现，例如理解并生成函数调用指令。
- **LoRA** 配置（如秩 `r`、缩放因子 `alpha`、目标模块等）可在脚本中调整。

### **3.2 训练数据构建**
- 针对函数调用等特定任务，构建了JSON格式的训练数据集。
- 数据集包含`instruction`（任务描述）、`input`（用户输入示例）、`output`（期望的模型输出，如函数调用JSON）字段。
- 训练集会包含正面示例（需要调用函数）和负面示例（无需调用），覆盖多种表达方式。

---

## 四、⚙️ 功能模块设计

### **4.1 整体架构**

**架构说明**：
- 采用模块化设计，主要组件包括前端UI、应用控制器、中间件、LLM服务、RAG模块和工具集。
- 各模块间通过接口调用进行通信，实现功能解耦。
- 系统流程：用户请求 → **Streamlit UI** → **应用控制器 (`qa_system.py`)** → **中间件 (`middleware.py`)** → **LLM服务 (`llm_service.py`)** / **RAG模块 (`resume_rag.py`)** / **工具 (`tools.py`)** → 结果返回。

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│                 │    │                 │    │                 │
│  Streamlit UI   │───▶│ Langchain Agent │───▶│   LLM Service   │
│    (app.py)     │    │ (middleware.py) │    │ (llm_service.py)│
│                 │    │                 │    │                 │
└─────────────────┘    └─────────────────┘    └─────────────────┘
        │                      │                      │
        ▼                      ▼                      ▼
┌─────────────────┐    ┌─────────────┐      ┌─────────────────┐
│ QA System       │    │   Tools     │      │  Vector Store   │
│ (qa_system.py)  │    │ (tools.py)  │      │  (Resume RAG)   │
└─────────────────┘    └─────────────┘      └─────────────────┘
```

### **4.2 LLM服务模块 (`llm_service.py`)**

**功能作用**：
- 负责大模型的加载、管理和推理。
- 使用 `transformers` 库加载模型和分词器，利用 `**device_map="auto"**` 和 `**accelerate**` 实现硬件自动分配。
- 根据不同的任务类型（通用问答、RAG、天气提示等）格式化输入Prompt。
- 调用模型生成回复，并对输出进行基础处理（如去除特殊标记）。
- 封装模型推理的核心逻辑，提供统一的 `generate_response` 接口。

### **4.3 Langchain中间件 (`middleware.py`)**

**功能作用**：
- 作为核心业务逻辑处理层，连接应用控制器和底层服务。
- 接收用户查询和历史记录，判断处理流程。
- 优先处理带有 **RAG** 上下文的查询，将其传递给LLM服务进行回答生成。
- 对于通用查询，调用LLM判断是否需要使用工具（如天气查询）或是否需要转至 **RAG** 模式。
- 注册和查找可用工具（如 `tools.py` 中的 `get_weather`）。
- 调用工具执行，并将工具结果与原始查询结合，再次调用LLM生成最终回复。
- 管理和格式化与工具交互的逻辑。

### **4.4 简历RAG知识库 (`resume_rag.py`)**

**功能作用**：
- 负责构建、加载和查询基于简历或其他文档的向量知识库。
- 初始化Hugging Face嵌入模型（如 **bge-small-zh-v1.5**）和 **FAISS** 向量存储。
- 提供处理文本文件和图片文件（通过 **OCR**）以提取内容的功能。
- 使用文本分割器（如`RecursiveCharacterTextSplitter`）将文档切块。
- 将文本块向量化并存入 **FAISS** 索引，支持本地加载和保存。
- 提供`search`方法，根据用户查询进行语义相似度检索，返回相关文档块。

### **4.5 应用入口与控制器 (`app.py`, `qa_system.py`)**

**功能作用**：
- `app.py` 作为 **Streamlit** 应用的入口，负责页面初始化、UI布局和基本交互。
- `qa_system.py` 作为后端的核心控制器（单例模式实现）。
- 初始化并持有LLM服务、中间件、RAG模块等实例。
- 加载固定问答数据 (`fixed_qa.json`)，并提供基于相似度匹配的快速回答机制。
- `process_query` 是主要的查询处理入口，根据是否启用 **RAG** 模式决定处理流程：
    - **RAG模式下**：先进行向量检索，然后检查固定问答，若未命中则调用中间件处理（传入RAG上下文）。
    - **非RAG模式下**：检查固定问答，若未命中则直接调用中间件处理。
- 提供文件上传接口（通过 `pages/_common_elements.py` 实现UI），调用 **RAG** 模块构建知识库。
- 管理会话状态（通过Streamlit的 `session_state`）。

---

## 五、🎨 前端设计需求

### **5.1 整体界面风格**
- **色彩方案**：以深色模式为主，采用科技蓝或暗灰色调，搭配对比度适中的文本颜色。
- **设计风格**：简约现代，注重信息的可读性和交互的流畅性。
- **布局**：采用多页面应用（MPA）结构，侧边栏导航，主区域显示内容。

### **5.2 功能区设计**
1.  **侧边导航栏 (Sidebar)**：
    *   应用Logo或名称。
    *   页面切换：提供“普通问答”、“简历问答”等主要功能页面的入口。
    *   可能包含全局设置或历史会话管理入口。
2.  **主聊天区域 (Main Area - 各页面)**：
    *   **普通问答页**:
        *   显示聊天历史记录，区分用户和AI消息气泡。
        *   支持Markdown格式渲染AI回复。
        *   底部包含文本输入框和发送按钮。
    *   **简历问答页**:
        *   包含文件上传区域（支持TXT, PDF, DOCX, 图片）。
        *   处理按钮，用于触发简历解析和知识库构建。
        *   构建完成后，展现与普通问答类似的聊天界面，但查询会基于简历知识库。
3.  **通用元素**:
    *   加载状态提示（如模型加载、知识库构建时）。
    *   错误信息反馈。

### **5.3 交互设计**
- 输入框支持回车发送。
- AI回复时可以考虑流式输出（打字机效果）以提升体验。
- 文件上传后有明确的状态反馈（处理中、成功、失败）。
- 聊天记录可滚动。

### **5.4 界面原型示意**
```
┌────────────────────────────────────────────────────────────┐
│ ✨ 兴之助                             [普通问答▼] [简历问答]│
├────────────┬───────────────────────────────────────────────┤
│            │                                               │
│ (侧边栏)   │  ┌───────────────────────────────────────────┐│
│            │  │ Sys: 您好！我是兴之助，有什么可以帮您？     ││
│ 页面导航    │  └───────────────────────────────────────────┘│
│            │                                               │
│ 文件上传区  │  ┌───────────────────────────────────────────┐│
│ (简历问答页)│  │ User: 介绍一下你的项目经历？                ││
│            │  └───────────────────────────────────────────┘│
│ ...        │                                               │
│            │  ┌───────────────────────────────────────────┐│
│            │  │ Sys: 我参与了... (Markdown格式回复)        ││
│            │  └───────────────────────────────────────────┘│
│            │                                               │
│            │  ┌───────────────────────────────────────────┐│
│            │  │ 请输入您的问题...                           ││
│            │  └─────────────────────────────┬─────────────┘│
│            │                                │ [发送]      ││
└────────────┴────────────────────────────────┴─────────────┘
```

---

## 六、📜 项目规则

项目开发必须严格遵循以下规则：

- **代码风格与优化**：
  - 追求代码简洁高效，避免冗余。
  - 注释清晰明了，必要时添加，主要解释意图而非逐行翻译。
  - 关注代码性能，尤其是在模型推理、数据处理等关键路径。

- **变量与配置管理**：
  - 使用统一的配置文件（`config.json`）管理可变参数（如模型路径、API密钥、日志级别等）。
  - 通过`src/utils.py`中的`Config`类读取配置，避免硬编码。

- **国际化与本地化**：
  - 系统界面、提示信息、日志等优先使用中文。

- **代码修改规范**：
  - 修改代码前需理解其上下文及潜在影响。
  - 优先通过扩展而非修改核心逻辑来添加新功能。
  - 相关文档（如README）需同步更新。
