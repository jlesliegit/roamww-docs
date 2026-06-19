# Context
## 29/05/2026 - 31/05/2026
## Status: complete
Whilst abstracting fat controller logic into service layer, these services were abstracted at different times leading to
misalignment and inconsistencies including: file handling was inconsistent across methods in different domains; file naming 
was inconsistent across domains; DB transaction patterns not consistent or, in the case of two delete methods, absent 
entirely; two and in some cases three DB writes in the same transactions in some methods. 

An audit was completed to define conventions and apply them across the methods. Inconsistencies that produce no 
behavioural difference were deliberately left in place. These are documented below.

## Decision
Service layer changes scoped to ensure correctness across domains and contract consistency. Accept some stylistic or 
idiomatic divergence between domains where it produces no difference in result or functionality of codebase. 
- Files stored as basename paths, deleted always using `'assets/'` prefix. Mixed formats cause silently-failing deletes. 
Basename-only storage addressed throughout application prevents this.
- All mutations wrapped in `DB::transaction`. Non-transactional side effects (e.g. deletion or external API calls), 
occur outside of transaction to aid data integrity. 
- Single DB write by using `update($payload)` where possible vs multiple writes in a transaction.
- Route-model binding vs slug lookup, use of `Arr::only` filtering, `load` vs `loadMissing` all left as is per service. 
No behavioural change in choice A vs choice B in these decisions. Inconsistencies to be left as-is unless clear 
behavioural justification emerges in future development.

## Consequences

### Positives
- Two data integrity bugs fixed. Non-transactional deletes in Article and Mix domain. System can no longer orphan rows 
with missing files due to partial failure.
- File-path contract is now consistent across all services reducing possibility of orphaned files over time
- Update methods now perform a single DB write where possible and removes ambiguous intermediate row state during DB 
transactions.
- Test coverage extended to ProductServiceTests which had no service level tests previously.
- Documented technical debt uncovered in this audit - Stripe calls in-transaction, data migration, orphaned Stripe 
products on delete. These issues have always existed but were invisible before this audit. Now written trail of 
outstanding bugs to fix before go-live.

### Trade offs
- Codebase now has deliberate inconsistencies between services. Currently, this ADR log is the only proof that this is 
intentional. 
- File-path code change was not paired with a migration. Frontend image rendering for these rows *will* break until 
frontend changes are addressed to handle. Decision made to not run migration as DB currently just contains mock, fake
data which is unimportant and frontend TypeScript migration (adr-007) is still not 100% complete. 
- `DB::transaction` is applied consistently across mutations methods even those which only include one single-write 
updates where the wrapper actually provides no additional safety. This is enforced regardless for code-consistency
across methods which are more important than route-model binding vs slug lookup.
- The Stripe-in-transaction issue is now known and named but not fixed. ProductService can still produce divergent state
between local DB and Stripe on partial failure. The audit raised the visibility of this bug without reducing its impact.
- Test coverage for ProductService is now included however is currently minimal - just three tests. Comprehensive testing 
still needs implementing.
