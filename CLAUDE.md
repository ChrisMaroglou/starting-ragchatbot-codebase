# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

### Running the Application
```bash
# Quick start with provided script (recommended)
chmod +x run.sh && ./run.sh

# Manual start
cd backend && uv run uvicorn app:app --reload --port 8000

# Install dependencies
uv sync
```

### Environment Setup
- Create `.env` file in root with: `ANTHROPIC_API_KEY=your_api_key_here`
- Ensure `docs/` directory exists for course materials

### Access Points
- Web Interface: http://localhost:8000
- API Documentation: http://localhost:8000/docs

## Architecture Overview

This is a **Course Materials RAG System** built with a tool-based architecture where Claude decides when to search vs. use existing knowledge.

### Core Design Patterns

**Tool-Based AI Architecture**: The system uses Anthropic's tool calling feature where Claude autonomously decides whether to search course content or answer from existing knowledge. This is implemented through:
- `search_tools.py`: Defines `CourseSearchTool` with semantic search capabilities
- `ai_generator.py`: Handles tool execution flow with Claude API
- `rag_system.py`: Orchestrates the entire query processing pipeline

**Two-Tier Vector Storage**: ChromaDB collections separate concerns:
- `course_catalog`: Course metadata (titles, instructors, lessons) for semantic course name resolution
- `course_content`: Actual lesson content chunks for retrieval

**Session-Based Conversation**: Maintains context across queries through `SessionManager` with configurable history limits.

### Key Data Flow

1. **Document Processing**: `DocumentProcessor` parses structured course files with expected format:
   - Line 1: `Course Title: [title]`
   - Line 2: `Course Link: [url]` 
   - Line 3: `Course Instructor: [instructor]`
   - Following: `Lesson N: [title]` markers with content

2. **Query Processing**: 
   - Frontend → FastAPI (`/api/query`) → RAG System → AI Generator
   - Claude uses tools when needed → Vector Store search → Response assembly
   - Sources tracked separately for UI display

3. **Semantic Search**: `VectorStore.search()` handles:
   - Course name resolution via semantic similarity
   - Lesson-specific filtering
   - ChromaDB metadata filtering with `$and` operators

### Component Responsibilities

- **`rag_system.py`**: Main orchestrator, coordinates all components
- **`ai_generator.py`**: Claude API integration, tool execution handler
- **`vector_store.py`**: ChromaDB operations, semantic search, course resolution
- **`search_tools.py`**: Tool definitions and result formatting
- **`document_processor.py`**: File parsing, text chunking (sentence-based with overlap)
- **`session_manager.py`**: Conversation history management
- **`models.py`**: Pydantic data models (Course, Lesson, CourseChunk)

### Configuration

All settings centralized in `config.py`:
- Chunk size: 800 chars with 100 char overlap
- Embedding model: `all-MiniLM-L6-v2`
- Claude model: `claude-sonnet-4-20250514`
- Max search results: 5, Max history: 2 exchanges

### ChromaDB Schema

**Course Catalog Collection**:
- Documents: Course titles (for semantic matching)
- Metadata: `title`, `instructor`, `course_link`, `lessons_json`, `lesson_count`
- IDs: Course titles (used as unique identifiers)

**Course Content Collection**:
- Documents: Lesson content chunks with context prefixes
- Metadata: `course_title`, `lesson_number`, `chunk_index`
- IDs: `{course_title}_{chunk_index}`

### Frontend Integration

- **API Endpoints**: 
  - `POST /api/query`: Query processing with session management
  - `GET /api/courses`: Course statistics and titles
- **Response Format**: `{answer, sources, session_id}`
- **Source Display**: Collapsible UI sections for citations
- **Session Handling**: Automatic session creation, conversation continuity

### Development Notes

- Course documents auto-load from `docs/` directory on startup
- Duplicate courses detected by title comparison during folder processing  
- Error handling throughout with graceful fallbacks
- CORS configured for development with hot reload
- ChromaDB persisted in `./chroma_db/` directory