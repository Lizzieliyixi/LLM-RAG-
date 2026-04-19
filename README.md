# 个人VibeCoding LLM+RAG的智能问答系统（本地部署，可直接下载使用）

一个基于 Flask 和 LangChain 的轻量级 RAG 应用，支持多模态文档上传、用于个人轻量化知识库管理、智能问答。

## 功能特性

- 📄 支持多种文档格式上传（PDF、DOCX、TXT、MD）
- 🗄️ 支持本地存储和 MinIO 对象存储
- 🔍 支持多种向量数据库（Chroma、Milvus，我这里用的Chroma,pip
install chromadb即可，轻量级应用型，企业项目用的是Milvus）
- 🤖 支持多种 LLM 模型（DeepSeek、OpenAI、Ollama）
- 💬 智能问答和对话管理
- 🔐 用户认证和权限管理
- 📊 知识库管理和文档检索

## 环境要求

- Python 3.13 或更高版本
- MySQL 5.7+ 或 8.0+
- uv 包管理器（推荐）
- Docker 和 Docker Compose（可选，用于 Milvus）

## 快速启动

### 1. 安装依赖

#### 使用 uv（推荐）

```bash
# 安装 uv（如果尚未安装，需要先下载miniconda或者python然后全局安装）
pip install uv

# 使用 uv 安装依赖
uv sync
```

### 2. 配置数据库

#### 使用本地 MySQL
（安装地址及教程推荐：https://dev.mysql.com/downloads/mysql；
https://blog.csdn.net/yellow1019/article/details/134616848）

```bash
# 登录 MySQL，前往bin路径下admin运行powershell
（例如我的mysql下载的msi文件放在路径：D:\mysql\mysql-8.0.45-winx64（可根据你的情况而定））
-->在D:\mysql\mysql-8.0.45-winx64\bin下运行
.\mysql -u root -p

# 初始化创建数据库
CREATE DATABASE rag CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

# 创建用户（可选）
CREATE USER 'rag_user'@'localhost' IDENTIFIED BY 'your_password';
GRANT ALL PRIVILEGES ON rag.* TO 'rag_user'@'localhost';
FLUSH PRIVILEGES;
```


```

### 3. 启动向量数据库

#### 使用 Chroma（默认选项，你也可以选择Milvus Lite，无需额外配置）

Chroma 会自动在本地创建，无需额外配置。

#### 使用 Milvus（需要自行配置）

```bash
# 启动 Milvus 及其依赖服务
docker-compose up -d

# 检查服务状态
docker-compose ps
```

### 4. 启动应用

uv run main.py
```

启动成功后，你会看到类似以下的输出：

```
INFO - 正在启动RAG服务器在0.0.0.0:5000
 * Running on http://0.0.0.0:5000
```

### 5. 访问应用

在浏览器中打开：`http://localhost:5000` 就可以进入知识库啦~

## 项目结构

```
├── app/                    # 应用主目录
│   ├── blueprints/        # Flask 蓝图
│   │   ├── auth.py       # 认证相关路由
│   │   ├── chat.py       # 聊天相关路由
│   │   ├── document.py   # 文档相关路由
│   │   ├── knowledgebase.py  # 知识库相关路由
│   │   ├── settings.py   # 设置相关路由
│   │   └── utils.py      # 工具路由
│   ├── models/           # 数据模型
│   ├── services/         # 业务逻辑层
│   ├── templates/        # HTML 模板
│   └── utils/           # 工具函数
├── chroma_db/           # Chroma 向量数据库存储
├── logs/               # 日志文件
├── storages/           # 文件存储目录
├── main.py            # 应用入口
├── pyproject.toml     # 项目配置
└── docker-compose.yml # Docker Compose 配置
```



## 许可证

MIT License
