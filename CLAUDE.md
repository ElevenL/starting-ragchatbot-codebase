# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

这是一个基于 Retrieval-Augmented Generation (RAG) 的课程材料问答系统。系统使用 ChromaDB 进行向量存储，Anthropic Claude 进行 AI 回答生成，并提供 Web 界面进行交互。

## 文档格式

课程文档遵循以下格式（见 `docs/` 目录示例）：

```
Course Title: [课程标题]
Course Link: [课程链接]
Course Instructor: [讲师姓名]

Lesson 0: [课时标题]
Lesson Link: [课时链接]
[课时内容...]

Lesson 1: [课时标题]
[课时内容...]
```

## 常用命令

### 安装依赖
```bash
uv sync
```

### 运行应用
```bash
# 使用启动脚本
./run.sh

# 或手动启动
cd backend
uv run uvicorn app:app --reload --port 8000
```

### 访问地址
- Web 界面: `http://localhost:8000`
- API 文档: `http://localhost:8000/docs`

## 架构概览

### 核心组件

**RAGSystem** (`backend/rag_system.py`) - 主协调器
- 初始化并管理所有核心组件
- 提供文档添加和查询功能
- 支持批量导入课程文件夹，包含重复检测避免重新处理

**VectorStore** (`backend/vector_store.py`) - 向量存储
- 基于 ChromaDB 实现
- 维护两个集合：
  - `course_catalog`：课程元数据，用于语义匹配课程名称
  - `course_content`：课程内容分块，用于实际内容搜索
- 使用 Sentence Transformers (`all-MiniLM-L6-v2`) 进行文本嵌入
- `search()` 方法统一处理课程名称解析、课时过滤和语义搜索

**DocumentProcessor** (`backend/document_processor.py`) - 文档处理
- 处理课程文档提取结构化信息（课程标题、课时等）
- 基于句子重叠的智能分块（800 字符，重叠 100 字符）
- 支持 `.pdf`, `.docx`, `.txt` 格式
- 句子分割使用正则表达式智能处理缩写，避免在 "e.g."、"Dr." 等处错误分割

**AIGenerator** (`backend/ai_generator.py`) - AI 生成器
- 集成 Anthropic Claude API
- 使用工具增强的提示模板
- 管理对话历史上下文

**SessionManager** (`backend/session_manager.py`) - 会话管理
- 保留最近的 `MAX_HISTORY` 条消息作为上下文（默认 2 条）
- 自动创建新会话（格式：`session_N`）
- 对话历史以字符串格式提供给 AI（`User: ... \nAssistant: ...`）

**ToolManager/CourseSearchTool** (`backend/search_tools.py`) - 搜索工具
- 实现基于工具的课程搜索，供 AI 调用
- 支持课程名称模糊匹配和特定课时内容过滤
- 提供语义搜索功能

**工具调用流程**：AI 根据问题类型决定是否使用 `search_course_content` 工具，工具调用结果会回传给 AI 生成最终回答。工具使用次数限制为每个查询最多一次。

### 应用启动流程

1. FastAPI 应用启动 (`backend/app.py`)
2. 初始化 `RAGSystem`（自动初始化所有核心组件）
3. 触发 `startup_event`，从 `../docs` 目录加载所有课程文档
4. ChromaDB 数据持久化到 `backend/chroma_db/` 目录

### API 端点

- `POST /api/query` - 处理查询，返回回答和来源
- `GET /api/courses` - 获取课程统计（总数和标题列表）

## 配置说明

配置位于 `backend/config.py`：

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `ANTHROPIC_API_KEY` | 从环境变量读取 | Claude API 密钥，必需 |
| `ANTHROPIC_MODEL` | `claude-sonnet-4-20250514` | 使用的 Claude 模型 |
| `EMBEDDING_MODEL` | `all-MiniLM-L6-v2` | 文本嵌入模型 |
| `CHUNK_SIZE` | 800 | 文本分块字符数 |
| `CHUNK_OVERLAP` | 100 | 分块重叠字符数 |
| `MAX_RESULTS` | 5 | 最大搜索结果数 |
| `MAX_HISTORY` | 2 | 对话历史保留消息数 |
| `CHROMA_PATH` | `./chroma_db` | ChromaDB 存储路径 |

## 开发注意事项

1. **环境变量**: 必须在 `.env` 文件中设置 `ANTHROPIC_API_KEY`
2. **数据持久化**: ChromaDB 数据存储在 `backend/chroma_db/`，修改此目录会丢失向量数据
3. **文档更新**: 将新文档放入 `docs/` 目录后重启应用即可自动加载
4. **重复检测**: 系统根据课程标题自动检测重复课程，不会重复处理
5. **Windows 用户**: 需使用 Git Bash 运行命令
6. **Python 版本**: 需要 Python 3.13 或更高版本
7. **API 模型**: 默认使用 `claude-sonnet-4-20250514`，如有更新需同步修改 `config.py`

## 数据模型

核心数据结构位于 `backend/models.py`：

- `Course`: 课程元数据（title, course_link, instructor, lessons[]）
- `Lesson`: 课时信息（lesson_number, title, lesson_link）
- `CourseChunk`: 向量存储的分块（content, course_title, lesson_number, chunk_index）
