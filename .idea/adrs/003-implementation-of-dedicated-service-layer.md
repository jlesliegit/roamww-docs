# Implementation of dedicated service layer across application 
## 21/04/2026 
## Status: proposed
## Context 

Currently, the application features 'fat' controllers. No abstraction of business logic or database transactions into 
dedicated service classes meaning that controllers are handling all of this alongside http responses. This is strictly 
violating the Single Responsibility Principle and also means that currently, the tests for the application have wide 
coverage however these are essentially just testing the http responses. If there is an issue with any of the logic, this
is currently very difficult to find the fault whilst all logic sits within a few fat controller methods.

## Decision 
Abstract business logic and database transactions from these controllers into dedicated service classes. Separate the 
concerns of each and implement logic-based testing following this to better enforce single responsibilities into methods
and better pinpoint failures across test cases as the application grows.

Abstract logic into single-action services to avoid a 'God Service' by simply moving all logic from controller to a 
Service


## Consequences
### Positives
Single responsibility principle enforced across application

Test failures easier to diagnose quicker 

Abstraction of logic to dedicated service layer significantly clears up codebase 

### Negatives
Extra layer of abstraction that will be more difficult to follow 

Integration of these services together for controllers to still function as they are currently 

*Risk* - implementing a 'God Service' - simply moving all logic to one service as opposed to properly abstracting logic 
to mulitple, single-action services based on action