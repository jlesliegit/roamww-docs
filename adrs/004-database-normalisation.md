# Database schema normalisation
## 28/04/2026
## Status: proposed
## Context
When development began, the codebase was approached one feature at a time. This has lead to a number of inconsistencies 
with how data is referred to across the application, duplicate columns for items which can share behaviour and incorrectly
defined relationships meaning id_columns where a pivot table should be used.
e.g. the `mixes` table contained a `resident_id` column whilst there was an already existing `mix_resident` table as the
relationship was not defined during initial development. 

## Decision
Strict definition of many-to-many relationships, utilisation of pivot_tables and using Laravel's `sync()` method to ensure
a single source of truth for relationship mapping. 

Intentionally keeping legacy columns whilst in development to be dropped later on so I can gradually refactor this, 
focusing on backend first to ensure data integrity. Keeping these legacy columns and providing temporary workarounds 
reduces context switching from backend -> frontend constantly and keeps focus on the larger architectural refactor decisions. 

## Consequences
### Positives
- Standardisation of data across application
- Consolidation of shared objects into a single place

### Trade-offs
- Shift in API Contracts e.g. moving from returning a single resident to an array of residents and processing this 
appropriately across backend + frontend 
- Testing env constraints - foreign key constraints with SQLite Doctrine DBAL needing a temporary workaround until legacy
columns completely dropped 
- Now a necessity to think about relations and utilise eager loading to adhere to new API contracts 