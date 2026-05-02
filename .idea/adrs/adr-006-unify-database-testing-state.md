# Context
The initial test suite utilized a mix of both `DatabaseMigrations` and `RefreshDatabase` traits across different test 
classes. This caused a deceptive bug: when test suites were run individually in isolation, they passed. However, when 
running the full test suite via `php artisan test`, several tests would fail with a `no such table` SQL error.

Because `DatabaseMigrations` physically drops and rebuilds database tables after every test, it was destroying the 
database schema mid-run. Subsequent tests utilizing `RefreshDatabase` (which assumes the schema is built once and 
relies on fast database transactions) were attempting to query tables that had been deleted by the legacy trait.

## Decision 
Standardise testing architecture to exclusively use `RefreshDatabase`.

Modern Laravel standard to reset state between tests vs destructive schema migrations

## Consequences 

### Positives
- Suite stability: Resolved `no such table` errors when running full test suite
- Massive performance gain: By removing behaviour of rebuilding schema between tests, full test execution time dropped from
~>60 secs to ~<20 secs
- Developer experience: 3x faster feedback loop significantly improves local development speed

### Trade offs
- No trade offs or negative consequences. Using modern Laravel approach is strict upgrade on architecture