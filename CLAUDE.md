# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

一个用于查询课程材料的 RAG（检索增强生成）系统。使用 ChromaDB 进行向量存储，Anthropic Claude 进行 AI 生成，FastAPI 作为后端。

## 常用命令

```bash
# 运行应用
./run.sh
# 或手动运行：
cd backend && uv run uvicorn app:app --reload --port 8000

# 安装依赖
uv sync
```

访问地址：
- Web 界面: http://localhost:8000
- API 文档: http://localhost:8000/docs

## 架构

### 核心组件 (backend/)

| 组件 | 文件 | 职责 |
|------|------|------|
| RAGSystem | `rag_system.py` | 核心协调器，整合所有组件 |
| AIGenerator | `ai_generator.py` | Claude API 集成，支持工具调用 |
| VectorStore | `vector_store.py` | ChromaDB 封装，管理两个集合 |
| DocumentProcessor | `document_processor.py` | 解析课程文档，文本分块 |
| CourseSearchTool | `search_tools.py` | 定义 Claude 工具调用的搜索工具 |

### 数据流

**文档导入流程：**
```
docs/*.txt → DocumentProcessor.process_course_document()
          → VectorStore.add_course_metadata() + add_course_content()
          → ChromaDB (course_catalog + course_content 两个集合)
```

**查询流程（Agentic RAG）：**
```
前端 → POST /api/query → RAGSystem.query()
    → AIGenerator.generate_response(tools=...)
    → Claude 决策：是否需要搜索？
       是 → CourseSearchTool.execute() → VectorStore.search()
          → Claude 基于搜索结果生成最终答案
       否 → Claude 直接回答
    → 返回响应和来源
```

### ChromaDB 集合

- **course_catalog**: 课程元数据（标题、讲师、课时信息）
- **course_content**: 分块文本内容，用于语义搜索

### 配置 (config.py)

关键设置：
- `CHUNK_SIZE=800`, `CHUNK_OVERLAP=100` - 文本分块参数
- `MAX_RESULTS=5` - 搜索结果数量限制
- `MAX_HISTORY=2` - 对话上下文长度
- `ANTHROPIC_MODEL="claude-sonnet-4-20250514"`

### 文档格式要求

`docs/` 目录下的课程文档格式：
```
Course Title: [标题]
Course Link: [链接]
Course Instructor: [讲师]

Lesson 0: [课时标题]
Lesson Link: [课时链接]
[内容...]

Lesson 1: [课时标题]
...
```

### 工具调用模式

系统使用 Claude 的工具调用（非固定检索）。Claude 根据查询类型自主决定是否搜索：
- 课程相关问题 → 触发 `search_course_content` 工具
- 通用知识问题 → 直接回答，不搜索
