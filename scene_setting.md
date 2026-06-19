# roamww project lifecycle 
### 04/21


### Vision for the project
roamww is a project that I undertook for a client. The goal was to create place to house everything to do with their brand, 
roam worldwide - this includes a place to house music mixes, articles, events and a shop with a Stripe integration.

This project is at a point where the application *works* however, I have taken the decision to undertake a deep refactor
of the application with a few goals in mind. This refactor is not just about fixing bugs, it is about creating resilient 
architecture and focusing on moving from a project that works to a project that is production grade.

This involves taking stock of the current codebase and focusing on a couple of core principles: 
1) Data Integrity 
2) Frontend rigour

### Current stage
The application is created using a mixture of JS in React and Laravel with Zustand to manage global state across the application.

Upon 'completing' the MVP, I sat down and reviewed the codebase and noticed a number of issues with how the applicationw 
was currently architected. This included issues such as: 
1) Race conditions - critical to resolve in an application which deals with limited stock and involves taking money
2) Manual state management - leading to issues with state persisting across the application - fragmented user experience 
3) Lack of strict typing - inconsistencies with what the application would receive by leaning on vanilla JS

### Refactor roadmap
### Phase 1 
Phase 1 of my refactor roadmap includes towards implementing deterministic state across. Currently the application is 
'guessing'. Focusing on waiting for responses from webooks before triggering events etc, using a URL to determine change 
vs using timeouts etc. 

### Phase 2
This involves significantly strengthening the backend resilience of the application. In an application that has a shop
with limited stock and a finite amount of items, ensuring that the backend is robust enough to work and avoid race 
conditions, uses database locks to ensure database integrity and implements database locking is key for success.

### Phase 3
Currently, logic across the application is very tightly coupled. In a professional-grade headless CMS setup, this logic 
needs to be completely decoupled and logic abstracted. Moving away from the current state of play, 'fat' controllers
handling business logic, database transactions and HTTP responses in one and moving towards abstracting this logic into 
dedicated service classes to enforce a Single Responsibility Principle across the application. 

### Phase 4 
Preparing the application for production. Focusing on hosting this myself, provisioning AWS services and implementing 
CI/CD pipelines so that that project can be iterated on even after going 'live'. Software is never *done*, so I want to
ensure that I set the project up in a way which means these incremental improvements or additions can be worked on in a 
way which does not affect the production environment. 


### The why 
I wanted a place to document my journey across this refactor. In my opinion, this marks a significant shift in my 
thinking and approach to software. Being a solo dev working on this project, having a place to look back upon my decision-
making and understanding *the why* behind moving from something that just works to incrementally improving this and 
moving towards something which is instead a much more robust piece of software personally for me is a big shift.

### Success metrics 
For this refactor to be a *success* I am aiming for a few different metrics. 
1) 100% sync between local and server state 
2) Near instant response times for cart and shop actions and no 'flickering' on data re-hydration
3) 100% abstraction of logic to enforce SRP across project
