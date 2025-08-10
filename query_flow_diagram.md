# RAG System Query Flow Diagram

```mermaid
sequenceDiagram
    participant U as User
    participant FE as Frontend<br/>(script.js)
    participant API as FastAPI<br/>(app.py)
    participant RAG as RAG System<br/>(rag_system.py)
    participant AI as AI Generator<br/>(ai_generator.py)
    participant TM as Tool Manager<br/>(search_tools.py)
    participant ST as Search Tool<br/>(CourseSearchTool)
    participant VS as Vector Store<br/>(vector_store.py)
    participant DB as ChromaDB<br/>(Embeddings)
    participant Claude as Anthropic<br/>Claude API

    %% User Input Phase
    U->>FE: Types query & clicks send
    FE->>FE: Disable input, show loading
    FE->>FE: Add user message to chat

    %% API Request Phase  
    FE->>API: POST /api/query<br/>{query, session_id}
    API->>API: Create session if needed
    API->>RAG: query(query, session_id)

    %% RAG Processing Phase
    RAG->>RAG: Build prompt & get history
    RAG->>AI: generate_response(query, history, tools)

    %% AI Generation Phase
    AI->>AI: Build system prompt + context
    AI->>Claude: messages.create(tools enabled)
    
    %% Tool Decision Point
    Claude->>AI: Response with tool_use
    AI->>AI: _handle_tool_execution()
    
    %% Tool Execution Phase
    AI->>TM: execute_tool("search_course_content", params)
    TM->>ST: execute(query, course_name, lesson_number)
    
    %% Vector Search Phase
    ST->>VS: search(query, course_name, lesson_number)
    VS->>VS: _resolve_course_name() if needed
    VS->>DB: query(embeddings, filters)
    DB->>VS: semantic search results
    VS->>ST: SearchResults(docs, metadata, distances)
    
    %% Results Formatting
    ST->>ST: _format_results() + store sources
    ST->>TM: formatted search results
    TM->>AI: tool execution results
    
    %% Final AI Response
    AI->>Claude: messages.create(with tool results)
    Claude->>AI: final response text
    AI->>RAG: response text
    
    %% Response Assembly
    RAG->>TM: get_last_sources()
    TM->>RAG: sources list
    RAG->>RAG: Update conversation history
    RAG->>API: (response, sources)
    
    %% API Response
    API->>FE: QueryResponse{answer, sources, session_id}
    
    %% Frontend Display
    FE->>FE: Remove loading animation
    FE->>FE: Render markdown response
    FE->>FE: Show collapsible sources
    FE->>FE: Re-enable input
    FE->>U: Display complete response

    %% Parallel Source Tracking
    Note over ST,TM: Sources tracked separately<br/>for UI citations
    Note over VS,DB: Semantic search with<br/>sentence transformers
    Note over AI,Claude: Tool-based architecture<br/>Claude decides when to search
```

## Flow Components Breakdown

### **Frontend Layer** 
- **User Interface**: Chat input, loading states, message display
- **API Communication**: HTTP POST to `/api/query`
- **Response Rendering**: Markdown parsing, source citations

### **API Layer**
- **FastAPI Endpoint**: Request validation, session management
- **Error Handling**: HTTP status codes, exception handling

### **RAG Orchestration**
- **Query Processing**: Prompt building, context management
- **Component Coordination**: AI ↔ Tools ↔ Vector Store

### **AI Processing**
- **Claude Integration**: System prompts, tool definitions
- **Tool Execution**: Dynamic tool calling based on query type
- **Response Generation**: Context-aware answer synthesis

### **Search Architecture**
- **Tool Manager**: Plugin system for extensible tools
- **Search Tool**: Course/lesson filtering, result formatting
- **Vector Store**: Semantic search, course name resolution

### **Data Layer**
- **ChromaDB**: Persistent vector storage
- **Embeddings**: Sentence transformers for similarity
- **Collections**: Separate course catalog + content storage

## Key Design Patterns

1. **Tool-Based Architecture**: Claude decides when to search vs use knowledge
2. **Semantic Resolution**: Fuzzy course name matching via embeddings  
3. **Parallel Source Tracking**: Search results + UI citations handled separately
4. **Session Continuity**: Conversation context maintained across queries
5. **Error Resilience**: Graceful fallbacks at each layer