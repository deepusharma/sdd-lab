# Data Model: Task CRUD API

**Date**: 2026-08-02
**Status**: Complete

## Overview

This document defines the data structures for the Task CRUD API. There is one primary entity (Task) with clear validation rules and state transitions.

## Entities

### Task

Represents a work item or todo with completion tracking.

**Fields**:

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | Integer | Primary key, auto-increment, not null | Unique identifier assigned on creation |
| `title` | Text | Required, max 500 characters | Task title/description |
| `description` | Text | Optional, max 2000 characters | Extended task details |
| `completed` | Boolean | Not null, default false | Completion status (true=complete, false=incomplete) |
| `created_at` | DateTime | Not null, default CURRENT_TIMESTAMP, UTC | When task was created |
| `updated_at` | DateTime | Not null, default CURRENT_TIMESTAMP, auto-update on modification, UTC | Last modification time |

**Validation Rules** (enforced at API layer):

- **VR-001**: Title is required (cannot be null or empty string)
- **VR-002**: Title length ≤ 500 characters (FR-007a)
- **VR-003**: Description length ≤ 2000 characters if provided
- **VR-004**: Completed must be boolean (true/false)
- **VR-005**: ID is read-only (assigned by system, cannot be set by user)
- **VR-006**: Timestamps are read-only (managed by database)

**Database Schema** (SQLite DDL):

```sql
CREATE TABLE IF NOT EXISTS tasks (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    title TEXT NOT NULL CHECK(length(title) <= 500 AND length(title) > 0),
    description TEXT CHECK(description IS NULL OR length(description) <= 2000),
    completed BOOLEAN NOT NULL DEFAULT 0,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Trigger to auto-update updated_at timestamp
CREATE TRIGGER IF NOT EXISTS update_task_timestamp
AFTER UPDATE ON tasks
FOR EACH ROW
BEGIN
    UPDATE tasks SET updated_at = CURRENT_TIMESTAMP WHERE id = NEW.id;
END;
```

## State Transitions

### Task Lifecycle

```
[Created] ---> [Incomplete] <---> [Complete]
                    |
                    v
                [Deleted]
```

**Transition Rules**:

1. **Create**: Task starts in `Incomplete` state (`completed=false`)
2. **Toggle Completion**: Can move bidirectionally between `Incomplete` and `Complete` (User Story 3)
3. **Update Content**: Title/description can be modified without affecting completion state (User Story 5)
4. **Delete**: Permanent removal from database (User Story 4)

**Idempotency**:
- Marking a task as complete when already complete → No-op, returns success (acceptance scenario 3 of User Story 3)
- Marking a task as incomplete when already incomplete → No-op, returns success

## Relationships

**None** - Task is a standalone entity with no foreign key relationships. This is a single-user system with no user accounts, task categories, or assignments.

## Data Access Patterns

Based on user stories and functional requirements, the following queries are needed:

### Pattern 1: Create Task
```sql
INSERT INTO tasks (title, description)
VALUES (?, ?);
```
Returns: newly created task with auto-generated ID

### Pattern 2: List All Tasks
```sql
SELECT id, title, description, completed, created_at, updated_at
FROM tasks
ORDER BY created_at DESC
LIMIT ?;
```
Default ordering: Most recent first (most recent `created_at`)

### Pattern 3: List Tasks by Completion Status
```sql
SELECT id, title, description, completed, created_at, updated_at
FROM tasks
WHERE completed = ?
ORDER BY created_at DESC
LIMIT ?;
```
Filter parameter: `completed` (0 for incomplete, 1 for complete per SQLite boolean storage)

### Pattern 4: Get Single Task
```sql
SELECT id, title, description, completed, created_at, updated_at
FROM tasks
WHERE id = ?;
```
Returns: single task or 404 if not found

### Pattern 5: Toggle Completion Status
```sql
UPDATE tasks
SET completed = ?
WHERE id = ?;
```
Input: new completion value (true/false)

### Pattern 6: Update Task Content
```sql
UPDATE tasks
SET title = ?, description = ?
WHERE id = ?;
```
Trigger automatically updates `updated_at`

### Pattern 7: Delete Task
```sql
DELETE FROM tasks
WHERE id = ?;
```
Returns: success confirmation or 404 if not found

## Data Constraints Summary

| Constraint | Source | Enforcement |
|------------|--------|-------------|
| Title required | FR-001, VR-001 | API validation (Pydantic) + DB CHECK constraint |
| Title ≤ 500 chars | FR-007a, VR-002 | API validation (Pydantic) + DB CHECK constraint |
| Description ≤ 2000 chars | Assumptions, VR-003 | API validation (Pydantic) + DB CHECK constraint |
| ID uniqueness | FR-002 | DB PRIMARY KEY |
| Timestamps auto-managed | Task entity definition | DB DEFAULT + TRIGGER |
| Parameterized queries only | FR-011, Constitution V | Code review enforcement |

## Serialization Format

**JSON Response Schema** (all endpoints return tasks in this format):

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

**Field Notes**:
- `created_at` and `updated_at` are ISO 8601 formatted UTC timestamps
- `completed` is boolean (not 0/1)
- `description` may be null if not provided

## Error Cases

| Scenario | Behavior | HTTP Status |
|----------|----------|-------------|
| Missing title on create | Return validation error | 400 |
| Title > 500 characters | Return validation error | 400 |
| Description > 2000 characters | Return validation error | 400 |
| Invalid task ID (non-integer) | Return validation error | 400 |
| Task ID not found | Return not found error | 404 |
| Database file corrupted | Return server error | 500 |
