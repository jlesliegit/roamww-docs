# Abstraction of testing logic

## 30/04/2026
## Status: Accepted
## Context

Original test suite relied on heavy end-to-end feature/HTTP tests. This had wide test coverage and ensured that the 
application was fully testable as developed however made debugging difficult. It was not clear when a test failed if this 
was because of the API contract, the HTTP request or business logic failing as it was all handled in the same test. 

## Decision
Adopt a formal testing strategy as a part of the application refactor. 

**Feature Tests** - reserved for route testing, middleware, validation, HTTP responses and API Structure + JSON responses
**Unit Tests** - reserved for testing singular service class methods, database transactions + file handling

## Consequences
### Positives
- Clarity on test failures - understanding exactly where the failure is to improve debugging efficiency 
- Resilient test suite - changes to JSON structure or HTTP routes no longer turn whole test suite red
- Clearer API Contracts by explicitly testing data shape sent to front end

### Trade-offs
- More boilerplate code written upfront to split 'happy path' across two test layers
- Strict mapping of responsibilities to avoid overlapping assertions