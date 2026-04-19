# Smart RAG v3 使用与架构说明（guidance）
---

## 一、在第一次部署完成后，以后如何启动（具体步骤）

### 1. 确保你的MySQL能正常运行：

1. 按 `Win + R`，输入 `services.msc` 回车，打开「服务」；
2. 找到 **MySQL** 或 **MySQL80** 之类名称的服务，状态为**正在运行**即可。

**命令行登录 MySQL**（建库、改表时）仍可用完整路径，例如：

```powershell
D:\mysql\mysql-8.0.45-winx64\bin\mysql.exe -u root -p
然后输入你设置好的root密码（我在源代码里默认设置的123456）
```

### 2. 启动本应用（每次开发/使用）

1. 确认 **MySQL 服务已运行**，且初始的数据库 `rag_v3` 已存在（你已创建过则不用重复建）。
2. 打开 **PowerShell** 或 **cmd**。
3. 进入项目根目录（与 `main.py`、`pyproject.toml` 同级）：

   ```powershell
   cd C:\Users\m7300\Desktop\smart_rag_v3
   ```

4. 启动 Flask 应用：

   ```powershell
   uv run main.py
   ```

5. 浏览器访问：**http://127.0.0.1:5000** 或 **http://localhost:5000**

### 3. 停止项目

在运行 `uv run main.py` 的终端里按 **Ctrl + C**。



## 二、产品解读：

**Summary**：
1. 这是基于 **Flask** 的 Web 应用，把用户上传的文档做 **解析 → 切片 → 向量化**，然后存入 **Chroma**；然后用户提问时做 **检索（向量 / 关键词 / 混合）**，把检索到的片段作为 **上下文**，交给 **大语言模型（LLM）** 流式生成回答，并尽量带上 **引用来源**的RAG框架的个人知识库。

**和「通用大模型聊天」的本质区别**：

- **普通聊天**：不调知识库，只用系统提示词 + LLM（`chat_service.chat_stream`）。
- **知识库问答（RAG）**：先按知识库检索文档块，再拼进提示词（`rag_service.ask_stream` → `chat_service.ask_stream`）。

---

## 三、RAG 主链路（从上传到回答）

```
上传文件 → 入库记录 → 触发「处理」→ 解析文本 → 切片 → Embedding 写入 Chroma
                                                              ↓
用户提问 ← 流式 LLM 回答 ← 拼 Prompt ← 检索（向量/BM25/混合）← 查询向量化
```

| RAG 阶段 | 本项目中的含义 | 主要代码位置 |
|----------|----------------|--------------|
| **文档接入** | 接收文件、校验类型与大小 | `document.py`（API）、`document_service.upload` |
| **文件存储** | 二进制存本地目录或 MinIO | `storage_service`、`local_storage` / MinIO |
| **解析（Load）** | PDF/DOCX/TXT/MD 抽成文本 | `parser_service` → `document_loader.py`（LangChain `PyPDFLoader` 等） |
| **切片（Chunk）** | 按知识库的 chunk_size / overlap 切分 | `text_splitter.py`（`RecursiveCharacterTextSplitter`） |
| **向量化（Embedding）** | 每个文本块变成向量 | `embedding_factory.py`；写入时由 `Chroma(..., embedding_function=...)` 触发 |
| **索引（Index）** | 向量 + 元数据持久化 | `vectordb/chroma.py` → `vectorstore.add_documents` |
| **检索（Retrieve）** | 根据问题找相关块 | `retrieval_service.py`（向量相似度 / BM25 / RRF 混合） |
| **重排（Rerank）** | 对候选文档再排序（可选） | `rerank_factory.py`；`retrieval_service` 里 `self.reranker` 当前为 **None**，默认**不走**重排 |
| **生成（Generate）** | 把检索结果拼成 context，流式调用 LLM | `rag_service.py`（`ChatPromptTemplate` + `chain.stream`） |
| **引用（Citation）** | 把命中的 chunk 元数据返回前端 | `rag_service._extract_citations` |

---

## 四、Chroma 向量库在本项目里「具体做了什么」？

- **配置入口**：`app/config.py` 中 `VECTOR_DB_TYPE = "chroma"`、`CHROMA_PERSIST_DIRECTORY`（默认项目下 `./chroma_db`）。
- **实现类**：`app/services/vectordb/chroma.py` 的 `ChromaVectorDB`。
- **集合（Collection）命名**：每个知识库一个集合，名称 **`kb_{知识库ID}`**（见 `document_service` 中 `collection_name = f"kb_{kb_id}"`）。
- **典型操作**：
  - **写入**：文档处理完成后 `vector_service.add_documents` → LangChain `Chroma.add_documents`（内部会对 `page_content` 做 **Embedding** 再存）。
  - **相似检索**：`retrieval_service.vector_search` → `similarity_search_with_score`（余弦空间在 `collection_metadata` 里配置了 `hnsw:space: cosine`）。
  - **关键词检索**：从同一 Chroma 集合 `get` 出**全量**文档与元数据，在内存里用 **jieba + BM25** 打分（**不是** Chroma 自带的全文引擎）。
  - **删除**：按 `doc_id` 等 metadata 查 id 再 `delete`（`delete_documents` 里对 Chroma API 的封装）。

**`vector_service`**（`vector_service.py`）只是 **工厂单例** 的别名，真实逻辑在 `vectordb` 包；换 **Milvus** 时改 `VECTOR_DB_TYPE` 即可走 `milvus.py`（需单独部署 Milvus）。

---

## 五、前后端怎么划分？

本项目是 **Flask 服务端渲染 + 少量 API + 模板内 JavaScript**，没有单独的 React/Vue 工程目录。

### 1. 算「前端 / UI」的部分

| 类型 | 路径 | 说明 |
|------|------|------|
| 页面 HTML | `app/templates/*.html` | 各功能页面骨架、表格、表单 |
| 公共布局 | `app/templates/base.html` | 导航栏、引用 Bootstrap 等 |
| 页面内脚本 | 各模板底部的 `<script>` | 调 `/api/v1/...`、处理 SSE 流式输出等 |

主要页面与功能对应关系：

| 模板文件 | 大致功能 |
|----------|----------|
| `home.html` | 首页 |
| `login.html` / `register.html` | 登录注册界面 |
| `kb_list.html` | 知识库列表 |
| `kb_detail.html` | 单个知识库详情、上传与文档列表 |
| `document_chunks.html` | 查看某文档的分块与向量侧信息 |
| `chat.html` | 聊天（含知识库选择与流式回答） |
| `settings.html` | LLM / Embedding / 检索参数等设置 |

**说明**：仓库中**没有**独立的 `app/static` 前端打包目录；样式与交互主要依赖模板里引用的 CDN（如 Bootstrap）和内联 JS。

### 2. 算「后端」的部分

| 类型 | 路径 | 说明 |
|------|------|------|
| 入口 | `main.py` | 启动 Flask |
| 应用工厂 | `app/__init__.py` | `create_app`、注册蓝图、`init_db()` |
| 配置 | `app/config.py` | 数据库、存储、向量库类型、默认 API 等 |
| 路由 / HTTP 层 | `app/blueprints/*.py` | 解析请求、调用 service、返回 HTML 或 JSON / SSE |
| 业务逻辑 | `app/services/*.py` | 文档、知识库、检索、RAG、聊天、会话等 |
| 向量存储抽象 | `app/services/vectordb/` | Chroma / Milvus 实现与工厂 |
| 工具与模型工厂 | `app/utils/*.py` | DB、Embedding、LLM、分词检索辅助等 |
| 数据模型 | `app/models/*.py` | SQLAlchemy 表结构（用户、知识库、文档、消息、设置等） |

---

## 六、是否使用 LangChain？用在什么地方？

**有，且集中在「文档抽象、向量存储、LLM 调用、Prompt」上。**

典型 import 与用途：

| 依赖方向 | 用途 |
|----------|------|
| `langchain_core` | `Document`、`ChatPromptTemplate`、与 LLM 的链式组合 `prompt \| llm` |
| `langchain_chroma` | `Chroma` 向量存储封装 |
| `langchain_community` | `PyPDFLoader`、`Docx2txtLoader`、`TextLoader`、`OllamaEmbeddings` 等 |
| `langchain_huggingface` | `HuggingFaceEmbeddings` |
| `langchain_openai` | `OpenAIEmbeddings` |
| `langchain_deepseek` / 等 | `ChatDeepSeek` 等聊天模型 |
| `langchain_text_splitters` | `RecursiveCharacterTextSplitter`（在 `TextSplitter` 中封装） |

**不是**「整段业务都交给 LangChain Agent 自动编排」，而是：**检索与数据流主要在自写 Service 里**，LangChain 负责 **文档类型、切分器、向量库适配、LLM 客户端、Prompt 模板** 等标准件。

---

## 七、按文件快速索引（想改什么去哪找）

### 1. 蓝图（路由）→ 用户操作

| 文件 | 典型路由/作用 |
|------|----------------|
| `auth.py` | 首页 `/` |
| `knowledgebase.py` | `/kb` 列表、`/api/v1/kb` 创建知识库等 |
| `document.py` | 文档上传 API、处理 API、分块展示页 |
| `chat.py` | `/chat` 页面；`/api/v1/chat` 普通聊天；`/api/v1/knowledgebases/<kb_id>/chat` **RAG 流式** |
| `settings.py` | `/settings` 页面；`/api/v1/settings` 读写配置 |

### 2. 服务层 → RAG 与数据

| 文件 | 职责摘要 |
|------|-----------|
| `document_service.py` | 上传、异步 `process`、解析+切片+写向量、删除时清向量与文件 |
| `retrieval_service.py` | 向量检索、BM25、混合 RRF、（可选）重排入口 |
| `rag_service.py` | 组检索结果、拼 context、`ChatPromptTemplate`、流式 LLM、引用 |
| `chat_service.py` | 普通聊天流式；`ask_stream` 转调 `rag_service` |
| `knowledgebase_service.py` | 知识库 CRUD 等 |
| `settings_service.py` | 全局设置（含默认 Embedding/LLM/检索模式） |
| `parser_service.py` | 转调 `DocumentLoader` |
| `vector_service.py` | 指向 `vectordb` 工厂单例 |

### 3. 工具与工厂

| 文件 | 职责 |
|------|------|
| `embedding_factory.py` | 按设置创建 HuggingFace / OpenAI / Ollama Embedding |
| `llm_factory.py` | 按设置创建 DeepSeek / OpenAI / Ollama 等 Chat 模型 |
| `text_splitter.py` | 切片策略与 chunk id 规则 |
| `document_loader.py` | 各格式加载为 LangChain `Document` 列表 |
| `db.py` | MySQL 连接、Session、`init_db` 建表 |
| `models_config.py` | 前端可选模型列表配置 |

---

## 八、数据存在哪里？（产品经理关心的「资产」）

| 资产 | 存放位置 |
|------|-----------|
| 业务元数据（用户、知识库、文档记录、会话、消息、设置） | **MySQL**（库名由 `config.py` 的 `DB_NAME` 决定） |
| 向量与文本块 | **Chroma 持久化目录**（默认 `./chroma_db`） |
| 上传的原始文件 | **本地** `./storages`（或 MinIO，若切换存储类型） |
| 日志 | `./logs`（见配置） |

---

## 九、可选增强与现状说明（避免误解）

1. **重排（Rerank）**：`rerank_factory.py` 已实现基于 CrossEncoder 的本地重排，但 `retrieval_service` 里 **`self.reranker = None`**，当前主线**不启用**重排。
2. **Milvus**：代码支持，需改配置并自行部署；默认 **Chroma** 即可跑通。
3. **混合检索**：设置里 `retrieval_mode = hybrid` 时，为 **向量 + BM25 结果 RRF 融合**，权重来自 `vector_weight`。

---

## 十、文档版本

- 本文档随仓库维护，便于新人快速建立「从启动到 RAG 各环节」的心智模型。
- 若你修改了 `config.py` 中的库名、端口或向量库类型，请同步更新自己的运维习惯与上述「数据存放」描述。
