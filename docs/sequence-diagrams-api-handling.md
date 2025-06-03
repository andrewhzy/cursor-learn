# Task Management API Request Handling Sequence Diagrams

This document contains sequence diagrams for the task management API endpoints, showing the interaction flow between frontend applications, API service, and database with simplified internal processing.

## Participants Legend
- **Frontend**: Internal frontend applications (Task Management UI)
- **API**: Main task management API service (handles all processing internally)
- **DB**: Primary Database (tasks and files)


## Endpoints 

| Method | Endpoint | Description | Auth Level |
|--------|----------|-------------|------------|
| POST   | `/rest/v1/tasks` | Upload Excel file and create tasks | User |
| GET    | `/rest/v1/tasks` | List user's tasks with filtering (no blob data) | User |
| GET    | `/rest/v1/tasks/{id}` | Get specific task details with blob files | User |
| PUT    | `/rest/v1/tasks/{id}` | Update/cancel a task | Owner |
| DELETE | `/rest/v1/tasks/{id}` | Delete a task | Owner |


## 1. POST /rest/v1/tasks - Excel File Upload and Task Creation

### Success Flow - Multi-Sheet Excel Upload
```mermaid
sequenceDiagram
    participant FE as Frontend App
    participant API as Backend API
    participant DB as Database
    
    FE->>API: POST /rest/v1/tasks<br/>Authorization: Bearer <jwt_token><br/>Content-Type: multipart/form-data<br/>File: monthly_reports.xlsx
    API->>API: Authenticate and Extract user context from JWT<br/>user_id: "user_123"
    API->>API: Validate file format and size
    API->>API: Parse and validate Excel file content
    API->>API: Split file into individual sheet files as blobs <br/> e.g. 3 sheets: ["January", "February", "March"]
    
    API->>DB: BEGIN TRANSACTION
    API->>API: Generate upload_batch_id
    
    loop For each sheet (3 sheets)
        API->>DB: INSERT task<br/>(user_id: "user_123", sheet_name, task_file_blob, upload_batch_id, status: "pending")
        DB-->>API: Return task_id
    end
    
    API->>DB: COMMIT TRANSACTION
    API-->>FE: 201 Created<br/>{upload_batch_id, tasks: [3 tasks], total_sheets: 3}
```

### Error Flow - Invalid Excel File
```mermaid
sequenceDiagram
    participant FE as Frontend App
    participant API as Backend API
    
    FE->>API: POST /rest/v1/tasks<br/>File: document.pdf (wrong format)
    API->>API: Extract user context from JWT
    API->>API: Validate file format - FAILED<br/>(not Excel format)
    API-->>FE: 400 Bad Request<br/>{"error": {"code": "INVALID_FILE_FORMAT", "message": "File must be Excel format (.xlsx or .xls)"}}
```

### Error Flow - Database Transaction Failure and Rollback
```mermaid
sequenceDiagram
    participant FE as Frontend App
    participant API as Backend API
    participant DB as Database
    
    FE->>API: POST /rest/v1/tasks<br/>Authorization: Bearer <jwt_token><br/>Content-Type: multipart/form-data<br/>File: monthly_reports.xlsx
    API->>API: Authenticate and Extract user context from JWT<br/>user_id: "user_123"
    API->>API: Validate file format and size
    API->>API: Parse and validate Excel file content
    API->>API: Split file into individual sheet files as blobs <br/> e.g. 3 sheets: ["January", "February", "March"]
    
    API->>DB: BEGIN TRANSACTION
    API->>API: Generate upload_batch_id
    
    API->>DB: INSERT task 1<br/>(user_id: "user_123", sheet_name: "January", task_file_blob, upload_batch_id)
    DB-->>API: Success - task_id: "task_1"
    
    API->>DB: INSERT task 2<br/>(user_id: "user_123", sheet_name: "February", task_file_blob, upload_batch_id)
    DB-->>API: Success - task_id: "task_2"
    
    API->>DB: INSERT task 3<br/>(user_id: "user_123", sheet_name: "March", task_file_blob, upload_batch_id)
    DB-->>API: ❌ ERROR: Transaction Failure<br/>(e.g., duplicate key, storage limit exceeded, connection timeout)
    
    API->>DB: ROLLBACK TRANSACTION
    DB-->>API: Transaction rolled back successfully<br/>All tasks removed from database
    
    API->>API: Clean up temporary file blobs<br/>Release allocated resources
    API-->>FE: 500 Internal Server Error<br/>{"error": {"code": "DATABASE_ERROR", <br/>"message": "Failed to create tasks due to database error", <br/>"details": "Transaction rolled back"}}
```


## 2. GET /rest/v1/tasks - List User Tasks (No Blob Data)

### Success Flow with Filtering
```mermaid
sequenceDiagram
    participant FE as Frontend App
    participant API as Backend API
    participant DB as Database
    
    FE->>API: GET /rest/v1/tasks?status=pending&page=1&per_page=20<br/>Authorization: Bearer <jwt_token>
    API->>API: Extract user context from JWT<br/>user_id: "user_123"
    API->>API: Parse query parameters and build filters
    
    API->>DB: SELECT id, user_id, original_filename, sheet_name, task_file_size, results_file_size, task_status, progress_percentage, created_at, updated_at<br/>FROM tasks WHERE user_id = 'user_123' AND status = 'pending'<br/>ORDER BY created_at DESC LIMIT 20 OFFSET 0<br/>(excludes blob columns for performance)
    DB-->>API: Return task list (5 pending tasks without blob data)
    
    API->>DB: SELECT COUNT(*)<br/>WHERE user_id = 'user_123' AND status = 'pending'
    DB-->>API: Return total count (25)
    
    API->>API: Build paginated response
    API-->>FE: 200 OK<br/>{data: [5 tasks], meta: {page: 1, total: 25, has_next: true}}
```

## 3. GET /rest/v1/tasks/{id} - Get Task Details with Blob Data

### Success Flow
```mermaid
sequenceDiagram
    participant FE as Frontend App
    participant API as Backend API
    participant DB as Database
    
    FE->>API: GET /rest/v1/tasks/123e4567-e89b-12d3-a456-426614174000<br/>Authorization: Bearer <jwt_token>
    API->>API: Extract user context from JWT<br/>user_id: "user_123", task_id: "123e4567"
    API->>API: Validate task ID format
    
    API->>DB: SELECT * FROM tasks<br/>WHERE id = '123e4567' AND user_id = 'user_123'<br/>(includes task_file_blob and results_file_blob)
    DB-->>API: Return complete task details<br/>{id, sheet_name: "January", status: "processing", progress: 75%, task_file_blob, results_file_blob, metadata}
    
    API->>API: Encode blob data as base64 for JSON response
    API-->>FE: 200 OK<br/>{task details with all metadata and blob data}
```

### Error Flow - Task Not Found or Access Denied
```mermaid
sequenceDiagram
    participant FE as Frontend App
    participant API as Backend API
    participant DB as Database
    
    FE->>API: GET /rest/v1/tasks/other-user-task<br/>Authorization: Bearer <jwt_token>
    API->>API: Extract user context from JWT<br/>user_id: "user_123"
    
    API->>DB: SELECT * FROM tasks<br/>WHERE id = 'other-user-task' AND user_id = 'user_123'
    DB-->>API: No results (task not found or belongs to another user)
    
    API-->>FE: 404 Not Found<br/>{"error": {"code": "TASK_NOT_FOUND", "message": "Task not found"}}
```

## 4. PUT /rest/v1/tasks/{id} - Update/Cancel Task

### Success Flow - Cancel Task
```mermaid
sequenceDiagram
    participant FE as Frontend App
    participant API as Backend API
    participant DB as Database
    
    FE->>API: PUT /rest/v1/tasks/123e4567<br/>Authorization: Bearer <jwt_token><br/>Content-Type: application/json<br/>{"action": "cancel"}
    API->>API: Extract user context from JWT<br/>user_id: "user_123", task_id: "123e4567"
    API->>API: Parse request body and validate action
    
    API->>DB: SELECT task<br/>WHERE id = '123e4567' AND user_id = 'user_123' FOR UPDATE
    DB-->>API: Return task<br/>{id, status: "processing", user_id: "user_123"}
    
    API->>API: Check if task can be cancelled<br/>(status = "pending" or "processing")
    API->>API: Stop internal task processing<br/>Clean up background operations
    
    API->>DB: UPDATE task<br/>SET status = 'cancelled', cancelled_at = NOW(), updated_at = NOW()<br/>WHERE id = '123e4567'
    DB-->>API: Update successful
    
    API->>DB: SELECT updated task (without blobs for response)
    DB-->>API: Return updated task with status "cancelled"
    
    API-->>FE: 200 OK<br/>{updated task with status "cancelled"}
```

### Error Flow - Cannot Cancel Completed Task
```mermaid
sequenceDiagram
    participant FE as Frontend App
    participant API as Backend API
    participant DB as Database
    
    FE->>API: PUT /rest/v1/tasks/completed-task<br/>Authorization: Bearer <jwt_token><br/>{"action": "cancel"}
    API->>API: Extract user context from JWT
    
    API->>DB: SELECT task<br/>WHERE id = 'completed-task' AND user_id = 'user_123'
    DB-->>API: Return task<br/>{status: "completed"}
    
    API->>API: Check cancellation eligibility<br/>Cannot cancel completed task
    API-->>FE: 400 Bad Request<br/>{"error": {"code": "INVALID_STATUS", "message": "Cannot cancel completed task"}}
```

## 5. DELETE /rest/v1/tasks/{id} - Delete Task

### Success Flow
```mermaid
sequenceDiagram
    participant FE as Frontend App
    participant API as Backend API
    participant DB as Database
    
    FE->>API: DELETE /rest/v1/tasks/123e4567<br/>Authorization: Bearer <jwt_token>
    API->>API: Extract user context from JWT<br/>user_id: "user_123", task_id: "123e4567"
    
    API->>DB: SELECT task<br/>WHERE id = '123e4567' AND user_id = 'user_123' FOR UPDATE
    DB-->>API: Return task<br/>{status: "cancelled", user_id: "user_123", task_file_blob, results_file_blob}
    
    API->>API: Check if task can be deleted<br/>(status != "processing")
    API->>API: Clean up file blobs and internal references
    
    API->>DB: DELETE FROM tasks<br/>WHERE id = '123e4567'
    DB-->>API: Delete successful
    
    API-->>FE: 204 No Content
```

### Error Flow - Cannot Delete Processing Task
```mermaid
sequenceDiagram
    participant FE as Frontend App
    participant API as Backend API
    participant DB as Database
    
    FE->>API: DELETE /rest/v1/tasks/processing-task<br/>Authorization: Bearer <jwt_token>
    API->>API: Extract user context from JWT
    
    API->>DB: SELECT task<br/>WHERE id = 'processing-task' AND user_id = 'user_123'
    DB-->>API: Return task<br/>{status: "processing"}
    
    API->>API: Check deletion eligibility<br/>Cannot delete processing task
    API-->>FE: 400 Bad Request<br/>{"error": {"code": "INVALID_STATUS", "message": "Cannot delete task in processing status"}}
```

## Request/Response Flow Summary

### Key Features Demonstrated
- **RESTful Endpoints**: All endpoints follow `/rest/v1/` pattern with proper HTTP methods
- **JWT Authentication**: User context extraction and validation on every request
- **User Ownership**: Tasks are scoped to the authenticated user
- **Internal Processing**: All file parsing and task processing handled within the main API
- **Performance Optimization**: List endpoint excludes blob data, detail endpoint includes blobs
- **Error Handling**: Comprehensive error responses with proper HTTP status codes
- **Database Transactions**: Atomic operations for data consistency
- **File Management**: Blob storage and retrieval handled internally with separate fields for task and results

### Performance Characteristics
- **Optimized Listing**: GET /tasks excludes blob data for fast pagination
- **Complete Details**: GET /tasks/{id} includes all blob data for full task information
- **Simplified Architecture**: No external service dependencies
- **Direct Database Access**: Minimal latency for data operations
- **Internal File Processing**: No network overhead for file operations
- **Background Processing**: Non-blocking task execution with results storage
- **Efficient Querying**: Optimized database queries with proper indexing 