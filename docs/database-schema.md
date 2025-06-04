# Database Schema Design

## Overview

This document defines the database schema for the Internal Task Management API system. The schema supports task management with Excel file processing, background task execution, and user-scoped data access.

**Database Technology:** MariaDB (MySQL-compatible)
**Schema Version:** 1.0
**Character Set:** utf8mb4 (full Unicode support)
**Collation:** utf8mb4_unicode_ci

## Main Tables

### 1. tasks - Main Task Management Table

Primary table storing task metadata, file blobs, and processing status.

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
    progress_percentage INTEGER DEFAULT 0,
    metadata JSONB,
    created_by VARCHAR(255) NOT NULL,
    
    CONSTRAINT valid_status CHECK (task_status IN ('queueing', 'processing', 'completed', 'cancelled', 'failed')),
    CONSTRAINT valid_progress CHECK (progress_percentage >= 0 AND progress_percentage <= 100),
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
```

#### Task Status Flow
- **queueing**: Task created, waiting for processing
- **processing**: Task is being executed
- **completed**: Task finished successfully
- **cancelled**: Task cancelled by user
- **failed**: Task failed with error

## Processing Tables

### 2. chat_evaluation_input
Stores parsed content from Excel files for chat evaluation tasks.

```sql
CREATE TABLE chat_evaluation_input (
    id BIGSERIAL PRIMARY KEY,
    task_id UUID NOT NULL,
    row_number INTEGER NOT NULL,
    question TEXT NOT NULL,
    golden_answer TEXT NOT NULL,
    golden_citations JSONB NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    -- Constraints
    CONSTRAINT unique_task_row_content UNIQUE (task_id, row_number),
    CONSTRAINT fk_task_content FOREIGN KEY (task_id) REFERENCES tasks(id) ON DELETE CASCADE,
    CONSTRAINT valid_row_number CHECK (row_number > 0)
);

-- Indexes
CREATE INDEX idx_chat_eval_input_task_id ON chat_evaluation_input(task_id);
CREATE INDEX idx_chat_eval_input_row_number ON chat_evaluation_input(task_id, row_number);
```

### 3. chat_evaluation_output
Stores processing results and similarity scores for each evaluated question.

```sql
CREATE TABLE chat_evaluation_output (
    id BIGSERIAL PRIMARY KEY,
    task_id UUID NOT NULL,
    row_number INTEGER NOT NULL,
    api_answer TEXT NOT NULL,
    api_citations JSONB NOT NULL,
    answer_similarity DECIMAL(5,4) NOT NULL,
    citation_similarity DECIMAL(5,4) NOT NULL,
    processing_time_ms INTEGER,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    -- Constraints
    CONSTRAINT unique_task_row_results UNIQUE (task_id, row_number),
    CONSTRAINT fk_task_results FOREIGN KEY (task_id) REFERENCES tasks(id) ON DELETE CASCADE,
    CONSTRAINT valid_row_number_results CHECK (row_number > 0),
    CONSTRAINT valid_answer_similarity CHECK (answer_similarity >= 0 AND answer_similarity <= 1),
    CONSTRAINT valid_citation_similarity CHECK (citation_similarity >= 0 AND citation_similarity <= 1),
    CONSTRAINT valid_processing_time CHECK (processing_time_ms >= 0)
);

-- Indexes
CREATE INDEX idx_chat_eval_output_task_id ON chat_evaluation_output(task_id);
CREATE INDEX idx_chat_eval_output_row_number ON chat_evaluation_output(task_id, row_number);
CREATE INDEX idx_chat_eval_output_similarity ON chat_evaluation_output(answer_similarity, citation_similarity);
```

## URL Cleaning Tables

### 4. url_cleaning_input
Stores original URLs to be cleaned from Excel files.

```sql
CREATE TABLE url_cleaning_input (
    id BIGSERIAL PRIMARY KEY,
    task_id UUID NOT NULL,
    row_number INTEGER NOT NULL,
    url TEXT NOT NULL,
    parsed_title VARCHAR(500),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    -- Constraints
    CONSTRAINT unique_task_row_url_data UNIQUE (task_id, row_number),
    CONSTRAINT fk_task_url_data FOREIGN KEY (task_id) REFERENCES tasks(id) ON DELETE CASCADE,
    CONSTRAINT valid_row_number_url CHECK (row_number > 0),
    CONSTRAINT valid_url_length CHECK (LENGTH(url) <= 2000)
);

-- Indexes
CREATE INDEX idx_url_cleaning_input_task_id ON url_cleaning_input(task_id);
CREATE INDEX idx_url_cleaning_input_row_number ON url_cleaning_input(task_id, row_number);
CREATE INDEX idx_url_cleaning_input_url ON url_cleaning_input USING hash(url);
```

### 5. url_cleaning_output
Stores cleaning results, success/failure status, and processing methods.

```sql
CREATE TABLE url_cleaning_output (
    id BIGSERIAL PRIMARY KEY,
    task_id UUID NOT NULL,
    row_number INTEGER NOT NULL,
    original_url TEXT NOT NULL,
    cleaned_url TEXT,
    status VARCHAR(20) NOT NULL,
    method VARCHAR(30) NOT NULL,
    processing_notes TEXT,
    processing_time_ms INTEGER,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    -- Constraints
    CONSTRAINT unique_task_row_url_results UNIQUE (task_id, row_number),
    CONSTRAINT fk_task_url_results FOREIGN KEY (task_id) REFERENCES tasks(id) ON DELETE CASCADE,
    CONSTRAINT valid_row_number_url_results CHECK (row_number > 0),
    CONSTRAINT valid_status CHECK (status IN ('success', 'failed', 'api_error', 'url_parse_error')),
    CONSTRAINT valid_method CHECK (method IN ('direct_search', 'title_search', 'both_failed', 'api_unavailable', 'url_parse_error')),
    CONSTRAINT valid_url_length_results CHECK (LENGTH(original_url) <= 2000),
    CONSTRAINT valid_cleaned_url_length CHECK (cleaned_url IS NULL OR LENGTH(cleaned_url) <= 2000),
    CONSTRAINT valid_processing_time_url CHECK (processing_time_ms >= 0)
);

-- Indexes
CREATE INDEX idx_url_cleaning_output_task_id ON url_cleaning_output(task_id);
CREATE INDEX idx_url_cleaning_output_row_number ON url_cleaning_output(task_id, row_number);
CREATE INDEX idx_url_cleaning_output_status ON url_cleaning_output(status);
CREATE INDEX idx_url_cleaning_output_method ON url_cleaning_output(method);
CREATE INDEX idx_url_cleaning_output_original_url ON url_cleaning_output USING hash(original_url);
```

## Table Relationships and Data Flow

### Relationship Diagram
```
tasks (main table)
├── chat_evaluation_input (1:many)
│   └── chat_evaluation_output (1:1 via task_id + row_number)
└── url_cleaning_input (1:many)
    └── url_cleaning_output (1:1 via task_id + row_number)
```

### Data Flow Pattern
1. **Content Tables**: Store parsed Excel data (batch insert)
2. **Results Tables**: Store processing outcomes (sequential insert)
3. **Final Report**: JOIN content + results for Excel generation

## Performance Considerations

### Partitioning Strategy (for high volume)
```sql
-- Partition by task_id hash for better performance with large datasets
CREATE TABLE chat_evaluation_input_partitioned (
    LIKE chat_evaluation_input INCLUDING ALL
) PARTITION BY HASH (task_id);

-- Create partitions
CREATE TABLE chat_evaluation_input_p0 PARTITION OF chat_evaluation_input_partitioned
    FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE chat_evaluation_input_p1 PARTITION OF chat_evaluation_input_partitioned
    FOR VALUES WITH (MODULUS 4, REMAINDER 1);
-- ... continue for p2, p3
```

### Cleanup Strategy
```sql
-- Auto-cleanup old completed tasks (run daily)
DELETE FROM chat_evaluation_input 
WHERE task_id IN (
    SELECT id FROM tasks 
    WHERE status = 'completed' 
    AND completed_at < NOW() - INTERVAL '30 days'
);

DELETE FROM url_cleaning_input 
WHERE task_id IN (
    SELECT id FROM tasks 
    WHERE status = 'completed' 
    AND completed_at < NOW() - INTERVAL '30 days'
);
```

## Data Types and Constraints Explanation

### JSONB Fields
- **golden_citations**: Array of citation URLs `["url1", "url2", "url3"]`
- **api_citations**: Array of API-returned citation URLs

### Similarity Scores
- **DECIMAL(5,4)**: Allows values like 0.8567 (4 decimal places)
- **Range**: 0.0000 to 1.0000 (0% to 100% similarity)

### Status Enums
- **Chat Evaluation**: Always 'success' (failures stop processing)
- **URL Cleaning**: 'success', 'failed', 'api_error', 'url_parse_error'

### Processing Methods
- **direct_search**: Found via direct URL search
- **title_search**: Found via extracted title search  
- **both_failed**: Neither search method worked
- **api_unavailable**: Glean Platform Services was down
- **url_parse_error**: Could not parse URL structure 

### URL Cleaning Status Values
- **success**: URL was successfully cleaned through search
- **failed**: Both direct and title-based searches failed
- **url_parse_error**: Original URL could not be parsed or validated
- **api_error**: Glean Platform Services was unavailable during processing
- **timeout_error**: Search request timed out
- **api_unavailable**: Glean Platform Services was down 