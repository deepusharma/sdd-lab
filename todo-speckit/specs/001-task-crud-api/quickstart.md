# Quickstart: Task CRUD API Validation

**Date**: 2026-08-02
**Purpose**: Runnable validation scenarios to prove the feature works end-to-end

## Prerequisites

Before running validation scenarios:

1. **Python 3.11+** installed
   ```bash
   python --version  # Should show 3.11 or higher
   ```

2. **Dependencies installed**
   ```bash
   pip install fastapi uvicorn pydantic pytest pytest-asyncio httpx
   ```

3. **Project structure** (see [plan.md](plan.md) for complete structure)
   ```
   todo-api/
   ├── app/
   │   ├── main.py
   │   ├── models/
   │   ├── database/
   │   ├── routers/
   │   └── middleware/
   └── tests/
       └── integration/
           └── test_tasks_api.py
   ```

## Setup

### 1. Start the API Server

From the project root:

```bash
# Development server with auto-reload
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

Expected output:
```
INFO:     Started server process [12345]
INFO:     Waiting for application startup.
INFO:     Database initialized successfully  # From FR-014a
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
```

Server is now ready at `http://localhost:8000`

### 2. Verify API Documentation

Open browser to `http://localhost:8000/docs` to view auto-generated Swagger UI.

## Validation Scenarios

Run these scenarios in order to validate all user stories end-to-end. Each scenario references the corresponding user story from [spec.md](spec.md).

### Scenario 1: Create Task (User Story 1 - P1)

**Goal**: Verify task creation with validation

```bash
# Create task with title only
curl -X POST http://localhost:8000/tasks \
  -H "Content-Type: application/json" \
  -d '{"title": "Buy groceries"}' \
  | jq

# Expected response (201 Created):
# {
#   "id": 1,
#   "title": "Buy groceries",
#   "description": null,
#   "completed": false,
#   "created_at": "2026-08-02T...",
#   "updated_at": "2026-08-02T..."
# }
```

**Validation checks**:
- ✓ Returns 201 status code
- ✓ Response includes auto-generated ID
- ✓ Default values set (completed=false, timestamps populated)
- ✓ Response time < 1 second (SC-001)

**Error case**: Missing title (Acceptance Scenario 2)
```bash
curl -X POST http://localhost:8000/tasks \
  -H "Content-Type: application/json" \
  -d '{}' \
  -w "\nHTTP Status: %{http_code}\n"

# Expected: 400 status with clear error message
# {
#   "error": "Validation Error",
#   "detail": "Title is required and cannot be empty"
# }
```

**Validation checks**:
- ✓ Returns 400 status code
- ✓ Error message is clear and actionable (SC-005)
- ✓ No stack trace exposed (FR-013)

---

### Scenario 2: List Tasks (User Story 2 - P2)

**Goal**: Verify task listing with filtering and limiting

```bash
# Create additional tasks for testing
curl -X POST http://localhost:8000/tasks \
  -H "Content-Type: application/json" \
  -d '{"title": "Write documentation", "description": "API docs"}'

curl -X POST http://localhost:8000/tasks \
  -H "Content-Type: application/json" \
  -d '{"title": "Run tests"}'

# List all tasks
curl http://localhost:8000/tasks | jq

# Expected response (200 OK):
# [
#   {"id": 3, "title": "Run tests", ...},
#   {"id": 2, "title": "Write documentation", ...},
#   {"id": 1, "title": "Buy groceries", ...}
# ]
# Note: Ordered by created_at DESC (most recent first)
```

**Validation checks**:
- ✓ Returns 200 status code
- ✓ All tasks present in response
- ✓ Ordered by creation date (newest first)
- ✓ Response time < 2 seconds for 3 tasks (SC-002)

**Filtering test** (Acceptance Scenario 4-5):
```bash
# Mark task 2 as complete first
curl -X PATCH http://localhost:8000/tasks/2/complete \
  -H "Content-Type: application/json" \
  -d '{"completed": true}'

# Filter incomplete tasks
curl "http://localhost:8000/tasks?status=incomplete" | jq

# Expected: Only tasks 1 and 3 (both incomplete)

# Filter complete tasks
curl "http://localhost:8000/tasks?status=complete" | jq

# Expected: Only task 2
```

**Limiting test** (Acceptance Scenario 3):
```bash
# Get first 2 tasks
curl "http://localhost:8000/tasks?limit=2" | jq

# Expected: Only 2 tasks returned
```

**Empty list test** (Acceptance Scenario 2):
```bash
# Delete all tasks first, then list
curl http://localhost:8000/tasks | jq

# Expected: [] (empty array) with 200 status
```

---

### Scenario 3: Toggle Completion Status (User Story 3 - P3)

**Goal**: Verify bidirectional status toggling

```bash
# Mark task 1 as complete
curl -X PATCH http://localhost:8000/tasks/1/complete \
  -H "Content-Type: application/json" \
  -d '{"completed": true}' \
  | jq

# Expected response (200 OK):
# {
#   "id": 1,
#   "title": "Buy groceries",
#   "completed": true,  # Changed from false
#   "updated_at": "..."  # New timestamp
# }
```

**Validation checks**:
- ✓ Completion status updated
- ✓ `updated_at` timestamp changed
- ✓ Response immediate (SC-003)

**Bidirectional toggle test** (Acceptance Scenario 2):
```bash
# Mark same task as incomplete
curl -X PATCH http://localhost:8000/tasks/1/complete \
  -H "Content-Type: application/json" \
  -d '{"completed": false}' \
  | jq

# Expected: completed=false, updated_at changed again
```

**Idempotency test** (Acceptance Scenario 3):
```bash
# Mark already-incomplete task as incomplete again
curl -X PATCH http://localhost:8000/tasks/1/complete \
  -H "Content-Type: application/json" \
  -d '{"completed": false}' \
  | jq

# Expected: Success (200), completed remains false
```

**Error case**: Non-existent task (Acceptance Scenario 4)
```bash
curl -X PATCH http://localhost:8000/tasks/999/complete \
  -H "Content-Type: application/json" \
  -d '{"completed": true}' \
  -w "\nHTTP Status: %{http_code}\n"

# Expected: 404 status with clear error
```

---

### Scenario 4: Update Task Content (User Story 5 - P5)

**Goal**: Verify title/description updates without affecting completion status

```bash
# Update title
curl -X PATCH http://localhost:8000/tasks/1 \
  -H "Content-Type: application/json" \
  -d '{"title": "Buy almond milk"}' \
  | jq

# Expected response (200 OK):
# {
#   "id": 1,
#   "title": "Buy almond milk",  # Changed
#   "completed": false,  # Unchanged
#   "updated_at": "..."  # New timestamp
# }
```

**Validation checks**:
- ✓ Title updated successfully
- ✓ Completion status preserved
- ✓ Changes persist immediately (SC-004a)

**Add description** (Acceptance Scenario 2):
```bash
curl -X PATCH http://localhost:8000/tasks/1 \
  -H "Content-Type: application/json" \
  -d '{"description": "Unsweetened, organic preferred"}' \
  | jq

# Expected: description field populated
```

**Validation error** (Acceptance Scenario 3):
```bash
# Title too long (>500 chars)
curl -X PATCH http://localhost:8000/tasks/1 \
  -H "Content-Type: application/json" \
  -d "{\"title\": \"$(python -c 'print("a"*501)')\"}" \
  -w "\nHTTP Status: %{http_code}\n"

# Expected: 400 status
# {
#   "error": "Validation Error",
#   "detail": "Title exceeds 500 character limit"
# }
```

**Empty title error** (Acceptance Scenario 5):
```bash
curl -X PATCH http://localhost:8000/tasks/1 \
  -H "Content-Type: application/json" \
  -d '{"title": ""}' \
  -w "\nHTTP Status: %{http_code}\n"

# Expected: 400 status
```

---

### Scenario 5: Delete Task (User Story 4 - P4)

**Goal**: Verify permanent task deletion

```bash
# Delete task 1
curl -X DELETE http://localhost:8000/tasks/1 | jq

# Expected response (200 OK):
# {
#   "message": "Task deleted successfully",
#   "id": 1
# }
```

**Validation checks**:
- ✓ Deletion confirmed
- ✓ Response time < 1 second (SC-004)

**Verify deletion** (Acceptance Scenario 2):
```bash
# List all tasks
curl http://localhost:8000/tasks | jq

# Expected: Task 1 no longer present
```

**Error case**: Delete non-existent task (Acceptance Scenario 3)
```bash
curl -X DELETE http://localhost:8000/tasks/999 \
  -w "\nHTTP Status: %{http_code}\n"

# Expected: 404 status
```

---

## Automated Test Execution

Run pytest integration tests to validate all scenarios programmatically:

```bash
# Run all tests
pytest tests/integration/test_tasks_api.py -v

# Expected output:
# test_create_task_success PASSED
# test_create_task_missing_title PASSED
# test_list_tasks PASSED
# test_filter_tasks_by_status PASSED
# test_toggle_completion_status PASSED
# test_update_task_content PASSED
# test_delete_task PASSED
# ... (one test per acceptance scenario)
```

**Success criteria**:
- ✓ All tests pass
- ✓ Each endpoint has at least one test (Principle III)
- ✓ Happy path + error case covered per endpoint (Principle III)

## Performance Validation

### Concurrency Test (SC-006)

```bash
# Create 50 concurrent requests using GNU parallel
seq 1 50 | parallel -j 50 \
  'curl -X POST http://localhost:8000/tasks \
   -H "Content-Type: application/json" \
   -d "{\"title\": \"Concurrent task {}\"}" -s -o /dev/null -w "%{http_code}\n"'

# Expected: All 50 requests return 201 status
# Verification: List tasks should show 50 tasks with no duplicates or data loss
```

### Persistence Test (SC-007)

```bash
# Create task
curl -X POST http://localhost:8000/tasks \
  -H "Content-Type: application/json" \
  -d '{"title": "Persistence test"}'

# Stop server (CTRL+C)
# Restart server
uvicorn app.main:app --reload

# List tasks
curl http://localhost:8000/tasks | jq

# Expected: "Persistence test" task still present
```

## Logging Verification (FR-014)

Check server console output for request logs:

```
INFO:     POST /tasks 201 45ms
INFO:     GET /tasks 200 12ms
INFO:     PATCH /tasks/1/complete 200 23ms
INFO:     DELETE /tasks/1 200 18ms
```

**Validation checks**:
- ✓ Every request logged
- ✓ Format includes method, path, status, duration
- ✓ Logged to stdout

## Database Initialization (FR-014a)

Check server startup logs:

```
INFO:     Database initialized successfully
```

Or if database already exists:
```
INFO:     Database connection established
```

If database file is manually deleted while server runs:
```
ERROR:    Database connection failed
INFO:     Returning 500 for all requests
```

## Success Criteria Checklist

Refer to [spec.md](spec.md) Success Criteria section for complete validation:

- [ ] SC-001: Task creation < 1 second ✓ (Scenario 1)
- [ ] SC-002: List 100 tasks < 2 seconds ✓ (Scenario 2)
- [ ] SC-003: Toggle completion immediate ✓ (Scenario 3)
- [ ] SC-004: Delete task < 1 second ✓ (Scenario 5)
- [ ] SC-004a: Update content immediate ✓ (Scenario 4)
- [ ] SC-005: Clear error messages ✓ (All error cases)
- [ ] SC-006: 50 concurrent requests ✓ (Concurrency test)
- [ ] SC-007: Data persists across restarts ✓ (Persistence test)

## Next Steps

After validating all scenarios:

1. Review [contracts/api.md](contracts/api.md) for complete API specification
2. Review [data-model.md](data-model.md) for database schema details
3. Proceed to `/speckit-tasks` to generate implementation tasks
