# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a RAG (Retrieval-Augmented Generation) chatbot system for querying course materials. It uses ChromaDB for vector storage, Anthropic's Claude API with tool calling for intelligent responses, and provides a web interface for user interaction.

## Development Commands

### Setup and Installation
```bash
# Install uv package manager (if not installed)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Install all dependencies
uv sync

# Create .env file from template and add your ANTHROPIC_API_KEY
cp .env.example .env
```

### Running the Application
```bash
# Quick start (recommended)
./run.sh

# Manual start
cd backend
uv run uvicorn app:app --reload --port 8000
```

The server runs on http://localhost:8000 with auto-reload enabled for development.

### Testing and Development
```bash
# Run with specific Python version
uv run --python 3.13 uvicorn app:app --reload

# Clear ChromaDB and rebuild (delete chroma_db folder)
rm -rf backend/chroma_db
```

## Architecture Overview

### Core Data Flow

**Document Ingestion:**
1. `DocumentProcessor` parses course files (.txt, .pdf, .docx) from `docs/` folder
2. Extracts course metadata (title, instructor, link) from first 3 lines
3. Identifies lessons via "Lesson N: Title" markers
4. Chunks lesson content into 800-char segments with 100-char overlap
5. Creates `CourseChunk` objects with course/lesson context embedded

**Vector Storage:**
- Two ChromaDB collections: `course_catalog` (for course metadata) and `course_content` (for text chunks)
- Uses `all-MiniLM-L6-v2` embedding model for semantic search
- Chunks are prefixed with context: "Course [title] Lesson [N] content: [chunk]"

**Query Processing:**
1. User query → `RAGSystem.query()`
2. `AIGenerator` receives query with Claude tool definitions
3. Claude decides whether to call `search_course_content` tool
4. `CourseSearchTool` performs semantic search with optional course/lesson filters
5. Tool returns formatted results to Claude
6. Claude synthesizes final response using search results
7. Response + sources returned to frontend

### Key Components

**RAGSystem (rag_system.py):**
- Main orchestrator that wires together all components
- Manages document ingestion from `docs/` folder
- Handles query processing with conversation history
- Uses `ToolManager` to provide search capabilities to Claude

**DocumentProcessor (document_processor.py):**
- Expected format: Line 1: "Course Title: [title]", Line 2: "Course Link: [url]", Line 3: "Course Instructor: [name]"
- Lesson markers: "Lesson N: Title" followed by optional "Lesson Link: [url]"
- Implements sentence-based chunking with configurable size/overlap (see config.py)

**VectorStore (vector_store.py):**
- Wraps ChromaDB with two collections
- `add_course_metadata()`: Stores course titles for fuzzy matching
- `add_course_content()`: Stores text chunks with lesson metadata
- `search()`: Unified interface supporting course_name and lesson_number filters
- Automatically matches partial course names via semantic search on catalog

**AIGenerator (ai_generator.py):**
- Manages Claude API interactions with tool calling
- System prompt defines behavior as course materials assistant
- Handles multi-turn tool use: Claude can call search multiple times per query
- Returns final synthesized response after tool use completes

**CourseSearchTool (search_tools.py):**
- Implements Tool interface for Claude tool calling
- Supports fuzzy course name matching (e.g., "MCP" matches full course title)
- Optional lesson_number filter for specific lessons
- Tracks sources (course + lesson) for frontend display
- Returns formatted results with [Course - Lesson N] headers

**SessionManager (session_manager.py):**
- Maintains conversation history per session_id
- Limits history to MAX_HISTORY (default 2) exchanges
- Formats history for Claude context window

### Configuration (config.py)

All settings are centralized in `Config` dataclass:
- `ANTHROPIC_MODEL`: claude-sonnet-4-20250514
- `EMBEDDING_MODEL`: all-MiniLM-L6-v2
- `CHUNK_SIZE`: 800 chars (sentence-based, not hard cutoff)
- `CHUNK_OVERLAP`: 100 chars
- `MAX_RESULTS`: 5 search results
- `MAX_HISTORY`: 2 conversation exchanges
- `CHROMA_PATH`: ./chroma_db

### API Endpoints (app.py)

**POST /api/query**
- Input: `{query: string, session_id?: string}`
- Output: `{answer: string, sources: string[], session_id: string}`
- Creates new session if not provided

**GET /api/courses**
- Output: `{total_courses: int, course_titles: string[]}`
- Returns catalog statistics

**Startup Event:**
- Auto-loads all documents from `../docs` on server start
- Uses `clear_existing=False` to avoid re-processing existing courses
- Checks existing course titles in ChromaDB before adding new ones

### Frontend Architecture

Single-page app (frontend/index.html, script.js, style.css):
- Displays course statistics in sidebar
- Chat interface with markdown rendering (marked.js)
- Maintains session_id across conversation
- Shows sources as clickable chips below responses

## Development Notes

### Adding New Tools for Claude

1. Create class extending `Tool` in search_tools.py
2. Implement `get_tool_definition()` returning Anthropic tool schema
3. Implement `execute(**kwargs)` method
4. Register in RAGSystem.__init__: `self.tool_manager.register_tool(YourTool(...))`

### Document Format Requirements

Course files must follow this structure:
```
Course Title: [title]
Course Link: [url]
Course Instructor: [name]

Lesson 0: Introduction
Lesson Link: [url]
[lesson content...]

Lesson 1: Next Topic
[lesson content...]
```

Parsing is flexible - metadata lines can appear in first 4 lines in any order. If no "Lesson N:" markers found, entire content is treated as single document.

### ChromaDB Persistence

- ChromaDB data stored in `backend/chroma_db/`
- Delete this folder to force full rebuild from `docs/`
- On startup, system checks existing course titles to avoid duplicates
- Each course identified by title (used as unique key)

### Conversation History

- Session IDs generated via `SessionManager.create_session()` (UUID4)
- History includes user query + AI response pairs
- Limited to MAX_HISTORY (2) most recent exchanges
- Formatted as list of `{role: "user"|"assistant", content: string}` for Claude API

### CORS and Proxy Configuration

FastAPI configured with:
- `TrustedHostMiddleware` allowing all hosts
- `CORSMiddleware` with wildcard origins for development
- `DevStaticFiles` serves frontend with no-cache headers
- Root path set to "" for deployment flexibility
