# Background Task Processing - Chat Evaluation

## Chat-Evaluation Processing Workflow

### Complete Processing Sequence for Chat-Evaluation Task Type
```mermaid
sequenceDiagram
    participant BG as Background Processor
    participant DB as Database
    participant Glenn as Glenn Chat API
    participant LLM as LLM Similarity Service
    
    Note over BG: Background Processing Loop
    
    BG->>DB: SELECT * FROM tasks<br/>WHERE status = 'pending' AND task_type = 'chat-evaluation'<br/>ORDER BY created_at LIMIT 1
    DB-->>BG: Return oldest chat-evaluation task<br/>{task_id, task_file_blob, user_id}
    
    BG->>DB: UPDATE tasks SET status = 'processing', started_at = NOW()<br/>WHERE task_id = 'task_123'
    
    Note over BG: Step 1: Parse Excel and Extract Data
    
    BG->>BG: Parse task_file_blob (Excel) <br/>Prepare batch data (e.g., 100 rows)
    
    BG->>DB: INSERT INTO chat_evaluation_input<br/>(task_id, row_number, question, golden_answer, golden_citations)<br/>VALUES (batch of all 100 rows)
    
    Note over BG: Step 2: Process Each Row
    
    BG->>DB: SELECT * FROM chat_evaluation_input<br/>WHERE task_id = 'task_123'<br/>ORDER BY row_number
    DB-->>BG: Return all content rows (100 rows)
    
    loop For each content row (1 to 100)
        Note over BG: Process Row N
        
        BG->>Glenn: POST /chat<br/>{"question": "What is AI?"}
        Glenn-->>BG: {"answer": "AI is...", "citations": ["url1", "url2"]}
        
        BG->>LLM: POST /similarity<br/>{"golden_answer": "...", "api_answer": "..."}
        LLM-->>BG: {"answer_similarity": 0.85}

        BG->>BG: calculate citation url similarity
        
        BG->>DB: INSERT INTO chat_evaluation_output<br/>(task_id, row_number, api_answer, api_citations, answer_similarity, citation_similarity)
    end
    
    Note over BG: Step 3: Generate Final Excel Report
    
    BG->>DB: SELECT ceo.*, cei.question, cei.golden_answer, cei.golden_citations<br/>FROM chat_evaluation_output ceo<br/>JOIN chat_evaluation_input cei ON ceo.task_id = cei.task_id AND ceo.row_number = cei.row_number<br/>WHERE ceo.task_id = 'task_123'<br/>ORDER BY row_number
    DB-->>BG: Return complete results with context<br/>(100 rows with all data)
    
    BG->>BG: Generate Excel file from results<br/> and Convert Excel to blob data
    
    BG->>DB: UPDATE tasks<br/>SET status = 'completed', completed_at = NOW(), results_file_blob = [excel_blob], results_file_size = [size]<br/>WHERE task_id = 'task_123'
    
    Note over BG: Processing Complete ✅
```

## Error Handling for Chat-Evaluation Processing

### API Failure During Row Processing
```mermaid
sequenceDiagram
    participant BG as Background Processor
    participant DB as Database
    participant Glenn as Glenn Chat API
    participant LLM as LLM Similarity Service
    
    Note over BG: Processing Row 45 of 100
    
    BG->>Glenn: POST /chat<br/>{"question": "Complex question..."}
    Glenn--xBG: 500 Internal Server Error<br/>API temporarily unavailable
    
    BG->>BG: Retry with exponential backoff<br/>Attempt 1, 2, 3 (max 3 retries)
    
    alt Max retries exceeded
        BG->>DB: UPDATE tasks<br/>SET status = 'failed', error_message = 'Glenn Chat API unavailable after row 45'<br/>WHERE task_id = 'task_123'
        Note over BG: Mark task as failed
    else Retry successful
        Glenn-->>BG: {"answer": "...", "citations": [...]}
        BG->>LLM: Continue with similarity calculation
        Note over BG: Continue processing remaining rows
    end
```

### Excel Parsing Failure
```mermaid
sequenceDiagram
    participant BG as Background Processor
    participant DB as Database
    
    BG->>DB: SELECT task_file_blob FROM tasks WHERE task_id = 'task_123'
    DB-->>BG: Return Excel blob data
    
    BG->>BG: Parse Excel file - FAILED<br/>Corrupted file or invalid format
    
    BG->>DB: UPDATE tasks<br/>SET status = 'failed', error_message = 'Unable to parse Excel file: invalid format or corrupted data'<br/>WHERE task_id = 'task_123'
    
    Note over BG: Task marked as failed, no further processing
```

## Performance Characteristics

### Processing Metrics
- **Row Processing Rate**: ~5-10 rows per minute (depending on API response times)
- **API Call Pattern**: 2 external API calls per row (Glenn Chat + LLM Similarity)
- **Progress Updates**: Every 10 rows processed (to avoid excessive database updates)
- **Memory Usage**: Processes rows sequentially to avoid loading entire dataset

### Database Operations
- **Bulk Insert**: All content rows inserted in single transaction
- **Sequential Processing**: Results inserted one row at a time
- **Final Aggregation**: Single query to join content + results for Excel generation
