# Remediation Patterns for Technical Debt

This reference provides common remediation strategies for high-criticality violations found in CAST Imaging.

## Efficiency (Performance)

### Heavy SQL in Loop
- **Issue**: SQL queries executed inside a loop.
- **Fix**: Refactor to use bulk operations or subqueries to reduce database round-trips.

### Large Memory Allocation
- **Issue**: Objects or collections that grow without bounds.
- **Fix**: Implement pagination, use streaming, or apply appropriate cache eviction policies.

## Security

### SQL Injection
- **Issue**: Concatenating user input into SQL strings.
- **Fix**: Use parameterized queries (PreparedStatements).

### Improper Error Handling
- **Issue**: Leaking system information in error messages.
- **Fix**: Use generic error messages for users and log detailed errors internally.

## Robustness (Reliability)

### Unclosed Resources
- **Issue**: Database connections, file streams, or sockets not explicitly closed.
- **Fix**: Use try-with-resources (Java) or ensure `close()` is called in `finally` blocks.

### Unhandled Exceptions
- **Issue**: Critical methods that swallow exceptions or don't catch specific failures.
- **Fix**: Implement specific catch blocks and proper recovery or logging logic.

## Maintainability

### High Cyclomatic Complexity
- **Issue**: Methods with too many branches/conditions.
- **Fix**: Extract sub-methods or use Strategy/State patterns to simplify logic.

### Duplicate Code
- **Issue**: Similar logic repeated across multiple files.
- **Fix**: Abstract common logic into a utility class or base class.
