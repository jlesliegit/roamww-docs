# Context
## 02/06/2026 - 04/06/2026
## Status: complete
During adr-009-service-layer-method-alignment audit, it was discovered that the current stripe implementation risked
state divergence between local DB and Stripe in the case of partial failure. This issue touches `storeProduct`, 
`updateProduct` and `deleteProduct`.

## Decision 
The proposed solution is eventual consistency with reconciliation. This was chosen out of three proposed solutions, the 
other two that it were chosen over were: 
- Outbox pattern - using single DB transaction to both update application state and save outgoing events to temp outbox 
before background process publishes. This solution was deemed overkill given the scope of the project - product writes
are currently infrequent enough that complexity of maintaining outbox, background worker + retry logic not justified.
- Saga/compensation - breaking distributed transactions into sequence of local transactions to commit. Deemed too brittle
and did not align in principle to other engineering processes implemented across project which favour robust solutions. 

This led to the third choice - eventual consistency with reconciliation. It must be noted, the solution is not deemed 
complete or shippable until reconciliation is properly implemented. Failure to comply with this will ignore potential
future divergences. 

Implementation principles:
- DB-first ordering for `updateProduct` + `deleteProduct`. Transaction commit first and Stripe call after. Stripe-first
ordering for `storeProduct`. Stripe call first and transaction commit after. Asymmetry is intentional and is due to 
structural differences between creating a Stripe-DB relationship vs modifying an existing one. 
- Idempotency keys on Stripe-mutating calls. Makes retries safe - essential as part of fix, not optional. 
- Daily reconciliation run chosen due to low product update frequency. 24-hour divergence window acceptable at current scale.
- Investigate -> correct manual workflow chosen over auto-correction. Chosen as auto-correcting risks reversing real
manual Stripe dashboard edits causing further divergence. Correction action depends on cause, cannot be determined 
automatically.
- Email alerts to own address, daily report of findings - even if no divergences. Absence of email indicates script isn't 
running, acting as own safety net. 


## Consequences

### Positives
- State divergence is no longer silent. 
- Increased visibility into Stripe state including detection of orphaned products from causes other than divergence e.g.
manual Stripe dashboard creations
- Foundation for future scaling. When application evolves to need outbox pattern, this can be added without rearchitecting.
- Idempotency keys added which make Stripe retries safe across application - not just these affected methods. 

### Trade-offs
- Decision made to run reconciliation script every 24 hours due to scale of project. This means up to 24-hour silent
divergence between these runs. May need to be addressed if scale increases. 
- Reconciliation script itself becomes critical infrastructure which needs managing. Plan is to send a daily info email 
to have visibility on this on a daily basis.
- Workflow: investigate -> correct requires attention per alert - not scalable practice.
- Initial reconciliation script will likely have an excess of historical orphans needing cleanup.
- As Stripe prices are immutable, price changes create a new Price object and archive the old, reconciliation compares 
local `price_pennies` vs active Stripe price. Edge cases such as multiple active prices via manual dashboard creation
or in-flight price transitions during reconciliation, may produce false positives. Must verify behaviour during development.
- Implementing idempotency keys. Without these, retries become unsafe and reconciliation cannot distinguish duplicates
from distinct products.
