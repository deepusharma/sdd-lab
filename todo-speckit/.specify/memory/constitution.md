<!--
Sync Impact Report
==================
Version: 1.0.0 (Initial ratification)
Date: 2026-08-02

Changes:
- Added Principle 1: Stack
- Added Principle 2: API Conventions
- Added Principle 3: Testing
- Added Principle 4: Error Handling
- Added Principle 5: Security
- Added Principle 6: Performance
- Added Principle 7: Observability
- Defined governance rules for learning/experimental approach
- Removed optional sections (security and performance integrated as principles)

Follow-up TODOs: None
-->

# Todo API — Speckit Build Constitution

## Core Principles

### I. Stack

Python 3.11+ required; FastAPI framework for REST API implementation; SQLite for data storage with no external service dependencies. This is a single-user learning exercise with no authentication required.

**Rationale**: Minimizes setup complexity and external dependencies, enabling focus on core API development patterns and testing practices.

### II. API Conventions

RESTful routes following standard conventions: POST/GET/PATCH/DELETE on `/tasks` endpoint. All requests and responses MUST use JSON format. Standard HTTP status codes MUST be used appropriately (200/201/400/404/etc.).

**Rationale**: Consistent API design improves predictability and follows industry standards, making the codebase easier to understand and extend.

### III. Testing

Every endpoint MUST have at least one automated test (pytest) before being considered done. Tests MUST cover the happy path and at least one error case per endpoint.

**Rationale**: Testing is the primary learning objective. Pre-implementation testing ensures functionality is verified before marking work complete.

### IV. Error Handling

Invalid input MUST return 4xx status codes with clear JSON error messages. Stack traces and internal implementation details MUST NOT be exposed to clients.

**Rationale**: Proper error handling improves API usability and prevents information leakage that could aid attacks or confuse users.

### V. Security

All input MUST be validated and sanitized. Database queries MUST use parameterized statements only — no string-concatenated SQL. No secrets or credentials may be committed to code.

**Rationale**: Even in a learning exercise, building secure coding habits prevents common vulnerabilities (SQL injection, credential leaks) from becoming ingrained practices.

### VI. Performance

No premature optimization. Avoid obviously quadratic algorithms. List endpoint SHOULD support an optional `limit` parameter.

**Rationale**: Balances learning about performance considerations without over-engineering a single-user application.

### VII. Observability

Every request MUST be logged to stdout including: HTTP method, path, status code, and request duration.

**Rationale**: Logging provides visibility into API behavior and aids debugging during development.

## Governance

This constitution defines the non-negotiable standards for the Todo API learning project. Given the learning/experimental context:

- Principles marked "MUST" are strict requirements that block completion
- Principles marked "SHOULD" are strong recommendations that may be deferred with justification
- All pull requests and code reviews SHOULD verify compliance with constitutional principles
- Amendments require updating this document with a version bump and rationale

**Amendment Process**:
- PATCH: Clarifications, typo fixes, non-semantic wording improvements
- MINOR: New principle added or existing principle materially expanded
- MAJOR: Principle removal, redefinition, or backward-incompatible governance change

**Version**: 1.0.0 | **Ratified**: 2026-08-02 | **Last Amended**: 2026-08-02
