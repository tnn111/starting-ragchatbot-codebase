# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

### Running the Application
```bash
# Quick start (recommended)
./run.sh

# Manual start
cd backend && uv run uvicorn app:app --reload --port 8000
```

### Package Management
```bash
# Install dependencies
uv sync

# Add new dependency
uv add package_name
```

### Environment Setup
- Create `.env` file with `ANTHROPIC_API_KEY=your_key_here`
- Requires Python 3.13+, uv package manager

## Architecture Overview

This is a **Retrieval-Augmented Generation (RAG) system** for querying course materials. The architecture follows a **tool-based approach** where Claude decides when to search the knowledge base.

### Core Flow
1. **User Query** → Frontend (JavaScript) → FastAPI → RAG System
2. **RAG System** → AI Generator (Claude API) → Tool Manager (if search needed)
3. **Tool Manager** → Vector Store (ChromaDB) → Formatted Results
4. **Response** flows back through the stack with source attribution

### Key Components

**Backend (`backend/`)**:
- `app.py` - FastAPI server with `/api/query` and `/api/courses` endpoints
- `rag_system.py` - **Main orchestrator** that coordinates all components
- `ai_generator.py` - Claude API integration with tool execution handling
- `search_tools.py` - **Tool system** where Claude calls `search_course_content` tool
- `vector_store.py` - ChromaDB interface with semantic search
- `document_processor.py` - Converts course docs to searchable chunks
- `session_manager.py` - Conversation context management
- `models.py` - Pydantic data models (Course, Lesson, CourseChunk)
- `config.py` - Centralized configuration with environment variables

**Frontend (`frontend/`)**:
- Simple HTML/CSS/JS interface that POSTs to `/api/query`
- Displays responses with markdown formatting and collapsible sources

**Data (`docs/`)**:
- Course documents with structured format: Title, Link, Instructor, Lessons
- Automatically loaded on startup and processed into vector embeddings

### Document Processing Pipeline
1. **Parse Structure**: Extract course metadata and lesson boundaries
2. **Chunk Content**: Sentence-aware chunking with configurable overlap (`CHUNK_SIZE: 800`, `CHUNK_OVERLAP: 100`)
3. **Add Context**: Prefix chunks with course/lesson information for better retrieval
4. **Store Vectors**: ChromaDB collections with embeddings via `all-MiniLM-L6-v2`

### Tool-Based Search Pattern
- Claude receives user query and decides whether to use `search_course_content` tool
- Tool supports filters: `course_name` (fuzzy matching) and `lesson_number`
- Search results include source attribution automatically tracked for UI display
- **One search per query maximum** enforced by system prompt

### Configuration
- All settings in `backend/config.py` using dataclass pattern
- ChromaDB path: `./chroma_db` (auto-created)
- Anthropic model: `claude-sonnet-4-20250514`
- Conversation history: 2 messages retained per session

### Session Management
- UUID-based sessions for conversation continuity
- Session created automatically if not provided
- History maintained for context-aware responses

### Key File Interactions
- `app.py` coordinates FastAPI → `rag_system.py` → `ai_generator.py`
- `ai_generator.py` handles Claude API and delegates tools to `search_tools.py`
- `search_tools.py` uses `vector_store.py` for actual retrieval
- `document_processor.py` populates `vector_store.py` on startup via `rag_system.py`