# Chat Evaluation Background Processing Sequence Diagram

## Chat Evaluation Processing Flow

### Success Flow - Complete Excel Processing
```mermaid
sequenceDiagram
    participant BG as Background Task Processor
    participant DB as Database
    participant Glean as Glean Platform Services
    participant LLM as LLM Similarity Service
    
    BG->>DB: SELECT task FROM tasks<br/>WHERE task_type = 'chat-evaluation' AND status = 'queueing'<br/>ORDER BY created_at ASC LIMIT 1
    DB-->>BG: Return task_id: "task_123", Excel file blob
    
    BG->>BG: Parse Excel file<br/>Extract 100 rows: questions, golden_answers, golden_citations
    
    BG->>DB: BEGIN TRANSACTION
    
    loop For each row (100 rows)
        BG->>DB: INSERT INTO chat_evaluation_input<br/>(task_id, row_number, question, golden_answer, golden_citations)
        DB-->>BG: Row inserted successfully
    end
    
    BG->>DB: UPDATE tasks SET status = 'processing', started_at = NOW()<br/>WHERE task_id = 'task_123'
    BG->>DB: COMMIT TRANSACTION
    
    loop For each row (100 iterations)
        BG->>DB: SELECT question, golden_answer, golden_citations<br/>FROM chat_evaluation_input WHERE task_id = 'task_123' AND row_number = X
        DB-->>BG: Return row data
        
        BG->>Glean: POST /chat<br/>{"question": "What is AI?"}
        Glean-->>BG: {"answer": "AI is...", "citations": ["url1", "url2"]}
        
        BG->>LLM: POST /similarity<br/>{"text1": "golden_answer", "text2": "api_answer"}
        LLM-->>BG: {"similarity": 0.85}
        
        BG->>LLM: POST /similarity<br/>{"citations1": ["url1"], "citations2": ["url2"]}
        LLM-->>BG: {"similarity": 0.92}
        
        BG->>DB: INSERT INTO chat_evaluation_output<br/>(task_id, row_number, api_answer, api_citations, answer_similarity, citation_similarity)
        DB-->>BG: Results stored
        
        BG->>DB: UPDATE tasks SET progress_percentage = (row_number / 100 * 100)<br/>WHERE task_id = 'task_123'
        DB-->>BG: Progress updated
    end
    
    BG->>BG: Generate Excel results file<br/>Combine input + output data
    BG->>DB: UPDATE tasks<br/>SET status = 'completed', completed_at = NOW(), results_file_blob = excel_data<br/>WHERE task_id = 'task_123'
    DB-->>BG: Task completed successfully
```

### Error Flow - API Failure During Processing
```mermaid
sequenceDiagram
    participant BG as Background Task Processor
    participant DB as Database
    participant Glean as Glean Platform Services
    participant LLM as LLM Similarity Service
    
    BG->>DB: SELECT task FROM tasks WHERE task_type = 'chat-evaluation'
    DB-->>BG: Return task_id: "task_123"
    
    BG->>Glean: POST /chat<br/>{"question": "Complex question..."}
    Glean--xBG: 500 Internal Server Error<br/>API temporarily unavailable
    
    BG->>BG: Retry after 30 seconds (attempt 2/3)
    BG->>BG: Retry after 60 seconds (attempt 3/3) - FAILED
    
    BG->>DB: UPDATE tasks<br/>SET status = 'failed', error_message = 'Glean Platform Services API unavailable after row 45'<br/>WHERE task_id = 'task_123'
    DB-->>BG: Task marked as failed
    
    alt API becomes available later
        Glean-->>BG: {"answer": "...", "citations": [...]}
        Note over BG: Manual retry would be needed to resume from row 45
    end
```

## Processing Characteristics

### Performance Metrics
- **Processing Rate**: ~10-15 rows per minute (depending on API response times)
- **Memory Usage**: Processes one row at a time to minimize memory footprint
- **Error Handling**: 3 retry attempts with exponential backoff
- **Progress Tracking**: Real-time progress updates per row
- **Transaction Safety**: Input data inserted in single transaction, results processed individually

### Data Flow Summary
1. **Task Selection**: FIFO queue processing of chat-evaluation tasks
2. **Excel Parsing**: Extract questions, golden answers, and citations  
3. **Bulk Insert**: Store all input data in chat_evaluation_input table
4. **Row Processing**: Sequential processing with API calls per row
5. **Results Storage**: Store API responses and similarity scores
6. **Progress Updates**: Update task progress percentage per row
7. **Final Assembly**: Generate Excel file with combined input/output data
8. **Completion**: Update task status and store results blob

### Error Recovery
- **API Failures**: Retry with exponential backoff (30s, 60s, 120s)
- **Partial Processing**: Resume from last successful row
- **Data Integrity**: Input data preserved even if processing fails  
- **Manual Recovery**: Failed tasks can be manually requeued
- **API Call Pattern**: 2 external API calls per row (Glean Platform Services + LLM Similarity)
