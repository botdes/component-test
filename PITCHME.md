@title[Introduction]

# Component test in ATE

---
@title[Plan]
## Plan

* What is ATE
* Component in testing pyramid
* Evaluation of component test in ATE
* Best practices
---

@title[What is ATE]

## What is ATE?
#### 15 backend engeneers
#### 63 gihub repos

---

## Component test in testing pyramid

Note:
Add a picture here:
Introduced fairly recently

---
## What requirements for component test
* Close to production
* Blackbox 
* Fast
* Running locally

@[1]
@[2-4]

Note: 

---


## Initial state
### Only Unit, Integration and Smoke Tests
* Test manually via datadog in dev and pre environments
* Issues in juice in dev
* Long feedback loop 
* Real AWS causing a lot of garbage

---
##
1) what is ATE
    - backed, 20 + micro services (http/sqs/kinesis)
2) component in testing piramind
    - unit/ integration / component / smoke test 
3) what requirements for component test
    - close to production
    - blackbox 
    - fast
    - running locally
4) Initial state: No component test:
    - test manually via datadog in dev / pre
    - failures in dev with juice 
    - very long feedback loop
5) Intermediate state: integration test using real AWS
    - slow
    - shared resources between tests
    - not very stable
    - leave lots of garbage in aws
    - issues with local run / security 
6) Component test using docker
    - fast
    - close to prod
    - nearly blackbox
    - not 100% similarity between environments (if isLocal)
    - mocked data in services, can be wrong assumptions
7) Component test best practices
    - use as blackbox, don't' check the db
    - don't mock "inconvenient" classes
    - test one thing 
    - check not only one happy path, but business related feature.
