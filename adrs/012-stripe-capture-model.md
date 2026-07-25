# Stripe Capture Model

## 22/07/2026
## Status: Accepted
## Context
Initial development of checkout flow meant that `createCheckoutSession` never set Stripe `capture_method`. This meant 
Stripe automatically applied its default capture model which is auto capture on checkout completion. This was discovered
after running a real test-mode checkout flow and checking `PaymentIntent` in the Stripe dashboard log. 
`ProcessCheckoutSession` then called `capture()` on an already succeeded payment intent with no try/catch block which 
threw an uncaught 500 error on every successful checkout. This throw precedes the `Mail::queue` block so the confirmed 
order email never actually sends. The cancel path called `cancel()` on an already-succeeded `PaymentIntent` and also threw.
Also, no refund path existed in the application. These discrepancies with the checkout were found in a pre-go-live audit 
on July 22.

This discovery led to a decision about how this payment should be captured. Two proposed options:
1) Authorise at checkout, capture later (manual)
2) Capture immediately and refund if needed (auto)

Two main points informed the choice:
1) Oversell probability is very low at launch. Controlled stock, low concurrency, single operator store. 
2) The site is under my active ownership both pre and post-launch and not handed off. 

## Decision
Because of the two points listed above, low oversell probability and the fact that once project is live I will be 
continuing maintenance of the application, the option that was decided to implement is option 2, auto capture and refund
if necessary.

Option 1 was rejected on the basis that it was deemed as over-engineering given the proposed scale of the shop. The shop 
will be going live with 1 item to begin with and ample stock given the size of the brand. As I will be the sole 
maintainer for this repo as well, this also means that if the scale of the project ever grows to the point where 
decision 1 makes sense, I am positioned to implement this change. For context, the point at which this upgrade would 
make sense would be when charge-then-refund events occur in more than 5% of orders in a one-month rolling period. 
This will be the trigger for the decision to implement decision 1 and move to manual capture.

## Consequences 

### Positives 
- Simplest correct implementation at current scale as it removes need to manage Stripe auth lifecycle.
- Manual capture's 7-day auth window is a failure mode that does not exist on auto capture. 
- No partial capture logic. Multi-item orders with a partial oversell don't need additional branching logic in webhook.
- Less state in pending window keeping `CheckoutSuccessPage` flow simpler to reason about and stabilise. 
- Fixes actual crash by removing the `capture()` on succeeded call plus wrapping in try/catch block stops the uncaught 
500 error allowing `OrderConfirmed` mailer to send. 

### Trade-offs
- In the case of oversell, the customer is charged initially and then refunded vs never charged. This is the one major
downside of this approach - it is a worse UX experience for the customer. This decision was accepted on the basis that 
this is a fair trade-off given how improbable this scenario actually is. Not because it is the perfect solution. 
- Charge then refund can incur small processing costs and creates a refund to reconcile where manual capture would 
automatically release an uncaptured auth at no cost. 
- Future migration to manual capture is more expensive than building now. Deferring the cost of this on the assumption 
that the site will not grow to the scale where this is a necessary change to make. However, this does not change the 
decision that implementing now would be over-engineering against a situation which is highly improbable. 
- Requires implementing refund paths which do not currently exist.  

