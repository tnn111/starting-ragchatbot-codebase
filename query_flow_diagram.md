# RAG System Query Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                FRONTEND                                         │
├─────────────────────────────────────────────────────────────────────────────────┤
│  User Interface (index.html)                                                   │
│  ┌─────────────────┐    ┌──────────────────┐    ┌─────────────────────────────┐ │
│  │   Chat Input    │    │   Send Button    │    │      Chat Messages         │ │
│  │   [text field]  │────│      [⏵]        │────│   [conversation history]   │ │
│  └─────────────────┘    └──────────────────┘    └─────────────────────────────┘ │
│                                   │                                             │
│  JavaScript (script.js)           │                                             │
│  ┌─────────────────────────────────▼─────────────────────────────────────────┐   │
│  │  sendMessage()                                                            │   │
│  │  • Validate input                                                         │   │
│  │  • Show loading animation                                                 │   │
│  │  • Disable UI                                                             │   │
│  │  • POST /api/query { query, session_id }                                 │   │
│  └─────────────────────────────────┬─────────────────────────────────────────┘   │
└─────────────────────────────────────┼─────────────────────────────────────────────┘
                                      │ HTTP Request
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                               BACKEND API                                      │
├─────────────────────────────────────────────────────────────────────────────────┤
│  FastAPI (app.py)                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────────┐ │
│  │  @app.post("/api/query")                                                   │ │
│  │  async def query_documents(request: QueryRequest):                         │ │
│  │  • Validate QueryRequest                                                   │ │
│  │  • Create session_id if needed                                             │ │
│  │  • Call rag_system.query(query, session_id)                               │ │
│  └─────────────────────────────┬───────────────────────────────────────────────┘ │
└─────────────────────────────────┼─────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                            RAG ORCHESTRATOR                                    │
├─────────────────────────────────────────────────────────────────────────────────┤
│  RAGSystem (rag_system.py)                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐ │
│  │  query(query, session_id):                                                 │ │
│  │  • Build prompt: "Answer this question about course materials: {query}"   │ │
│  │  • Get conversation history from session_manager                           │ │
│  │  • Call ai_generator.generate_response() with tools                        │ │
│  └─────────────┬───────────────────────────────┬───────────────────────────────┘ │
└─────────────────┼───────────────────────────────┼─────────────────────────────────┘
                  │                               │
                  ▼                               ▼
    ┌─────────────────────────────┐    ┌─────────────────────────────┐
    │     SESSION MANAGER         │    │      AI GENERATOR           │
    │   (session_manager.py)      │    │    (ai_generator.py)        │
    │                             │    │                             │
    │ • get_conversation_history()│    │ • generate_response()       │
    │ • add_exchange()            │    │ • Call Claude API           │
    │ • create_session()          │    │ • Handle tool execution     │
    └─────────────────────────────┘    └─────────────┬───────────────┘
                                                     │
                                                     ▼
                                      ┌─────────────────────────────┐
                                      │      CLAUDE API             │
                                      │   (Anthropic Service)       │
                                      │                             │
                                      │ • Process query & context   │
                                      │ • Decide on tool usage      │
                                      │ • Return response/tool_call │
                                      └─────────────┬───────────────┘
                                                    │
                                                    ▼ (if tool needed)
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              TOOL SYSTEM                                       │
├─────────────────────────────────────────────────────────────────────────────────┤
│  ToolManager (search_tools.py)                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────────┐ │
│  │  execute_tool("search_course_content", **kwargs):                          │ │
│  │  • Route to CourseSearchTool                                               │ │
│  │  • Execute search with parameters                                          │ │
│  │  • Track sources for UI                                                    │ │
│  └─────────────────────────────┬───────────────────────────────────────────────┘ │
└─────────────────────────────────┼─────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                            SEARCH ENGINE                                       │
├─────────────────────────────────────────────────────────────────────────────────┤
│  CourseSearchTool (search_tools.py)                                            │
│  ┌─────────────────────────────────────────────────────────────────────────────┐ │
│  │  execute(query, course_name?, lesson_number?):                             │ │
│  │  • Call vector_store.search() with filters                                 │ │
│  │  • Format results with course/lesson context                               │ │
│  │  • Store sources: ["Course Title - Lesson X", ...]                        │ │
│  └─────────────────────────────┬───────────────────────────────────────────────┘ │
└─────────────────────────────────┼─────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           VECTOR DATABASE                                      │
├─────────────────────────────────────────────────────────────────────────────────┤
│  VectorStore (vector_store.py)                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────────┐ │
│  │  search(query, course_name?, lesson_number?):                              │ │
│  │  • Generate embeddings for query                                           │ │
│  │  • Query ChromaDB collections                                              │ │
│  │  • Apply course/lesson filters                                             │ │
│  │  • Return SearchResults with metadata                                      │ │
│  └─────────────────────────────┬───────────────────────────────────────────────┘ │
│                                │                                               │
│  ┌─────────────────────────────▼───────────────────────────────────────────────┐ │
│  │                        ChromaDB                                            │ │
│  │  ┌─────────────────────┐  ┌─────────────────────┐  ┌────────────────────┐  │ │
│  │  │   course_metadata   │  │   course_content    │  │   Embeddings       │  │ │
│  │  │   collection        │  │   collection        │  │   (SentenceTransf) │  │ │
│  │  └─────────────────────┘  └─────────────────────┘  └────────────────────┘  │ │
│  └─────────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────────┘

                                  RESPONSE FLOW
                                      ▲
                                      │
              ┌───────────────────────┼───────────────────────┐
              │ 1. SearchResults      │ 6. Final Response     │
              │ 2. Formatted Results  │ 5. QueryResponse      │
              │ 3. Tool Results       │ 4. Generated Answer   │
              │    to Claude          │    + Sources          │
              └───────────────────────┼───────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          FRONTEND RESPONSE                                     │
├─────────────────────────────────────────────────────────────────────────────────┤
│  JavaScript (script.js)                                                        │
│  ┌─────────────────────────────────────────────────────────────────────────────┐ │
│  │  • Remove loading animation                                                 │ │
│  │  • Display response with markdown formatting                               │ │
│  │  • Show sources in collapsible section                                     │ │
│  │  • Update session_id for conversation continuity                           │ │
│  │  • Re-enable user input                                                    │ │
│  └─────────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────────┘

KEY DATA STRUCTURES:
┌─────────────────────────────────────┐  ┌─────────────────────────────────────┐
│         QueryRequest                │  │        QueryResponse                │
│  {                                  │  │  {                                  │
│    query: "What is RAG?",           │  │    answer: "RAG is...",             │
│    session_id: "uuid-123"           │  │    sources: ["Course1-Lesson2"],   │
│  }                                  │  │    session_id: "uuid-123"           │
└─────────────────────────────────────┘  └─────────────────────────────────────┘

┌─────────────────────────────────────┐  ┌─────────────────────────────────────┐
│         SearchResults               │  │         CourseChunk                 │
│  {                                  │  │  {                                  │
│    documents: ["content1", ...],    │  │    content: "Lesson content...",    │
│    metadata: [{"course": "..."}, ...],│    course_title: "Course Name",     │
│    distances: [0.1, 0.2, ...]       │  │    lesson_number: 1,                │
│  }                                  │  │    chunk_index: 0                   │
└─────────────────────────────────────┘  └─────────────────────────────────────┘
```

## Flow Summary

1. **User Input** → JavaScript captures and validates
2. **HTTP Request** → POST to `/api/query`
3. **API Validation** → FastAPI processes request
4. **RAG Orchestration** → Builds context and calls AI
5. **Claude Decision** → Determines if search is needed
6. **Tool Execution** → Searches vector database if required
7. **Vector Search** → ChromaDB semantic search with filters
8. **Result Formatting** → Structures response with sources
9. **AI Response** → Claude generates final answer
10. **HTTP Response** → Returns structured JSON
11. **UI Update** → Displays formatted response with sources