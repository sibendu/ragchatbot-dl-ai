# RAG Chatbot Architecture Documentation

## Complete Query Processing Flow: Frontend → Backend → Response

This document provides a detailed trace of how a user query flows through the RAG chatbot system, from user input to response display.

---

## Phase 1: Frontend - User Input Capture

**File: `frontend/script.js`**

### 1. User submits query (script.js:45-96)
- User types question and clicks send or presses Enter
- `sendMessage()` function is triggered

### 2. Input handling (script.js:46-52)
- Query text is trimmed and validated
- Input field is disabled to prevent duplicate submissions
- User message is added to chat UI

### 3. HTTP POST request (script.js:63-72)
```javascript
fetch(`${API_URL}/query`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
        query: query,
        session_id: currentSessionId  // null for first query
    })
})
```

---

## Phase 2: Backend - API Endpoint

**File: `backend/app.py`**

### 4. FastAPI receives request (app.py:56-74)
- Endpoint: `POST /api/query`
- Request validated against `QueryRequest` Pydantic model
- Creates new session_id if not provided (app.py:62-63)

### 5. Delegates to RAG System (app.py:66)
```python
answer, sources = rag_system.query(request.query, session_id)
```

---

## Phase 3: RAG System - Query Orchestration

**File: `backend/rag_system.py`**

### 6. RAG System processes query (rag_system.py:102-140)

#### a. Builds prompt (rag_system.py:114)
```python
prompt = f"""Answer this question about course materials: {query}"""
```

#### b. Retrieves conversation history (rag_system.py:116-119)
- If session exists, gets formatted history from SessionManager
- History includes up to 2 previous exchanges (configurable)

#### c. Calls AI Generator (rag_system.py:122-127)
```python
response = self.ai_generator.generate_response(
    query=prompt,
    conversation_history=history,
    tools=self.tool_manager.get_tool_definitions(),
    tool_manager=self.tool_manager
)
```

---

## Phase 4: AI Generation - Claude API Interaction

**File: `backend/ai_generator.py`**

### 7. First Claude API call (ai_generator.py:43-87)

#### a. Prepares system prompt (ai_generator.py:61-65)
- Uses static SYSTEM_PROMPT with instructions for tool usage
- Appends conversation history if available

#### b. Builds API parameters (ai_generator.py:68-77)
```python
api_params = {
    "model": "claude-sonnet-4-20250514",
    "temperature": 0,
    "max_tokens": 800,
    "messages": [{"role": "user", "content": query}],
    "system": system_content,
    "tools": tools,  # CourseSearchTool definition
    "tool_choice": {"type": "auto"}
}
```

#### c. Sends request to Claude (ai_generator.py:80)
```python
response = self.client.messages.create(**api_params)
```

### 8. Claude decides whether to use tools (ai_generator.py:83-87)
- If `stop_reason == "tool_use"`: Claude wants to search course content
- Otherwise: Claude answers directly from general knowledge

---

## Phase 5: Tool Execution (if Claude requests it)

**Files: `backend/ai_generator.py` → `backend/search_tools.py` → `backend/vector_store.py`**

### 9. Tool execution handler (ai_generator.py:89-135)

#### a. Extracts tool calls (ai_generator.py:108-120)
```python
for content_block in initial_response.content:
    if content_block.type == "tool_use":
        tool_result = tool_manager.execute_tool(
            content_block.name,  # "search_course_content"
            **content_block.input  # {query, course_name?, lesson_number?}
        )
```

### 10. ToolManager dispatches (search_tools.py:135-140)
- Routes to registered `CourseSearchTool`

### 11. CourseSearchTool executes (search_tools.py:52-86)

#### a. Calls vector store (search_tools.py:66-70)
```python
results = self.store.search(
    query=query,
    course_name=course_name,
    lesson_number=lesson_number
)
```

### 12. VectorStore semantic search (vector_store.py:61-100)

#### a. Resolves course name (vector_store.py:79-83)
- If course_name provided, uses semantic search on `course_catalog` collection
- Finds best matching course title (e.g., "MCP" → "Introduction to MCP")

#### b. Builds metadata filter (vector_store.py:86, 118-133)
```python
# Example filter for specific course and lesson
filter_dict = {"$and": [
    {"course_title": "Introduction to MCP"},
    {"lesson_number": 2}
]}
```

#### c. Queries ChromaDB (vector_store.py:93-98)
- Searches `course_content` collection with embeddings
- Uses SentenceTransformer (all-MiniLM-L6-v2) to embed query
- Performs cosine similarity search
- Returns top 5 most relevant chunks (configurable)

### 13. Formats search results (search_tools.py:88-114)
```
[Introduction to MCP - Lesson 2]
<chunk content...>

[Introduction to MCP - Lesson 3]
<chunk content...>
```
- Tracks sources for UI display (search_tools.py:112)

### 14. Returns tool results to AIGenerator (ai_generator.py:116-120)
- Wrapped in `tool_result` format for Claude API

### 15. Second Claude API call (ai_generator.py:127-135)
- Sends conversation with tool results back to Claude
- Claude synthesizes final answer using search results
- No tools included in this call

---

## Phase 6: Response Handling

**File: `backend/rag_system.py`**

### 16. RAG System post-processing (rag_system.py:129-140)

#### a. Extracts sources (rag_system.py:130)
```python
sources = self.tool_manager.get_last_sources()
```
- Gets sources tracked during tool execution (e.g., ["Introduction to MCP - Lesson 2"])

#### b. Updates conversation history (rag_system.py:136-137)
```python
self.session_manager.add_exchange(session_id, query, response)
```

#### c. Returns to API layer (rag_system.py:140)
```python
return response, sources  # (answer_text, [source_strings])
```

---

## Phase 7: API Response

**File: `backend/app.py`**

### 17. FastAPI formats response (app.py:68-72)
```python
return QueryResponse(
    answer=answer,           # Claude's synthesized answer
    sources=sources,         # List of course/lesson sources
    session_id=session_id    # For maintaining conversation
)
```

---

## Phase 8: Frontend - Display Response

**File: `frontend/script.js`**

### 18. JavaScript processes response (script.js:74-95)

#### a. Parses JSON (script.js:76)
```javascript
const data = await response.json()
```

#### b. Updates session ID (script.js:79-81)
- Stores session_id for subsequent queries

#### c. Removes loading indicator (script.js:84)

#### d. Renders answer (script.js:85)
```javascript
addMessage(data.answer, 'assistant', data.sources)
```

### 19. Message rendering (script.js:113-138)
- Converts markdown to HTML using Marked.js (script.js:120)
- Displays sources in collapsible section if present (script.js:124-131)
- Scrolls chat to bottom

---

## Summary Flow Diagram

```
User Input (Frontend)
    ↓
[1] JavaScript captures query + session_id
    ↓
[2] POST /api/query → FastAPI
    ↓
[3] RAGSystem.query()
    ├─ Get conversation history (SessionManager)
    └─ Call AIGenerator.generate_response()
        ↓
[4] First Claude API Call (with tools)
    └─ Claude decides: tool_use or direct answer
        ↓
[5] If tool_use → ToolManager.execute_tool()
    └─ CourseSearchTool.execute()
        └─ VectorStore.search()
            ├─ Resolve course name (semantic search on catalog)
            ├─ Build metadata filter
            └─ Query ChromaDB (semantic search on content)
                ↓
[6] Format search results → Return to AIGenerator
    ↓
[7] Second Claude API Call (with tool results)
    └─ Claude synthesizes final answer
        ↓
[8] Update conversation history (SessionManager)
    ↓
[9] Return QueryResponse(answer, sources, session_id)
    ↓
[10] JavaScript renders response
    └─ Display answer (markdown) + sources
```

---

## Key Components

### Frontend Components
- **script.js**: Handles user interaction, API communication, and UI updates
- **index.html**: Chat interface with sidebar for course statistics
- **style.css**: Dark theme styling with responsive layout

### Backend Components
- **app.py**: FastAPI application with REST endpoints
- **rag_system.py**: Main orchestrator coordinating all components
- **ai_generator.py**: Claude API wrapper with tool calling support
- **search_tools.py**: Tool definitions and execution management
- **vector_store.py**: ChromaDB interface for semantic search
- **session_manager.py**: Conversation history management
- **document_processor.py**: Document parsing and chunking
- **models.py**: Pydantic data models

### Data Flow
1. **User Query** → Frontend captures input
2. **HTTP Request** → POST to /api/query
3. **Session Management** → Retrieve or create session
4. **RAG Orchestration** → Coordinate search and generation
5. **AI Tool Calling** → Claude decides to search
6. **Vector Search** → ChromaDB semantic search
7. **Response Generation** → Claude synthesizes answer
8. **History Update** → Save conversation context
9. **API Response** → Return answer + sources
10. **UI Rendering** → Display with markdown formatting

---

## Key Features of the Flow

### 1. Session Management
- Maintains conversation context across queries
- Stores up to 2 previous exchanges (configurable via `MAX_HISTORY`)
- Session IDs tracked by frontend for continuity

### 2. Tool-Based Architecture
- Claude autonomously decides when to search course content
- Single tool: `search_course_content` with optional filters
- Supports general knowledge questions without searching

### 3. Semantic Search
- **Two-level search system**:
  - Course catalog: Semantic matching for course names
  - Course content: Similarity search on text chunks
- Uses SentenceTransformer embeddings (all-MiniLM-L6-v2)
- ChromaDB persistent vector store

### 4. Metadata Filtering
- Filter by course name (with fuzzy matching)
- Filter by lesson number
- Combined filters using AND logic

### 5. Source Tracking
- Tracks which courses/lessons were used
- Provides citations for transparency
- Displayed in collapsible UI section

### 6. Conversation History
- Formatted as "User: ... Assistant: ..." context
- Included in system prompt for Claude
- Limited to prevent token overflow

### 7. Error Handling
- Try-catch blocks throughout the stack
- Graceful degradation on search failures
- User-friendly error messages

---

## Configuration

### Key Settings (config.py)
- `ANTHROPIC_MODEL`: "claude-sonnet-4-20250514"
- `EMBEDDING_MODEL`: "all-MiniLM-L6-v2"
- `CHUNK_SIZE`: 800 characters
- `CHUNK_OVERLAP`: 100 characters
- `MAX_RESULTS`: 5 search results per query
- `MAX_HISTORY`: 2 conversation exchanges
- `CHROMA_PATH`: "./chroma_db"

### API Endpoints
- **POST /api/query**: Process user queries
- **GET /api/courses**: Get course statistics
- **GET /**: Serve static frontend files

---

## Technology Stack

### Backend
- **FastAPI**: Modern Python web framework
- **Anthropic Claude API**: AI generation with tool calling
- **ChromaDB**: Vector database for embeddings
- **Sentence Transformers**: Text embedding generation
- **Pydantic**: Data validation and models

### Frontend
- **Vanilla JavaScript**: No framework dependencies
- **Marked.js**: Markdown rendering
- **Fetch API**: HTTP requests
- **CSS3**: Modern styling with variables

### Data Storage
- **ChromaDB Collections**:
  - `course_catalog`: Course metadata for semantic matching
  - `course_content`: Text chunks with embeddings
- **Persistent storage**: Local file system

---

## Document Format

Course documents follow this structure:

```
Course Title: <Title>
Course Link: <URL>
Course Instructor: <Name>

Lesson 0: <Lesson Title>
Lesson Link: <URL>
<Lesson content...>

Lesson 1: <Lesson Title>
Lesson Link: <URL>
<Lesson content...>
```

This structured format enables:
- Extraction of course metadata
- Lesson-level organization
- Granular search filtering

---

## Performance Optimizations

1. **Static System Prompt**: Pre-built to avoid string operations
2. **Base API Parameters**: Reused across calls
3. **Efficient History Formatting**: Only when needed
4. **Limited Context Window**: Prevents token overflow
5. **Vector Search Caching**: ChromaDB's built-in caching
6. **Single Tool Limit**: AI system prompt restricts to one search per query

---

## Security Considerations

1. **API Key Management**: Stored in `.env` file (gitignored)
2. **CORS Configuration**: Allows all origins (development mode)
3. **Input Validation**: Pydantic models validate all requests
4. **Error Message Sanitization**: Generic errors to prevent information leakage
5. **Rate Limiting**: Not implemented (consider for production)

---

## Future Enhancement Opportunities

1. **Streaming Responses**: Real-time token streaming
2. **Multi-turn Tool Usage**: Allow multiple searches per query
3. **Caching Layer**: Redis for frequent queries
4. **Authentication**: User accounts and permissions
5. **Analytics**: Query logging and performance metrics
6. **File Upload**: Dynamic course document ingestion
7. **Advanced Filters**: Date ranges, content types, difficulty levels
8. **Feedback Loop**: User ratings to improve retrieval
