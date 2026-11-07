# Research: Task CRUD API

**Date**: 2026-08-02
**Status**: Complete

## Overview

This document consolidates technical research and decisions for the Task CRUD API implementation. All technical context items from the plan were pre-specified in the constitution, so this phase focuses on best practices and implementation patterns rather than technology selection.

## Technology Decisions

### Decision 1: FastAPI Framework

**Rationale**: FastAPI provides automatic API documentation (OpenAPI/Swagger), built-in request validation via Pydantic, async support for performance, and type hints for better code quality. It's the constitutional requirement and well-suited for building REST APIs quickly.

**Alternatives Considered**:
- Flask: More lightweight but requires manual validation and documentation setup
- Django REST Framework: More features but heavier weight, overkill for a learning exercise

**Best Practices**:
- Use dependency injection for database connections
- Leverage Pydantic models for request/response validation
- Enable automatic API docs at `/docs` endpoint
- Use APIRouter to organize endpoints by resource

### Decision 2: SQLite Database with Auto-Initialization

**Rationale**: SQLite is zero-configuration, file-based, and sufficient for single-user scenarios. Auto-initialization (FR-006a) ensures developers can start immediately without manual database setup.

**Alternatives Considered**:
- PostgreSQL: Overkill for single-user, requires separate server process
- In-memory storage: Data wouldn't persist across restarts (violates SC-007)

**Best Practices**:
- Use context managers for connection handling
- Implement migrations via simple SQL schema file
- Enable WAL mode for better concurrency
- Use parameterized queries exclusively (constitutional requirement)
- Log initialization events (FR-014a)

### Decision 3: Pydantic for Data Validation

**Rationale**: Pydantic integrates seamlessly with FastAPI and provides automatic validation of title/description length limits (500/2000 chars), required field enforcement, and JSON serialization.

**Alternatives Considered**:
- Manual validation: Error-prone, repetitive code
- Marshmallow: Requires more boilerplate than Pydantic

**Best Practices**:
- Define separate schemas for request (create/update) and response
- Use Field() validators for length constraints
- Leverage Pydantic's built-in error formatting for clear 400 responses

### Decision 4: pytest with TestClient

**Rationale**: pytest is the constitutional requirement. FastAPI provides TestClient for integration testing without running a real server.

**Alternatives Considered**:
- unittest: More verbose, less powerful fixtures
- Manual curl testing: Not automated, violates testing principle

**Best Practices**:
- Use pytest fixtures for test database and client setup
- Separate unit tests (models) from integration tests (endpoints)
- Test both happy path and error cases per endpoint (constitutional requirement)
- Use in-memory SQLite (`:memory:`) for faster test execution

## Implementation Patterns

### Pattern 1: Request Logging Middleware

**Decision**: Implement logging as FastAPI middleware to capture method, path, status, and duration for all requests (FR-014).

**Approach**:
- Use `@app.middleware("http")` decorator
- Calculate duration using `time.time()` before/after request
- Log to stdout using Python's `logging` module
- Format: `{method} {path} {status_code} {duration_ms}ms`

### Pattern 2: Error Response Format

**Decision**: Standardize JSON error responses to meet constitutional error handling requirements.

**Format**:
```json
{
  "error": "Brief error type (e.g., 'Validation Error')",
  "detail": "Specific, actionable message (e.g., 'Title exceeds 500 character limit')"
}
```

**Approach**:
- Use FastAPI's HTTPException for standard errors
- Implement custom exception handler for validation errors
- Never expose stack traces (FR-013)

### Pattern 3: Database Schema Evolution

**Decision**: Use a single `schema.sql` file for initial setup. For this learning exercise, manual schema changes are acceptable (no migration framework needed).

**Rationale**: Keeps complexity low per constitution. Single-user setup means no production migration concerns.

## Performance Considerations

### Concurrency Strategy

**Decision**: FastAPI's async support is sufficient for 50 concurrent requests (SC-006). SQLite's default locking handles concurrency for single-user scenario.

**No Action Needed**:
- FastAPI async handlers provide concurrency
- SQLite handles row-level locking automatically
- No connection pooling required (single-user)

### Query Optimization

**Decision**: No indexes beyond primary key (ID) initially. Add only if performance tests show need.

**Rationale**: Constitutional principle VI prohibits premature optimization. With expected ~100 tasks maximum, sequential scans are fast enough.

**Future Optimization** (only if needed):
- Index on `completed` field if filtering becomes slow
- Index on `created_at` if sorting becomes slow

## Security Patterns

### Input Sanitization

**Decision**: Rely on Pydantic's string validation + parameterized queries. No additional sanitization needed.

**Rationale**:
- Pydantic validates data types and lengths
- Parameterized queries prevent SQL injection (constitutional requirement)
- JSON parsing handles escaping automatically
- No HTML/XSS risk (API-only, no web UI)

## Open Questions

None - all technical context items were pre-specified in the constitution.
