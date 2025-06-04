# Chat Evaluation Background Processing Sequence Diagram - Version 2

## Chat Evaluation Processing Flow (In-Memory)

### Success Flow - Complete Excel Processing (In-Memory)
```mermaid
sequenceDiagram
    participant BG as Background Task Processor
    participant DB as Database (tasks table)
    participant MEM as In-Memory Processor
    participant Glean as Glean Platform Services
    participant LLM as LLM Similarity Service
    
    BG->>DB: SELECT task FROM tasks<br/>WHERE task_type = 'chat-evaluation' AND status = 'queueing'<br/>ORDER BY created_at ASC LIMIT 1
    DB-->>BG: Return task_id: "task_123", Excel file blob
    
    BG->>DB: UPDATE tasks SET status = 'processing', started_at = NOW()<br/>WHERE task_id = 'task_123'
    DB-->>BG: Task status updated
    
    BG->>MEM: Load Excel blob and parse in-memory<br/>Extract 100 rows: questions, golden_answers, golden_citations
    MEM-->>BG: Parsed data structure ready
    
    loop For each row (100 iterations)
        BG->>MEM: Get next question data (row X)
        MEM-->>BG: Return {question, golden_answer, golden_citations}
        
        BG->>Glean: POST /chat<br/>{"question": "What is AI?"}
        Glean-->>BG: {"answer": "AI is...", "citations": ["url1", "url2"]}
        
        BG->>LLM: POST /similarity<br/>{"text1": "golden_answer", "text2": "api_answer"}
        LLM-->>BG: {"similarity": 0.85}
        
        BG->>LLM: POST /similarity<br/>{"citations1": ["url1"], "citations2": ["url2"]}
        LLM-->>BG: {"similarity": 0.92}
        
        BG->>MEM: Store result in memory<br/>{row_number, api_answer, api_citations, similarities}
        MEM-->>BG: Result stored in memory
    end
    
    BG->>MEM: Generate final results JSON<br/>Combine all processing results with summary statistics
    MEM-->>BG: Complete results structure
    
    BG->>BG: Generate Excel results file from in-memory data<br/>Create results_file_blob
    
    BG->>DB: UPDATE tasks<br/>SET status = 'completed', completed_at = NOW(),<br/>final_results = {complete_json}, results_file_blob = {excel_data}<br/>WHERE task_id = 'task_123'
    DB-->>BG: Task completed successfully
```

### Error Flow - API Failure During Processing (In-Memory)
```mermaid
sequenceDiagram
    participant BG as Background Task Processor
    participant DB as Database (tasks table)
    participant MEM as In-Memory Processor
    participant Glean as Glean Platform Services
    participant LLM as LLM Similarity Service
    
    BG->>DB: SELECT task FROM tasks WHERE task_type = 'chat-evaluation'
    DB-->>BG: Return task_id: "task_123", Excel blob
    
    BG->>MEM: Parse Excel data in-memory
    MEM-->>BG: 100 rows loaded in memory
    
    Note over BG: Processing row 45 of 100
    
    BG->>Glean: POST /chat<br/>{"question": "Complex question..."}
    Glean--xBG: 500 Internal Server Error<br/>API temporarily unavailable
    
    BG->>BG: Retry after 30 seconds (attempt 2/3)
    BG->>BG: Retry after 60 seconds (attempt 3/3) - FAILED
    
    BG->>MEM: Save partial results (44 completed rows)
    MEM-->>BG: Partial results saved
    
    BG->>DB: UPDATE tasks<br/>SET status = 'failed', error_message = 'Glean Platform Services API unavailable after row 45'<br/>WHERE task_id = 'task_123'
    DB-->>BG: Task marked as failed
    
    alt API becomes available later
        Glean-->>BG: {"answer": "...", "citations": [...]}
        Note over BG: Manual retry would be needed to resume processing
    end
```

## Processing Characteristics

### Performance Metrics
- **Processing Rate**: ~15-20 rows per minute (depending on API response times)
- **Memory Usage**: Entire Excel dataset loaded in memory for processing
- **Error Handling**: 3 retry attempts with exponential backoff
- **Transaction Safety**: Single atomic update with complete results

### Data Flow Summary
1. **Task Selection**: FIFO queue processing of chat-evaluation tasks
2. **In-Memory Loading**: Load entire Excel file into memory structures
3. **Sequential Processing**: Process each row with API calls
4. **Memory Accumulation**: Store all results in memory during processing
5. **Final Assembly**: Generate complete results JSON and Excel file
6. **Atomic Completion**: Single database update with all final data

### Memory Management
- **Structured Data**: Organized in-memory data structures for processing
- **Result Accumulation**: Build complete results during processing
- **Memory Cleanup**: Automatic cleanup after task completion
- **Large File Handling**: Chunked processing for very large Excel files

### Error Recovery
- **API Failures**: Retry with exponential backoff, save partial results on failure
- **Memory Errors**: Graceful handling of memory limitations
- **Partial Processing**: Save partial results for manual retry/resume
- **Data Integrity**: Complete results or failure, no partial database states
- **API Call Pattern**: 2 external API calls per row (Glean Platform Services + LLM Similarity)

### Results Storage Format
```json
{
  "total_questions": 100,
  "processed_questions": 100,
  "processing_time_ms": 450000,
  "average_answer_similarity": 0.85,
  "average_citation_similarity": 0.92,
  "results": [
    {
      "row_number": 1,
      "question": "What is AI?",
      "golden_answer": "AI is artificial intelligence...",
      "api_answer": "AI refers to artificial intelligence...",
      "answer_similarity": 0.85,
      "citation_similarity": 0.92,
      "processing_time_ms": 1250
    }
  ],
  "summary": {
    "high_similarity_count": 75,
    "medium_similarity_count": 20,
    "low_similarity_count": 5
  }
}
``` 