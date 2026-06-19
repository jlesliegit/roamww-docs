# Context
## 20/05/2026
## Status: Accepted
Initially, response shapes were dictated completely by models and ORM. Eager loading relationships were never declared.
The initial approach was to *blacklist* columns I did not want to receive instead of *whitelisting* those which I wanted 
to include in the response shape. 

In turn, the frontend types were written as mirrors of these implicit shapes but no contract was enforced between.

This was flagged when refactoring the orders domain. On the frontend, types declared that orders have products, the 
backend correctly sent them, the frontend accepted them and these products were loaded however rendered nowhere on the
application. This was simply wasting compute by sending these when not needed. 

This initially slipped under the radar as the application presented no errors, the UI did not render this information, 
the data was not incorrect just unused. 

## Decision 
Adopt API resources as standard for all domain responses to enforce one singular declared data contract per domain slice.

Application wide convention, not just on the Orders domain where it proved to be a problem. 

Changes behaviour from 'fixing a type' to forcing end-to-end trace of data that is sent vs data that is consumed.

## Consequences 

### Positives
- Catching this on Orders domain flagged a missing section to render OrderItem specific data. Added after enforcing contract
through API resources
- Response shapes become explicit, enforcing a single source of truth
- Consistency across application domains 

### Trade offs 
- Orders is the only domain where the payoff is directly demonstrated
- Contract is explicit however not self-enforcing. Enforced at write-time by discipline not tooling
- Time cost defining these across domains and mirroring on frontend to enforce integrity of data sent vs consumed
- Frontend types also remain mirrors of these resources and can drift over time. Will be the case unless generated types 
introduced
- Resource envelope ('data' wrapper) means that this data needs to be unwrapped properly across application 
`response.data.data` to properly access - needed addressing in backend + frontend response unwraps to access data
- To verify - Other domains may be shipping unused fields just not caught yet 