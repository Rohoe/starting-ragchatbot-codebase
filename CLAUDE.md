# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a RAG (Retrieval-Augmented Generation) chatbot that answers questions about educational course materials. It uses ChromaDB for semantic search, Anthropic's Claude API with tool calling, and a vanilla JavaScript frontend.

## Important: Package Manager

**Always use `uv` for all Python operations:**
- **Dependency management**: Use `uv add`/`uv remove`/`uv sync` - NEVER use `pip install`/`pip uninstall`
- **Running Python files**: Use `uv run python file.py` - NEVER use `python file.py` directly
- **Running scripts**: Use `uv run <command>` - e.g., `uv run uvicorn app:app`

## Development Commands

### Running the Application
```bash
# Quick start (recommended)
./run.sh

# Manual start
cd backend && uv run uvicorn app:app --reload --port 8000
```

### Dependency Management

**IMPORTANT: Use `uv` to manage ALL dependencies. Do NOT use `pip install`, `pip uninstall`, or any pip commands.**

```bash
# Install/sync dependencies (DO NOT use pip install -r requirements.txt)
uv sync

# Add a new dependency (DO NOT use pip install package-name)
uv add package-name

# Remove a dependency (DO NOT use pip uninstall)
uv remove package-name

# Update dependencies
uv lock --upgrade

# Run Python commands (DO NOT use python directly without uv)
uv run python script.py
```

### Environment Setup
```bash
# Create .env file from template
cp .env.example .env
# Then add your ANTHROPIC_API_KEY
```

### Running Python Files

**CRITICAL: Always prefix Python commands with `uv run`**

```bash
# CORRECT - Running Python files
uv run python backend/app.py
uv run python -m pytest tests/

# WRONG - Do NOT run Python directly
python backend/app.py          # ❌ Will not use project dependencies
python -m pytest tests/        # ❌ Will not use project dependencies

# CORRECT - Running any command that needs dependencies
uv run uvicorn app:app
uv run pytest
uv run black .

# WRONG - Do NOT run commands directly
uvicorn app:app                # ❌ Will not find dependencies
pytest                         # ❌ Will not find dependencies
```

### Testing API Endpoints
```bash
# The FastAPI docs are auto-generated at:
# http://localhost:8000/docs
```

## Architecture Overview

### Request Flow: Two-Stage Claude API Pattern

The system makes **two sequential calls** to Claude API for each user query:

1. **First call (with tools)**: Claude receives the query and tool definitions, decides whether to use `search_course_content` tool
2. **Tool execution**: If Claude calls the tool, the system executes the semantic search in ChromaDB
3. **Second call (with results)**: Claude receives the search results and synthesizes the final answer

This pattern is implemented in `ai_generator.py:_handle_tool_execution()`.

### Three-Layer Storage Model

**1. Document Processing Layer** (`document_processor.py`)
- Parses course files with specific format (Course Title → Instructor → Lessons)
- Uses sentence-based chunking with overlap (800 chars, 100 overlap)
- Adds context prefixes to chunks (e.g., "Course X Lesson Y content: ...")

**2. Vector Store Layer** (`vector_store.py`)
- **Two ChromaDB collections**:
  - `course_catalog`: Stores course metadata for semantic course name resolution
  - `course_content`: Stores text chunks with embeddings for content search
- Smart course name matching: Partial names like "Anthropic" → full course title via semantic search
- Filtering support: Can filter by exact course title and/or lesson number

**3. RAG Orchestration Layer** (`rag_system.py`)
- Coordinates all components: document processor, vector store, AI generator, session manager
- Manages tool definitions and execution through `ToolManager`
- Handles conversation history (keeps last 2 Q&A exchanges via `session_manager.py`)

### Tool Calling Architecture

The system implements Anthropic's tool calling pattern:

- **Tool Definition** (`search_tools.py:27-50`): Defines the `search_course_content` tool schema
- **Tool Manager** (`search_tools.py:116-154`): Registry pattern for managing multiple tools
- **Tool Execution** (`search_tools.py:52-86`): Executes search, formats results, tracks sources
- **Source Tracking**: Tool stores sources in `last_sources`, which are retrieved by RAG system and sent to frontend

### Session Management

- Sessions are created on first query, persisted across requests via `session_id`
- History is formatted as string and injected into Claude's system prompt
- Automatic pruning: Only keeps last `MAX_HISTORY * 2` messages (default: 4 messages = 2 exchanges)
- See `session_manager.py` and `config.py:MAX_HISTORY`

## Key Configuration Points

All configuration is centralized in `backend/config.py`:

```python
ANTHROPIC_MODEL: str = "claude-sonnet-4-20250514"
EMBEDDING_MODEL: str = "all-MiniLM-L6-v2"  # Sentence-transformers model
CHUNK_SIZE: int = 800
CHUNK_OVERLAP: int = 100
MAX_RESULTS: int = 5        # Top-k search results
MAX_HISTORY: int = 2        # Conversation exchanges to remember
CHROMA_PATH: str = "./chroma_db"
```

## Document Format Requirements

Course documents in `docs/` must follow this structure:

```
Course Title: [title]
Course Link: [url]
Course Instructor: [name]

Lesson 0: [lesson title]
Lesson Link: [lesson url]
[lesson content...]

Lesson 1: [next lesson title]
...
```

Supported formats: `.txt`, `.pdf`, `.docx`

Processing happens automatically on server startup via `app.py:@app.on_event("startup")`.

## Data Flow Patterns

### Query Processing Pipeline

```
User Query (Frontend)
  ↓
FastAPI endpoint (/api/query)
  ↓
RAGSystem.query()
  ↓
Get conversation history from SessionManager
  ↓
AIGenerator.generate_response() [First Claude call with tools]
  ↓
Claude decides to call search_course_content
  ↓
ToolManager.execute_tool()
  ↓
CourseSearchTool.execute()
  ↓
VectorStore.search()
  ├─ Step 1: Resolve course name (semantic search in catalog)
  ├─ Step 2: Build filter (course + lesson)
  └─ Step 3: Search content (semantic search with embeddings)
  ↓
Format results with headers + track sources
  ↓
AIGenerator._handle_tool_execution() [Second Claude call with results]
  ↓
Claude synthesizes answer from search results
  ↓
RAGSystem gets sources from ToolManager
  ↓
SessionManager stores Q&A exchange
  ↓
Return (answer, sources, session_id) to Frontend
  ↓
Frontend renders markdown + displays sources
```

### Document Ingestion Pipeline

```
Startup event
  ↓
RAGSystem.add_course_folder("../docs")
  ↓
For each file:
  DocumentProcessor.process_course_document()
    ├─ Parse metadata (title, instructor, link)
    ├─ Split by lesson markers (Lesson X:)
    ├─ Chunk text with overlap (sentence-based)
    └─ Add context prefixes to chunks
  ↓
VectorStore.add_course_metadata(course)
  └─ Add to course_catalog collection
  ↓
VectorStore.add_course_content(chunks)
  └─ Add to course_content collection
  └─ Embeddings auto-generated by ChromaDB
```

## Important Implementation Details

### ChromaDB Persistence
- Database is stored in `backend/chroma_db/` (gitignored)
- Data persists across server restarts
- Duplicate prevention: System checks `get_existing_course_titles()` before adding courses
- To reset database: Delete `backend/chroma_db/` directory or call `vector_store.clear_all_data()`

### Frontend-Backend Communication
- Frontend uses relative paths (`/api/*`) to work with any host
- Session ID is returned on first query and stored in `currentSessionId`
- Sources are displayed in collapsible `<details>` element
- Markdown rendering via `marked.js` CDN library

### AI Generator Settings
- Temperature: 0 (deterministic responses)
- Max tokens: 800 (responses are concise)
- System prompt includes conversation history and tool usage instructions
- See `ai_generator.py:SYSTEM_PROMPT` for prompt engineering details

### Search Result Formatting
Results are formatted with course/lesson context:
```
[Course Title - Lesson 2]
[chunk content here]

[Course Title - Lesson 3]
[chunk content here]
```

This format helps Claude understand the source context when synthesizing answers.

## Common Modification Patterns

### Adding a New Tool
1. Create a class inheriting from `Tool` in `search_tools.py`
2. Implement `get_tool_definition()` and `execute()` methods
3. Register the tool in `RAGSystem.__init__()`: `self.tool_manager.register_tool(YourTool(...))`

### Changing Chunk Strategy
Modify `document_processor.py:chunk_text()` and update `config.py:CHUNK_SIZE` and `CHUNK_OVERLAP`.

### Adding New Course Fields
1. Update `models.py:Course` or `Lesson` Pydantic models
2. Update `document_processor.py:process_course_document()` parsing logic
3. Update `vector_store.py:add_course_metadata()` storage logic

### Modifying Conversation Context
Change `config.py:MAX_HISTORY` to adjust how many exchanges are kept. The system automatically prunes older messages.

## File Organization

```
backend/
├── app.py              # FastAPI app, endpoints, startup
├── rag_system.py       # Main orchestrator connecting all components
├── ai_generator.py     # Claude API client with tool handling
├── vector_store.py     # ChromaDB interface (two collections)
├── document_processor.py  # Parse & chunk course documents
├── search_tools.py     # Tool definitions & execution
├── session_manager.py  # Conversation history tracking
├── config.py           # Centralized configuration
└── models.py           # Pydantic data models

frontend/
├── index.html          # Chat UI structure
├── script.js           # API calls, message handling
└── style.css           # Responsive styling

docs/                   # Course materials (auto-loaded on startup)
```
