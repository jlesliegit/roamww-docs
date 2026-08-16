# roamww-docs
> Architectural Decision Records (ADRs) and refactoring documentation + reflection for the roamww platform

## Project Overview
**roamww** is a full-stack music and lifestyle platform built for an independent record label. 

The project features a headless CMS for content management and a fully integrated Stripe e-commerce storefront.

This repo is a place for me to document the project's architecture, focusing on the evolution of the platform from initial MVP phase through a major refactor ahead of go-live. 

## Tech Stack
* **Backend:** Laravel (PHP), MySQL
* **Frontend:** React, JavaScript migrated to TypeScript
* **Payments:** Stripe API

## Refactor Roadmap
To aid with robustness, the application is currently being refactored. This documentation is tracking the progress against the six key pillars of this refactor:
1. **Architecture:** Abstracting logic into Service Classes and implementing Form Requests across the application
2. **Frontend Rigour:** TypeScript Migration and implementing API Resources
3. **Data Integrity:** Schema normalisation and Dynamic Eager Loading
4. **Resilience:** Pessimistic Database locking for Inventory management (*complete*)
5. **Infrastructure:** AWS Provisioning
6. **DevOps:** CI/CD Pipelines and zero-downtime deployment

## Architectural Decision Records
Each ADR details the context, the decision choices and their consequences. These decisions are across the project's
life from MVP to go-live. 

Earlier ADRs are not as substantial - they are intentionally preserved as written to show development and progression in 
how I approached these decisions. 

- [ADR-001: Deterministic cart clearing](adrs/001-deterministic-cart-clearing.md)
  - Status: **Accepted**
- [ADR-002: Global state for SoundCloud media player](adrs/002-global-state-for-soundcloud-media-player.md)
  - Status: **Accepted**
- [ADR-003: Implementation of dedicated service layer](adrs/003-implementation-of-dedicated-service-layer.md)
  - Status: **Accepted**
- [ADR-004: Database normalisation](adrs/004-database-normalisation.md)
  - Status: **Proposed**
- [ADR-005: Abstraction of testing logic](adrs/005-abstraction-of-testing-logic.md)
  - Status: **Accepted**
- [ADR-006: Unify database testing state](adrs/006-unify-database-testing-state.md)
  - Status: **Accepted**
- [ADR-007: TypeScript application migration](adrs/007-typescript-application-migration.md)
  - Status: **Accepted**
- [ADR-008: Adopt API resources for all domain responses](adrs/008-adopt-api-resources-for-all-domain-responses.md)
  - Status: **Accepted**
- [ADR-009: Service layer method alignment](adrs/009-service-layer-method-alignment.md)
  - Status: **Accepted**
- [ADR-010: Stripe consistency design](adrs/010-stripe-consistency-design.md)
  - Status: **Accepted**
- [ADR-011: Stripe integration testability](adrs/011-stripe-integration-testability.md)
  - Status: **Accepted**
- [ADR-012: Stripe Capture Model](adrs/012-stripe-capture-model.md)
  - Status: **Accepted**
- [ADR-013: Backend authentication model](adrs/013-backend-authentication-model.md)
  - Status: **Accepted**