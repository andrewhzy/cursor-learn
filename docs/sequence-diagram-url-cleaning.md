# Background Task Processing - URL Cleaning

## URL-Cleaning Processing Workflow

### Complete Processing Sequence for URL-Cleaning Task Type
```mermaid
sequenceDiagram
    participant BG as Background Processor
    participant DB as Database
    participant Glean as Glean Search API
    
    Note over BG: Background Processing Loop
    
    BG->>DB: SELECT * FROM tasks<br/>WHERE status = 'pending' AND task_type = 'url-cleaning'<br/>ORDER BY created_at LIMIT 1
    DB-->>BG: Return oldest url-cleaning task<br/>{task_id, task_file_blob, user_id}
    
    BG->>DB: UPDATE tasks SET status = 'processing', started_at = NOW()<br/>WHERE task_id = 'task_456'
    
    Note over BG: Step 1: Parse Excel and Extract Data
    
    BG->>BG: Parse task_file_blob (Excel)<br/>Extract all rows: [url]<br/>Prepare batch data (e.g., 100 URLs)
    
    BG->>DB: INSERT INTO url_cleaning_input<br/>(task_id, row_number, url)<br/>VALUES (batch of all 100 rows)
    
    Note over BG: Step 2: Process Each URL Row
    
    BG->>DB: SELECT * FROM url_cleaning_input<br/>WHERE task_id = 'task_456'<br/>ORDER BY row_number
    DB-->>BG: Return all URL rows (100 rows)
    
    loop For each URL row (1 to 100)
        Note over BG: Process URL Row N
        
        BG->>Glean: POST /search<br/>{"query": "https://example.com/article-title"}
        
        alt Glean returns results
            Glean-->>BG: {"results": [{"url": "https://cleaned-example.com/article"}]}
            BG->>BG: Extract cleanedUrl from first result
            BG->>DB: INSERT INTO url_cleaning_output<br/>(task_id, row_number, original_url, cleaned_url, status: 'success', method: 'direct_search')
        else No results from direct search
            Glean-->>BG: {"results": []}
            BG->>BG: Parse URL and extract title<br/>e.g., "article-title" from URL path
            
            BG->>Glean: POST /search<br/>{"query": "article-title"}
            
            alt Glean returns results with title
                Glean-->>BG: {"results": [{"url": "https://cleaned-example.com/article"}]}
                BG->>BG: Extract cleanedUrl from first result
                BG->>DB: INSERT INTO url_cleaning_output<br/>(task_id, row_number, original_url, cleaned_url, status: 'success', method: 'title_search')
            else Still no results
                Glean-->>BG: {"results": []}
                BG->>DB: INSERT INTO url_cleaning_output<br/>(task_id, row_number, original_url, cleaned_url: null, status: 'failed', method: 'both_failed')
            end
        end
    end
    
    Note over BG: Step 3: Generate Final Excel Report
    
    BG->>DB: SELECT uco.*, uci.url as original_url<br/>FROM url_cleaning_output uco<br/>JOIN url_cleaning_input uci ON uco.task_id = uci.task_id AND uco.row_number = uci.row_number<br/>WHERE uco.task_id = 'task_456'<br/>ORDER BY row_number
    DB-->>BG: Return complete results with context<br/>(100 rows with all data)
    
    BG->>BG: Generate Excel file from results<br/>Columns: [row_number, original_url, cleaned_url, status, method, processing_notes]
    BG->>BG: Convert Excel to blob data
    
    BG->>DB: UPDATE tasks<br/>SET status = 'completed', completed_at = NOW(), results_file_blob = [excel_blob], results_file_size = [size]<br/>WHERE task_id = 'task_456'
    
    Note over BG: Processing Complete ✅
```

## Error Handling for URL-Cleaning Processing

### Glean Search API Failure
```mermaid
sequenceDiagram
    participant BG as Background Processor
    participant DB as Database
    participant Glean as Glean Search API
    
    Note over BG: Processing URL 50 of 100
    
    BG->>Glean: POST /search<br/>{"query": "https://complex-url.com/path"}
    Glean--xBG: 500 Internal Server Error<br/>API temporarily unavailable
    
    BG->>BG: Retry with exponential backoff<br/>Attempt 1, 2, 3 (max 3 retries)
    
    alt Max retries exceeded
        BG->>DB: INSERT INTO url_cleaning_output<br/>(task_id, row_number, original_url, cleaned_url: null, status: 'failed', message: 'failed to search')
        Note over BG: Mark this URL as API error, continue with next URL
    else Retry successful
        Glean-->>BG: {"results": [...]}
        Note over BG: Continue normal processing flow
    end
```

### Excel Parsing Failure
```mermaid
sequenceDiagram
    participant BG as Background Processor
    participant DB as Database
    
    BG->>DB: SELECT task_file_blob FROM tasks WHERE task_id = 'task_456'
    DB-->>BG: Return Excel blob data
    
    BG->>BG: Parse Excel file - FAILED<br/>Corrupted file or invalid format
    
    BG->>DB: UPDATE tasks<br/>SET status = 'failed', error_message = 'Unable to parse Excel file: invalid format or corrupted data'<br/>WHERE task_id = 'task_456'
    
    Note over BG: Task marked as failed, no further processing
```

### URL Parsing Error
```mermaid
sequenceDiagram
    participant BG as Background Processor
    participant DB as Database
    participant Glean as Glean Search API
    
    Note over BG: Processing malformed URL
    
    BG->>Glean: POST /search<br/>{"query": "invalid-url-format"}
    Glean-->>BG: {"results": []}
    
    BG->>BG: Parse URL to extract title - FAILED<br/>Malformed URL structure
    
    BG->>DB: INSERT INTO url_cleaning_output<br/>(task_id, row_number, original_url, cleaned_url: null, status: 'failed', method: 'url_parse_error')
    
    Note over BG: Continue with next URL
```

## Performance Characteristics

### Processing Metrics
- **URL Processing Rate**: ~10-20 URLs per minute (depending on Glean API response times)
- **API Call Pattern**: 1-2 Glean Search API calls per URL (direct search + optional title search)
- **Fallback Strategy**: Two-step approach (direct URL search → title-based search)
- **Memory Usage**: Processes URLs sequentially to avoid loading entire dataset

### Database Operations
- **Bulk Insert**: All URL rows inserted in single batch operation
- **Sequential Processing**: Results inserted one URL at a time
- **Final Aggregation**: Single query to join URL data + results for Excel generation

### Success Rate Expectations
- **Direct Search Success**: ~60-70% of URLs cleaned on first attempt
- **Title Search Success**: ~20-25% additional URLs cleaned on second attempt  
- **Overall Success Rate**: ~80-90% of URLs successfully cleaned
- **Failure Cases**: Broken URLs, non-existent content, API limitations 