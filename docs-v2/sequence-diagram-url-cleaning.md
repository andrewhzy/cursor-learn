# URL Cleaning Background Processing Sequence Diagram - Version 2

## URL Cleaning Processing Flow (In-Memory)

### Success Flow - Complete URL Processing (In-Memory)
```mermaid
sequenceDiagram
    participant BG as Background Task Processor
    participant DB as Database (tasks table)
    participant MEM as In-Memory Processor
    participant Glean as Glean Platform Services
    
    BG->>DB: SELECT task FROM tasks<br/>WHERE task_type = 'url-cleaning' AND status = 'queueing'<br/>ORDER BY created_at ASC LIMIT 1
    DB-->>BG: Return task_id: "task_456", Excel file blob
    
    BG->>DB: UPDATE tasks SET status = 'processing', started_at = NOW()<br/>WHERE task_id = 'task_456'
    DB-->>BG: Task status updated
    
    BG->>MEM: Load Excel blob and parse in-memory<br/>Extract 100 URLs from first column
    MEM-->>BG: URL list ready in memory
    
    loop For each URL (100 iterations)
        BG->>MEM: Get next URL data (row X)
        MEM-->>BG: Return {original_url, parsed_title}
        
        Note over BG: Step 1: Direct URL Search
        BG->>Glean: POST /search<br/>{"query": "https://example.com/article-title"}
        
        alt Glean Platform Services returns results
            Glean-->>BG: {"results": [{"url": "https://cleaned-example.com/article"}]}
            BG->>MEM: Store result in memory<br/>{original_url, cleaned_url, status: 'success', method: 'direct_search'}
        else No results from direct search
            Glean-->>BG: {"results": []}
            Note over BG: Step 2: Title-Based Search Fallback
            BG->>Glean: POST /search<br/>{"query": "article-title"}
            
            alt Glean Platform Services returns results with title
                Glean-->>BG: {"results": [{"url": "https://cleaned-example.com/article"}]}
                BG->>MEM: Store result in memory<br/>{original_url, cleaned_url, status: 'success', method: 'title_search'}
            else No results from title search
                Glean-->>BG: {"results": []}
                BG->>MEM: Store result in memory<br/>{original_url, cleaned_url: null, status: 'failed', method: 'both_failed'}
            end
        end
    end
    
    BG->>MEM: Generate final results JSON<br/>Combine all URL processing results with summary statistics
    MEM-->>BG: Complete results structure
    
    BG->>BG: Generate Excel results file from in-memory data<br/>Create results_file_blob
    
    BG->>DB: UPDATE tasks<br/>SET status = 'completed', completed_at = NOW(),<br/>final_results = {complete_json}, results_file_blob = {excel_data}<br/>WHERE task_id = 'task_456'
    DB-->>BG: Task completed successfully
```

### Glean Platform Services API Failure (In-Memory)
```mermaid
sequenceDiagram
    participant BG as Background Task Processor
    participant DB as Database (tasks table)
    participant MEM as In-Memory Processor
    participant Glean as Glean Platform Services
    
    BG->>DB: SELECT task FROM tasks WHERE task_type = 'url-cleaning'
    DB-->>BG: Return task_id: "task_456", Excel blob
    
    BG->>MEM: Parse Excel data in-memory
    MEM-->>BG: 100 URLs loaded in memory
    
    Note over BG: Processing URL 50 of 100
    
    BG->>Glean: POST /search<br/>{"query": "https://complex-url.com/path"}
    Glean--xBG: 500 Internal Server Error<br/>API temporarily unavailable
    
    BG->>BG: Retry after 30 seconds (attempt 2/3)
    BG->>BG: Retry after 60 seconds (attempt 3/3) - FAILED
    
    BG->>MEM: Store error result in memory<br/>{original_url, status: 'api_error', method: 'api_unavailable'}
    MEM-->>BG: Error result stored
    
    alt API becomes available
        Glean-->>BG: {"results": [...]}
        Note over BG: Continue processing remaining URLs
    end
```

### URL Parsing Error (In-Memory)
```mermaid
sequenceDiagram
    participant BG as Background Task Processor
    participant DB as Database (tasks table)
    participant MEM as In-Memory Processor
    participant Glean as Glean Platform Services
    
    BG->>MEM: Get next URL from memory
    MEM-->>BG: Return malformed URL: "not-a-valid-url-format"
    
    BG->>BG: Validate URL format - FAILED<br/>Cannot parse URL structure
    
    BG->>Glean: POST /search<br/>{"query": "invalid-url-format"}
    Glean-->>BG: {"results": []}
    
    BG->>MEM: Store error result in memory<br/>{original_url, status: 'url_parse_error', method: 'url_parse_error', notes: 'Invalid URL format'}
    MEM-->>BG: Error recorded, continue with next URL
```

## Processing Characteristics

### Performance Metrics
- **Processing Rate**: ~25-35 URLs per minute (depending on Glean Platform Services response times)
- **Memory Usage**: Entire URL dataset loaded in memory for processing
- **Error Handling**: 3 retry attempts with exponential backoff for API failures
- **Success Rate**: ~70-80% URLs successfully cleaned (depends on data quality)
- **API Call Pattern**: 1-2 Glean Platform Services calls per URL (direct search + optional title search)

### Data Flow Summary
1. **Task Selection**: FIFO queue processing of url-cleaning tasks
2. **In-Memory Loading**: Load entire Excel file into memory structures
3. **Sequential Processing**: Process each URL with search API calls
4. **Two-Step Search**: Direct URL search followed by title-based search fallback
5. **Memory Accumulation**: Store all results in memory during processing
6. **Final Assembly**: Generate complete results JSON and Excel file
7. **Atomic Completion**: Single database update with all final data

### Memory Management
- **Structured Data**: Organized in-memory data structures for URL processing
- **Result Accumulation**: Build complete results during processing
- **Memory Cleanup**: Automatic cleanup after task completion
- **Large File Handling**: Chunked processing for very large URL lists

### Error Recovery
- **API Failures**: Retry with exponential backoff, record as 'api_error' if all retries fail
- **URL Parse Errors**: Record as 'url_parse_error' and continue processing
- **Memory Errors**: Graceful handling of memory limitations
- **Partial Processing**: Save partial results for manual retry/resume
- **Data Integrity**: Complete results or failure, no partial database states
- **Fallback Strategy**: Two-step search process (direct URL → title extraction)

### Results Storage Format
```json
{
  "total_urls": 100,
  "processed_urls": 100,
  "processing_time_ms": 180000,
  "success_count": 80,
  "failed_count": 20,
  "results": [
    {
      "row_number": 1,
      "original_url": "https://example.com/old-article",
      "cleaned_url": "https://example.com/article",
      "status": "success",
      "method": "direct_search",
      "processing_time_ms": 500
    },
    {
      "row_number": 2,
      "original_url": "https://broken-link.com/page",
      "cleaned_url": null,
      "status": "failed",
      "method": "both_failed",
      "processing_time_ms": 2000
    }
  ],
  "summary": {
    "direct_search_success": 60,
    "title_search_success": 20,
    "api_failures": 15,
    "parse_errors": 5
  }
}
``` 