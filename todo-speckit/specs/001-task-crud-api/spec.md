# Feature Specification: Task CRUD API

**Feature Branch**: `001-task-crud-api`

**Created**: 2026-08-02

**Status**: Draft

**Input**: User description: "Build a task/todo REST API: create a task, list tasks, mark a task complete, delete a task. Back it with a simple database (SQLite is fine). No auth needed. Keep it small."

## Clarifications

### Session 2026-08-02

- Q: Can users toggle a task's completion status back to incomplete after marking it complete? → A: Yes, users can toggle between complete and incomplete status bidirectionally
- Q: What is the maximum allowed length for task titles? → A: 500 characters
- Q: Should users be able to filter the task list to show only complete or only incomplete tasks? → A: Yes, support optional ?status=complete or ?status=incomplete query parameter
- Q: How should the system respond when the database file is missing or corrupted at startup? → A: Automatically create new database file if missing; return 500 if corrupted
- Q: Should users be able to update a task's title or description after creation? → A: Yes, add PATCH/PUT endpoint to update title and/or description independently

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Create New Task (Priority: P1)

A user needs to capture a new task by sending task details to the API and receiving confirmation that the task was created.

**Why this priority**: Creating tasks is the foundational capability. Without it, no other task operations are possible. This delivers immediate value as a minimal task capture system.

**Independent Test**: Can be fully tested by sending a POST request with task data and verifying the response contains the created task with a unique identifier. Delivers value as a basic task logging system.

**Acceptance Scenarios**:

1. **Given** no tasks exist, **When** user submits a new task with title "Buy groceries", **Then** system returns the created task with a unique ID and the submitted title
2. **Given** the system is running, **When** user submits a task without a title, **Then** system returns a 400 error with a clear message indicating title is required
3. **Given** a task was just created, **When** user retrieves the task list, **Then** the newly created task appears in the results

---

### User Story 2 - List All Tasks (Priority: P2)

A user needs to retrieve all existing tasks to review what needs to be done.

**Why this priority**: Viewing tasks is essential for task management but depends on having tasks created first. This enables users to see their work items.

**Independent Test**: Can be fully tested by creating several tasks via the create endpoint, then calling the list endpoint and verifying all tasks are returned. Delivers value as a task review tool.

**Acceptance Scenarios**:

1. **Given** three tasks exist in the system, **When** user requests the task list, **Then** system returns all three tasks with their details
2. **Given** no tasks exist, **When** user requests the task list, **Then** system returns an empty list with a 200 status
3. **Given** 100 tasks exist, **When** user requests the task list with a limit of 10, **Then** system returns only the first 10 tasks
4. **Given** 5 complete and 3 incomplete tasks exist, **When** user requests the task list with status=incomplete filter, **Then** system returns only the 3 incomplete tasks
5. **Given** 5 complete and 3 incomplete tasks exist, **When** user requests the task list with status=complete filter, **Then** system returns only the 5 complete tasks

---

### User Story 3 - Toggle Task Completion Status (Priority: P3)

A user needs to mark a task as completed or incomplete to track progress and distinguish finished work from pending items. Users can toggle status bidirectionally to correct mistakes.

**Why this priority**: Completion tracking adds task lifecycle management. While valuable, users can manage tasks without it by deleting completed items.

**Independent Test**: Can be fully tested by creating a task, marking it complete via PATCH request, then unmarking it back to incomplete, verifying the status changes at each step. Delivers value as a progress tracking feature.

**Acceptance Scenarios**:

1. **Given** an incomplete task exists, **When** user marks it as complete, **Then** system updates the task status to complete and returns the updated task
2. **Given** a complete task exists, **When** user marks it as incomplete, **Then** system updates the task status to incomplete and returns the updated task
3. **Given** a task is already marked complete, **When** user marks it complete again, **Then** system accepts the request (idempotent operation)
4. **Given** a non-existent task ID, **When** user attempts to change its status, **Then** system returns a 404 error with a clear message

---

### User Story 4 - Delete Task (Priority: P4)

A user needs to permanently remove tasks that are no longer relevant or were created in error.

**Why this priority**: Deletion provides cleanup capability. Users can work effectively without it by ignoring unwanted tasks, making this the lowest priority core operation.

**Independent Test**: Can be fully tested by creating a task, deleting it via DELETE request, then verifying it no longer appears in the task list. Delivers value as a data hygiene tool.

**Acceptance Scenarios**:

1. **Given** a task exists, **When** user deletes the task, **Then** system removes the task and returns a success confirmation
2. **Given** a task was deleted, **When** user requests the task list, **Then** the deleted task does not appear
3. **Given** a non-existent task ID, **When** user attempts to delete it, **Then** system returns a 404 error with a clear message

---

### User Story 5 - Update Task Content (Priority: P5)

A user needs to modify a task's title or description after creation to correct mistakes or add details without recreating the task.

**Why this priority**: Content editing improves user experience by allowing corrections and refinements. However, users can work around this by deleting and recreating tasks, making it the lowest priority enhancement.

**Independent Test**: Can be fully tested by creating a task, updating its title or description via PATCH request, and verifying the changes persist. Delivers value as a content correction tool.

**Acceptance Scenarios**:

1. **Given** a task exists with title "Buy milk", **When** user updates the title to "Buy almond milk", **Then** system updates the task and returns the modified task with the new title
2. **Given** a task exists with no description, **When** user adds a description, **Then** system updates the task and returns it with the new description
3. **Given** a task exists, **When** user updates with a title exceeding 500 characters, **Then** system returns a 400 error with a clear validation message
4. **Given** a non-existent task ID, **When** user attempts to update it, **Then** system returns a 404 error with a clear message
5. **Given** a task exists, **When** user updates with an empty title, **Then** system returns a 400 error indicating title is required

---

### Edge Cases

- What happens when a user submits a task with title exceeding 500 characters? (System returns 400 with validation error per FR-007a)
- How does the system handle concurrent requests to mark the same task complete? (Idempotent behavior expected per User Story 3)
- Database file missing at startup: System auto-creates new database and logs the event
- Database file corrupted: System returns 500 error for all requests until manually repaired
- How does the system respond to malformed JSON in request bodies? (Returns 400 per FR-007)
- What happens when the database reaches storage limits? (Database error handling TBD during implementation)

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST provide an endpoint to create a new task with a title and optional description
- **FR-002**: System MUST assign a unique identifier to each created task
- **FR-003**: System MUST provide an endpoint to retrieve all tasks
- **FR-004**: System MUST provide an endpoint to toggle a task's completion status bidirectionally (complete ↔ incomplete)
- **FR-004a**: System MUST provide an endpoint to update a task's title and/or description independently of completion status
- **FR-005**: System MUST provide an endpoint to delete a task by its unique identifier
- **FR-006**: System MUST persist all task data in SQLite database
- **FR-006a**: System MUST automatically create the database file with proper schema if it does not exist at startup
- **FR-007**: System MUST validate all input and return 400 status for invalid requests
- **FR-007a**: System MUST reject task titles exceeding 500 characters with a clear validation error message
- **FR-008**: System MUST return 404 status when attempting operations on non-existent tasks
- **FR-009**: System MUST use JSON format for all request and response bodies
- **FR-010**: System MUST return appropriate HTTP status codes (200, 201, 400, 404, 500)
- **FR-011**: System MUST sanitize all user input to prevent injection attacks
- **FR-012**: List endpoint SHOULD support an optional limit parameter to control result count
- **FR-012a**: List endpoint MUST support an optional status query parameter to filter by completion status (values: "complete", "incomplete")
- **FR-013**: System MUST NOT expose stack traces or internal error details in responses
- **FR-014**: System MUST log each request including method, path, status code, and duration
- **FR-014a**: System MUST log database initialization events (creation, connection errors) at startup

### Key Entities *(include if feature involves data)*

- **Task**: Represents a work item or todo; Contains unique identifier (ID), title (required text), description (optional text), completion status (boolean), creation timestamp, and last modified timestamp
- **Task List**: A collection of all tasks returned by the list endpoint; Ordered by creation date (most recent first by default)

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Users can create a new task and receive confirmation in under 1 second
- **SC-002**: Users can retrieve a list of up to 100 tasks in under 2 seconds
- **SC-003**: Users can toggle a task between complete and incomplete status and see the update immediately
- **SC-004**: Users can delete a task and verify its removal in under 1 second
- **SC-004a**: Users can update a task's title or description and see the changes persist immediately
- **SC-005**: 100% of invalid requests return clear, actionable error messages without exposing system internals
- **SC-006**: System successfully handles at least 50 concurrent task creation requests without data loss
- **SC-007**: All task data persists across system restarts

## Assumptions

- Users interact with the API programmatically (via curl, Postman, or custom clients) — no web UI required
- Single-user assumption — no concurrent user access patterns need to be handled
- No authentication or authorization required — all endpoints are publicly accessible
- Task titles are limited to 500 characters maximum
- Task descriptions are limited to 2000 characters maximum
- SQLite database file will be stored in the application directory
- System will run on a single server/instance — no distributed deployment needed
- Data backup and recovery procedures are out of scope
- Task priorities, tags, or categorization features are out of scope for this version
- No task assignment or collaboration features needed
- System will use UTC timestamps for all date/time fields
