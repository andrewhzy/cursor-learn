# Database Schema Design - Version 2 (Simplified)

## Overview

This document defines the simplified database schema for the Internal Task Management API system (Version 2). This version uses a single-table design where all processing is done in-memory with final results stored directly in the main tasks table.

**Database Technology:** MariaDB (MySQL-compatible)
**Schema Version:** 2.0
**Character Set:** utf8mb4 (full Unicode support)
**Collation:** utf8mb4_unicode_ci
**Architecture**: Single table design with in-memory processing

## Single Table Design

### 1. tasks - Complete Task Management Table

The only table needed for this simplified architecture, storing task metadata, file blobs, processing status, and final results.

```sql
CREATE TABLE tasks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id VARCHAR(255) NOT NULL,
    original_filename VARCHAR(500) NOT NULL,
    sheet_name VARCHAR(255) NOT NULL,
    task_file_blob BYTEA NOT NULL,
    task_file_size BIGINT NOT NULL,
    results_file_blob BYTEA,
    results_file_size BIGINT,
    file_mime_type VARCHAR(100) DEFAULT 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
    task_status VARCHAR(50) NOT NULL DEFAULT 'queueing',
    task_type VARCHAR(50) NOT NULL DEFAULT 'chat-evaluation',
    upload_batch_id UUID NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    started_at TIMESTAMP WITH TIME ZONE,
    completed_at TIMESTAMP WITH TIME ZONE,
    cancelled_at TIMESTAMP WITH TIME ZONE,
    error_message TEXT,
    processing_metadata JSONB,
    final_results JSONB,
    created_by VARCHAR(255) NOT NULL,
    
    CONSTRAINT valid_status CHECK (task_status IN ('queueing', 'processing', 'completed', 'cancelled', 'failed')),
    CONSTRAINT valid_task_type CHECK (task_type IN ('chat-evaluation', 'url-cleaning'))
);

-- Indexes for performance optimization  
CREATE INDEX idx_tasks_user_id ON tasks(user_id);
CREATE INDEX idx_tasks_status ON tasks(task_status);
CREATE INDEX idx_tasks_type_status ON tasks(task_type, task_status);
CREATE INDEX idx_tasks_upload_batch ON tasks(upload_batch_id);
CREATE INDEX idx_tasks_created_at ON tasks(created_at DESC);
CREATE INDEX idx_tasks_user_status ON tasks(user_id, task_status);
CREATE INDEX idx_tasks_background_processing ON tasks(task_type, task_status, created_at) WHERE task_status = 'queueing';

-- Additional indexes for results queries
CREATE INDEX idx_tasks_completed_results ON tasks(user_id, task_status) WHERE task_status = 'completed' AND final_results IS NOT NULL;
CREATE INDEX idx_tasks_processing_metadata ON tasks USING GIN(processing_metadata) WHERE processing_metadata IS NOT NULL;
CREATE INDEX idx_tasks_final_results ON tasks USING GIN(final_results) WHERE final_results IS NOT NULL;
```

#### Task Status Flow
- **queueing**: Task created, waiting for processing
- **processing**: Task is being executed in-memory
- **completed**: Task finished successfully with results stored in `final_results`
- **cancelled**: Task cancelled by user
- **failed**: Task failed with error stored in `error_message`

#### Enhanced Fields for Version 2

**New/Enhanced Columns:**
- **`processing_metadata`** (JSONB): Stores processing configuration and temporary metadata
- **`final_results`** (JSONB): Stores the complete processing results directly in the table

**Chat Evaluation Results Structure:**
```json
{
  "total_questions": 100,
  "processed_questions": 100,
  "average_answer_similarity": 0.85,
  "average_citation_similarity": 0.92,
  "results": [
    {
      "row_number": 1,
      "question": "What is AI?",
      "golden_answer": "AI is...",
      "api_answer": "AI is artificial intelligence...",
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

**URL Cleaning Results Structure:**
```json
{
  "total_urls": 100,
  "processed_urls": 100,
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

## Architecture Benefits

### Simplified Design Advantages

**Performance Benefits:**
- **Reduced Database Load**: No intermediate table operations
- **Atomic Operations**: Single table updates for complete transactions
- **Faster Queries**: No complex JOINs required
- **Better Caching**: Single table caching strategy

**Operational Benefits:**
- **Simplified Backup/Restore**: Only one table to manage
- **Easier Monitoring**: Single table metrics
- **Reduced Complexity**: No relationship management
- **Faster Development**: Simpler data model

**Scalability Benefits:**
- **Horizontal Scaling**: Easier table partitioning
- **Replication**: Simplified master-slave setup
- **Sharding**: Single table sharding strategy
- **Memory Efficiency**: In-memory processing reduces storage needs

### Data Management Strategy

#### Processing Workflow
1. **Task Creation**: Basic task record created with `task_status = 'queueing'`
2. **In-Memory Processing**: Background processor loads task and processes entirely in-memory
3. **Final Storage**: Complete results stored in `final_results` JSONB field
4. **Completion**: Task status updated to `completed` with timestamp

#### Memory Management
- **Chunk Processing**: Large Excel files processed in memory chunks
- **Streaming Results**: Results streamed directly to `final_results` field
- **Error Handling**: Processing errors stored in `error_message` field
- **Resource Cleanup**: Automatic memory cleanup after processing

## Performance Considerations

### Indexing Strategy
- **Primary Access Patterns**: User queries, status filtering, type-based processing
- **JSONB Indexes**: GIN indexes on result fields for complex queries
- **Composite Indexes**: Multi-column indexes for common query patterns
- **Partial Indexes**: Status-specific indexes for background processing

### Storage Optimization
```sql
-- Optimize for large JSONB fields
ALTER TABLE tasks SET (toast_tuple_target = 8160);

-- Compress old completed tasks
CREATE OR REPLACE FUNCTION compress_old_tasks() RETURNS void AS $$
BEGIN
    UPDATE tasks 
    SET processing_metadata = NULL 
    WHERE task_status = 'completed' 
    AND completed_at < NOW() - INTERVAL '7 days'
    AND processing_metadata IS NOT NULL;
END;
$$ LANGUAGE plpgsql;

-- Schedule regular compression
SELECT cron.schedule('compress-old-tasks', '0 2 * * *', 'SELECT compress_old_tasks();');
```

### Cleanup Strategy
```sql
-- Auto-cleanup old completed tasks (run monthly)
DELETE FROM tasks 
WHERE task_status IN ('completed', 'cancelled', 'failed')
AND (completed_at < NOW() - INTERVAL '90 days' 
     OR cancelled_at < NOW() - INTERVAL '90 days'
     OR (task_status = 'failed' AND updated_at < NOW() - INTERVAL '30 days'));

-- Archive important results before cleanup
CREATE TABLE tasks_archive AS 
SELECT id, user_id, original_filename, task_type, final_results, completed_at
FROM tasks 
WHERE task_status = 'completed' 
AND completed_at BETWEEN NOW() - INTERVAL '90 days' AND NOW() - INTERVAL '7 days';
```

## Migration from Version 1

### Migration Strategy
If migrating from the multi-table Version 1 design:

```sql
-- Migrate chat evaluation data
UPDATE tasks 
SET final_results = (
    SELECT jsonb_build_object(
        'total_questions', COUNT(*),
        'results', jsonb_agg(
            jsonb_build_object(
                'row_number', cei.row_number,
                'question', cei.question,
                'golden_answer', cei.golden_answer,
                'api_answer', ceo.api_answer,
                'answer_similarity', ceo.answer_similarity,
                'citation_similarity', ceo.citation_similarity
            ) ORDER BY cei.row_number
        )
    )
    FROM chat_evaluation_input cei
    JOIN chat_evaluation_output ceo ON cei.task_id = ceo.task_id AND cei.row_number = ceo.row_number
    WHERE cei.task_id = tasks.id
)
WHERE task_type = 'chat-evaluation' AND task_status = 'completed';

-- Migrate URL cleaning data
UPDATE tasks 
SET final_results = (
    SELECT jsonb_build_object(
        'total_urls', COUNT(*),
        'results', jsonb_agg(
            jsonb_build_object(
                'row_number', uci.row_number,
                'original_url', uci.url,
                'cleaned_url', uco.cleaned_url,
                'status', uco.status,
                'method', uco.method
            ) ORDER BY uci.row_number
        )
    )
    FROM url_cleaning_input uci
    JOIN url_cleaning_output uco ON uci.task_id = uco.task_id AND uci.row_number = uco.row_number
    WHERE uci.task_id = tasks.id
)
WHERE task_type = 'url-cleaning' AND task_status = 'completed';
```

## Monitoring and Observability

### Key Metrics
- **Task Processing Rate**: Tasks completed per hour
- **Memory Usage**: Peak memory usage during processing
- **Processing Time**: Average time per task type
- **Error Rate**: Failed tasks percentage
- **Storage Growth**: JSONB field size trends

### JSONB Query Examples
```sql
-- Find tasks with high similarity scores
SELECT id, user_id, final_results->'average_answer_similarity' as avg_similarity
FROM tasks 
WHERE task_type = 'chat-evaluation' 
AND task_status = 'completed'
AND (final_results->>'average_answer_similarity')::numeric > 0.9;

-- Count successful URL cleanings
SELECT user_id, COUNT(*) as successful_cleanings
FROM tasks
WHERE task_type = 'url-cleaning'
AND task_status = 'completed'
AND (final_results->>'success_count')::integer > 0
GROUP BY user_id;
```

This simplified Version 2 design provides all the functionality of the original design while significantly reducing complexity and improving performance through the single-table, in-memory processing approach. 