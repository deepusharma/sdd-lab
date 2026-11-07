# API Contract: Task CRUD REST API

**Date**: 2026-08-02
**Version**: 1.0.0
**Base URL**: `http://localhost:8000`

## Overview

This document defines the REST API contract for the Task CRUD API. All endpoints follow RESTful conventions, use JSON for request/response bodies, and return standard HTTP status codes.

## General Conventions

### Content Type
- **Request**: `Content-Type: application/json`
- **Response**: `Content-Type: application/json`

### Status Codes
- `200 OK`: Successful GET, PATCH, DELETE operations
- `201 Created`: Successful POST (create) operation
- `400 Bad Request`: Invalid input (validation errors, malformed JSON)
- `404 Not Found`: Task ID does not exist
- `500 Internal Server Error`: Database errors, server failures

### Error Response Format
All error responses follow this structure:
```json
{
  "error": "Error type",
  "detail": "Specific, actionable error message"
}
```

### Common Headers
**Request Logging**: All requests are logged with method, path, status code, and duration (FR-014).

## Endpoints

### 1. Create Task

**Endpoint**: `POST /tasks`

**Description**: Create a new task with a title and optional description.

**Request Body**:
```json
{
  "title": "string (required, max 500 chars)",
  "description": "string (optional, max 2000 chars)"
}
```

**Success Response** (`201 Created`):
```json
{
  "id": 1,
  "title": "Buy groceries",
  "description": "Milk, eggs, bread",
  "completed": false,
  "created_at": "2026-08-02T14:30:00Z",
  "updated_at": "2026-08-02T14:30:00Z"
}
```

**Error Responses**:
- `400 Bad Request`: Title missing, title > 500 chars, description > 2000 chars, invalid JSON

**Examples**:
```bash
# Minimal request (title only)
curl -X POST http://localhost:8000/tasks \
  -H "Content-Type: application/json" \
  -d '{"title": "Buy groceries"}'

# Full request (title + description)
curl -X POST http://localhost:8000/tasks \
  -H "Content-Type: application/json" \
  -d '{"title": "Buy groceries", "description": "Milk, eggs, bread"}'
```

---

### 2. List Tasks

**Endpoint**: `GET /tasks`

**Description**: Retrieve all tasks, optionally filtered by completion status and limited by count.

**Query Parameters**:
- `status` (optional): Filter by completion status
  - Values: `complete`, `incomplete`
  - Default: No filter (return all tasks)
- `limit` (optional): Maximum number of tasks to return
  - Type: Integer > 0
  - Default: No limit (return all tasks)

**Success Response** (`200 OK`):
```json
[
  {
    "id": 2,
    "title": "Write documentation",
    "description": null,
    "completed": true,
    "created_at": "2026-08-02T15:00:00Z",
    "updated_at": "2026-08-02T15:30:00Z"
  },
  {
    "id": 1,
    "title": "Buy groceries",
    "description": "Milk, eggs, bread",
    "completed": false,
    "created_at": "2026-08-02T14:30:00Z",
    "updated_at": "2026-08-02T14:30:00Z"
  }
]
```

**Empty List** (`200 OK`):
```json
[]
```

**Ordering**: Tasks ordered by `created_at` descending (most recent first)

**Error Responses**:
- `400 Bad Request`: Invalid query parameter values

**Examples**:
```bash
# Get all tasks
curl http://localhost:8000/tasks

# Get only incomplete tasks
curl http://localhost:8000/tasks?status=incomplete

# Get first 10 tasks
curl http://localhost:8000/tasks?limit=10

# Get incomplete tasks, limited to 5
curl "http://localhost:8000/tasks?status=incomplete&limit=5"
```

---

### 3. Toggle Task Completion Status

**Endpoint**: `PATCH /tasks/{id}/complete`

**Description**: Toggle a task's completion status bidirectionally (complete ↔ incomplete).

**Path Parameters**:
- `id` (required): Task ID (integer)

**Request Body**:
```json
{
  "completed": true  // or false
}
```

**Success Response** (`200 OK`):
```json
{
  "id": 1,
  "title": "Buy groceries",
  "description": "Milk, eggs, bread",
  "completed": true,
  "created_at": "2026-08-02T14:30:00Z",
  "updated_at": "2026-08-02T16:00:00Z"
}
```

**Idempotency**: Marking a task with the same status it already has returns success without error.

**Error Responses**:
- `400 Bad Request`: Invalid completed value (not boolean), invalid JSON
- `404 Not Found`: Task ID does not exist

**Examples**:
```bash
# Mark task as complete
curl -X PATCH http://localhost:8000/tasks/1/complete \
  -H "Content-Type: application/json" \
  -d '{"completed": true}'

# Mark task as incomplete
curl -X PATCH http://localhost:8000/tasks/1/complete \
  -H "Content-Type: application/json" \
  -d '{"completed": false}'
```

---

### 4. Update Task Content

**Endpoint**: `PATCH /tasks/{id}`

**Description**: Update a task's title and/or description without affecting completion status.

**Path Parameters**:
- `id` (required): Task ID (integer)

**Request Body** (at least one field required):
```json
{
  "title": "string (optional, max 500 chars)",
  "description": "string (optional, max 2000 chars, null to clear)"
}
```

**Success Response** (`200 OK`):
```json
{
  "id": 1,
  "title": "Buy almond milk",
  "description": "Updated description",
  "completed": false,
  "created_at": "2026-08-02T14:30:00Z",
  "updated_at": "2026-08-02T16:15:00Z"
}
```

**Error Responses**:
- `400 Bad Request`: Empty title, title > 500 chars, description > 2000 chars, no fields provided, invalid JSON
- `404 Not Found`: Task ID does not exist

**Examples**:
```bash
# Update title only
curl -X PATCH http://localhost:8000/tasks/1 \
  -H "Content-Type: application/json" \
  -d '{"title": "Buy almond milk"}'

# Update description only
curl -X PATCH http://localhost:8000/tasks/1 \
  -H "Content-Type: application/json" \
  -d '{"description": "Updated description"}'

# Update both
curl -X PATCH http://localhost:8000/tasks/1 \
  -H "Content-Type: application/json" \
  -d '{"title": "Buy almond milk", "description": "Updated description"}'

# Clear description
curl -X PATCH http://localhost:8000/tasks/1 \
  -H "Content-Type: application/json" \
  -d '{"description": null}'
```

---

### 5. Delete Task

**Endpoint**: `DELETE /tasks/{id}`

**Description**: Permanently delete a task.

**Path Parameters**:
- `id` (required): Task ID (integer)

**Success Response** (`200 OK`):
```json
{
  "message": "Task deleted successfully",
  "id": 1
}
```

**Error Responses**:
- `404 Not Found`: Task ID does not exist

**Examples**:
```bash
# Delete task
curl -X DELETE http://localhost:8000/tasks/1
```

---

## Validation Rules

### Title Validation
- Required for create and must not be empty string
- Maximum length: 500 characters (FR-007a)
- Cannot be empty on update (if provided)

### Description Validation
- Optional for all operations
- Maximum length: 2000 characters
- Can be null

### ID Validation
- Must be a positive integer
- Validated automatically by path parameter type

### Completed Validation
- Must be boolean (true/false)
- Not nullable

## Error Messages

### Validation Errors (400)
```json
{
  "error": "Validation Error",
  "detail": "Title exceeds 500 character limit"
}
```

```json
{
  "error": "Validation Error",
  "detail": "Title is required and cannot be empty"
}
```

### Not Found Errors (404)
```json
{
  "error": "Not Found",
  "detail": "Task with ID 999 does not exist"
}
```

### Server Errors (500)
```json
{
  "error": "Internal Server Error",
  "detail": "An unexpected error occurred"
}
```

**Note**: Stack traces are never exposed in error responses (FR-013).

## Security Notes

1. **Input Sanitization**: All inputs validated and sanitized (FR-011)
2. **SQL Injection Prevention**: Parameterized queries used exclusively
3. **No Authentication**: All endpoints publicly accessible (single-user assumption)
4. **Error Handling**: No internal details exposed in error messages

## Performance Expectations

See [../spec.md](../spec.md) Success Criteria for detailed performance targets:
- Task operations < 1 second (SC-001, SC-003, SC-004, SC-004a)
- List operations < 2 seconds for up to 100 tasks (SC-002)
- 50 concurrent requests supported (SC-006)
