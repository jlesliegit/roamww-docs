# Context
## 14/06/2026 - 16/06/2026
## Status: completed [16/06/2026]
During adr-010 work, it was discovered that the `updateStripeProductMetadata` method calls Stripe SDK static methods: 
`Price::all`, `Price::update` & `Product::update` directly. This means that the method can't be mocked and forces any 
tests to need to make real network calls. As a result, the price-archive path was never tested.

This left a bug in `currentActivePriceId` undetected. It returns `$sortedPrices->first->id`, which on a Laravel 
Collection is a higher-order proxy, not property access. Rather than returning the price ID string the `?string` 
signature promises, it silently returns a whole Price object. The object is truthy, so it passes the `if ($oldPriceId)` 
guard and gets passed to `Price::update` as the ID - meaning the old price is never archived and orphaned active prices 
accumulate on every price change.

This is the concrete cause of the multiple active prices and resulting false positive referenced as a risk in adr-010. 
The issue was initially missed because the failure was concealed at three points: the proxy returned silently, the 
downstream `Price::update` error was swallowed by the try/catch and only logged, and the method still returned a valid Product.

## Decision
Proposed solution is decouple Stripe SDK from service via dependency injection. This allows the calls to be mockable, 
allowing for testing and ensuring that the method behaves as expected. 

Rejected alternative solution: testing the Stripe calls by making real API calls. This was rejected for a number of reasons:
Speed, necessity for live credentials in testing, inability to run offline (this will be an issue once planned CI operations 
implemented in project). 

Decoupling the services will make the StripeService methods mockable and testable, starting with 
`updateStripeProductMetadata` and addressing the others later.

The actual fix for the named bug, moving from `first->id` to `first()->id` in the `currentActivePriceId` method resolves 
the silent type error. 

This also raised that the current swallow-and-log behaviour may need reframing for the future - this error was silently 
failing because of this approach. Not addressing directly in this fix but earmarked for later observation. 

### Revised
Dependency injection was rejected - not an appropriate solution for the issue as issue was initially misdiagnosed. 
Real issue was in the logic, not in the Stripe interaction. The dependency injection would not have tested the behaviour
that was broken within the codebase. 

Because of this, active price selection logic was instead extracted to a public, testable method. API calls + injection 
now rejected alternatives. 

## Consequences

### Positives
- Price selection path becomes unit-testable without need for API calls.
- Resolves orphaned active prices failure at source.

### Trade-offs
- Swallow and log behaviour still in place. Currently deferred.
- Despite being an internal helper, selection method is public to allow direct testing. This is an accepted trade off for
the project.
- Fetch and mapping for price selection untested. Would require mocking SDK interaction which was judged as 
over-engineering for codebase.
- Price ties intentionally untested. This is accepted as low-risk as the application has a single admin user and cases 
where this may be needed, two price updates at same second, are slim to zero chance given this context. 
