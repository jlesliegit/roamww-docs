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
