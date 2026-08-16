# Backend Authentication Model

## 14/08/2026
## Status: Accepted
## Context
Initial development used Sanctum personal access tokens (bearer tokens): `AuthController::login` calls `createToken` and
returns token in the JSON body. All admin write and other protected routes authenticate via `auth::sanctum`, resolving 
the bearer token. This process is already working end-to-end for application. 

Pre-go-live audit conducted on Aug 14th revealed that codebase also contains partial, non-functional cookie/SPA (stateful)
auth scaffolding: `SACTUM_STATEFUL_DOMAINS` set, `config/sanctum.php guard => ['web']`, CORS `supports_credentials: true`,
and auto-registered `/sanctum/csrf-cookie` route. However, `EnsureFrontendRequestsAreStateful`, the middleware which 
actually activates cookie auth was absent meaning the path does not function. Scaffolding was put in place whilst decision
was being made, and not finished once bearer token approach was decided upon. 

This created a decision about which auth model to commit to for go-live. 
1) Migrate to cookie/session (Sanctum SPA) based auth utilising HttpOnly cookies with token not reachable from JS. 
2) Retain bearer tokens (tokens stored client side and sent as `Authorization: Bearer`)

Points informing the decision:
1) Whether frontend and API will be same-site or cross-site is not yet decided. Bearer is decision-agnostic, cookie auth
depends on the answer.
2) The cookie path is not near-complete, missing load-bearing middleware so option 1 is build-and-validate option not a 
finish the last mile one. 
3) The security case for cookies is token theft via XSS injection. In the application, the XSS bearing fields are all 
admin-only and there is only a sole admin for the site. 

## Decision
Retain bearer tokens for go-live. 

Option 1 was rejected as premature given the indecision around frontend/API same-site/cross-site decision. Keeping bearer
tokens keeps decision open whereas committing to cookies forces cross-site cookie config that can't be finalised until 
decision is set. Further rejected on timing, migrating to cookie-based auth would mean building missing middleware, 
rebuilding CORS and validating cross-origin cookie behaviour. 

As a sole maintainer, I am positioned to migrate later *if* the conditions change. The trigger for revisiting:
Production deployment decision resolves to same-site AND untrusted (non-admin) user-submitted content enters app. At this
point, cookies become both low-cost and more valuable, meaning the migration is warranted. 

## Consequences
### Positives
- Production decision-agnostic. No dependancy on undecided same-site/cross-site question.
- Immune to CSRF by construction - token not auto-attached by browser so no CSRF-token flow to maintian. 
- Uses model already wired and working, no new load-bearing middleware introduced just before launch. 

### Trade-offs
- Bearer token store client-side is reachable by injected JS so can be exposed with an XSS bug - the exposure cookies 
would remove. Accepted on the basis that the XSS-bearing fields are admin-only and there is only a single admin user,
not because bearer is the more secure model. This makes server-side sanitisation of fields the required mitigation
(this is tracked separately. Needed regardless of auth model, not a consequence of this decision, a pre-planned decision).
- Deferring migration is more expensive than building cookies now. Accepted because the conditions that would require 
cookies don't hold at go-live and have been pre-set. 
- Leaved abandoned SPA scaffolding in repo that must be removed so it can't half-fire in prod - cleanup work created by 
this decision.


