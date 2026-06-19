# Context
## 03/05/2026
## Status: Accepted
Application was initially written in plain vanilla JS. Although initial state was that application worked, after 
reviewing codebase, the decision was made that as the backend was refactored, this presented a perfect opportunity to 
introduce TypeScript to enforce strict typing and adeherance to API contracts. 

The primary objective is to enforce strict typing declarations and standardise API boundaries across the frontend, 
transforming unpredictable API payloads into strict, predictable contracts before they reach the component layer.

## Decision
Migrate frontend React architecture to TypeScript to enforce strict interface declarations for component props, state 
hooks and API responses

## Consequences

### Positives
- Improved code predictability
- Ability to catch errors at compile time and not in runtime 

### Trade offs
- Required a lot of manual refactor time both planning and implementing these types. Understanding exactly which data
each component needed and how to best hydrate these