# Database Schema Design - Task Processing Tables

## Overview
This document defines the database schema for task-specific processing tables. Each task type has two tables: one for storing parsed content from Excel files, and another for storing processing results.

## Chat Evaluation Tables

### 1. chat_evaluation_input
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

### 2. chat_evaluation_output
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

### 3. url_cleaning_input
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

### 4. url_cleaning_output
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