# Deterministic Cart Clearing 

## 20/04/2026
## Status: Accepted
## Context 
Application uses asynchronous Stripe webhook to handle order completion. I noticed that upon succesful checkout, Stripe 
redirects, the user is redirected straight back to the `/success` page, sometimes before webhook had finished processing 
meaning that the `cart_items` table was not cleared for associated cart session. 

This resulted in the application re-hydrating stale data from the database and showing previous order's items despite 
successful payment and webhook firing correct signal back. 

## Decision
Implemented a _deterministic guard_ rather than initial thought which was to simply use a `setTimeout`. 

1. The `useCartStore` checks for `sessionId` parameter in URL on mount
2. If parameter is present (indicating a successful Stripe session) store bypasses standard `fetchCart` logic
3. Store executes `clearCartLocal` to reset state immediately 
4. Webhook remains source of truth, clearing database once `checkout.session.completed` event is verified

## Results

### Positives
Instant feedback

Solution does not rely on slow webhook processing times

No websockets needed to 'check' if webhook is completed

### Trade-offs
Relies on presence of `stripe_session_id` is query string

Must understand that the URL state takes precedence over standard hydration flow during checkout