# URL Cleaning Background Processing Sequence Diagram

## URL Cleaning Processing Flow

### Success Flow - Complete URL Processing (100 URLs)
```mermaid
sequenceDiagram
    participant BG as Background Task Processor
    participant DB as Database
    participant Glean as Glean Platform Services
    
    BG->>DB: SELECT task FROM tasks<br/>WHERE task_type = 'url-cleaning' AND status = 'queueing'<br/>ORDER BY created_at ASC LIMIT 1
    DB-->>BG: Return task_id: "task_456", Excel file blob
    
    BG->>BG: Parse Excel file<br/>Extract 100 URLs from first column
    
    BG->>DB: BEGIN TRANSACTION
    
    loop For each URL (100 URLs)
        BG->>DB: INSERT INTO url_cleaning_input<br/>(task_id, row_number, url, parsed_title)
        DB-->>BG: URL inserted successfully
    end
    
    BG->>DB: UPDATE tasks SET status = 'processing', started_at = NOW()<br/>WHERE task_id = 'task_456'
    BG->>DB: COMMIT TRANSACTION
    
    loop For each URL (100 iterations)
        BG->>DB: SELECT url FROM url_cleaning_input<br/>WHERE task_id = 'task_456' AND row_number = X
        DB-->>BG: Return URL: "https://example.com/article-title"
        
        Note over BG: Step 1: Direct URL Search
        BG->>Glean: POST /search<br/>{"query": "https://example.com/article-title"}
        
        alt Glean Platform Services returns results
            Glean-->>BG: {"results": [{"url": "https://cleaned-example.com/article"}]}
            BG->>DB: INSERT INTO url_cleaning_output<br/>(task_id, row_number, original_url, cleaned_url, status: 'success', method: 'direct_search')
        else No results from direct search
            Glean-->>BG: {"results": []}
            Note over BG: Step 2: Title-Based Search Fallback
            BG->>Glean: POST /search<br/>{"query": "article-title"}
            
            alt Glean Platform Services returns results with title
                Glean-->>BG: {"results": [{"url": "https://cleaned-example.com/article"}]}
                BG->>DB: INSERT INTO url_cleaning_output<br/>(task_id, row_number, original_url, cleaned_url, status: 'success', method: 'title_search')
            else No results from title search
                Glean-->>BG: {"results": []}
                BG->>DB: INSERT INTO url_cleaning_output<br/>(task_id, row_number, original_url, cleaned_url: null, status: 'failed', method: 'both_failed')
            end
        end
        
        BG->>DB: UPDATE tasks SET progress_percentage = (row_number / 100 * 100)<br/>WHERE task_id = 'task_456'
        DB-->>BG: Progress updated
    end
    
    BG->>BG: Generate Excel results file<br/>Combine original + cleaned URLs
    BG->>DB: UPDATE tasks<br/>SET status = 'completed', completed_at = NOW(), results_file_blob = excel_data<br/>WHERE task_id = 'task_456'
    DB-->>BG: Task completed successfully
```

### Glean Platform Services API Failure
```mermaid
sequenceDiagram
    participant BG as Background Task Processor
    participant DB as Database
    participant Glean as Glean Platform Services
    
    BG->>DB: SELECT task FROM tasks WHERE task_type = 'url-cleaning'
    DB-->>BG: Return task_id: "task_456"
    
    BG->>Glean: POST /search<br/>{"query": "https://complex-url.com/path"}
    Glean--xBG: 500 Internal Server Error<br/>API temporarily unavailable
    
    BG->>BG: Retry after 30 seconds (attempt 2/3)
    BG->>BG: Retry after 60 seconds (attempt 3/3) - FAILED
    
    BG->>DB: INSERT INTO url_cleaning_output<br/>(task_id, row_number, original_url, status: 'api_error', method: 'api_unavailable')
    DB-->>BG: Error recorded
    
    alt API becomes available
        Glean-->>BG: {"results": [...]}
        Note over BG: Continue processing remaining URLs
    end
```

### URL Parsing Error
```mermaid
sequenceDiagram
    participant BG as Background Task Processor
    participant DB as Database
    participant Glean as Glean Platform Services
    
    BG->>DB: SELECT url FROM url_cleaning_input WHERE task_id = 'task_456'
    DB-->>BG: Return malformed URL: "not-a-valid-url-format"
    
    BG->>BG: Validate URL format - FAILED<br/>Cannot parse URL structure
    
    BG->>Glean: POST /search<br/>{"query": "invalid-url-format"}
    Glean-->>BG: {"results": []}
    
    BG->>DB: INSERT INTO url_cleaning_output<br/>(task_id, row_number, original_url, status: 'url_parse_error', method: 'url_parse_error', processing_notes: 'Invalid URL format')
    DB-->>BG: Error recorded, continue with next URL
```

## Processing Characteristics

### Performance Metrics
- **Processing Rate**: ~20-30 URLs per minute (depending on Glean Platform Services response times)
- **Memory Usage**: Processes one URL at a time to minimize memory footprint
- **Error Handling**: 3 retry attempts with exponential backoff for API failures
- **Success Rate**: ~70-80% URLs successfully cleaned (depends on data quality)
- **URL Processing Rate**: ~10-20 URLs per minute (depending on Glean Platform Services response times)
- **API Call Pattern**: 1-2 Glean Platform Services calls per URL (direct search + optional title search)

### Data Flow Summary
1. **Task Selection**: FIFO queue processing of url-cleaning tasks
2. **Excel Parsing**: Extract URLs from first column of Excel file
3. **Bulk Insert**: Store all URLs in url_cleaning_input table
4. **URL Processing**: Sequential processing with search API calls per URL
5. **Two-Step Search**: Direct URL search followed by title-based search fallback
6. **Results Storage**: Store cleaned URLs, status, and processing method
7. **Progress Updates**: Update task progress percentage per URL
8. **Final Assembly**: Generate Excel file with original and cleaned URLs
9. **Completion**: Update task status and store results blob

### Error Recovery
- **API Failures**: Retry with exponential backoff, record as 'api_error' if all retries fail
- **URL Parse Errors**: Record as 'url_parse_error' and continue processing
- **Partial Processing**: Resume from last processed URL
- **Data Integrity**: Input URLs preserved even if cleaning fails
- **Manual Recovery**: Failed tasks can be manually requeued
- **Fallback Strategy**: Two-step search process (direct URL → title extraction)