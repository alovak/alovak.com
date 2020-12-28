---
title: "Notes: Go + Microservices = Go Kit by Peter Bourgon"
date: 2020-12-28
tags: microservices, go
layout: post
---

Peter Bourgon in his talk [Go + Microservices = Go Kit](https://www.youtube.com/watch?v=NX0sHF8ZZgw&t=1075s) explains and shows how [Go Kit](https://github.com/go-kit/kit) helps with building microservices and making them production ready. Here are my notes:

Problems solved by microservices:

- team is too large and it can't work effectively on shared codebase
- teams are blocked on other teams and can't make a progress
- communication overhead

Problems caused by (with) microservices:

- unstable APIs (experiments with business domain) are going to present a whole lot of friction
- integration testing of entire microservice fleet may work for some time, but what what we should do is to optimize for MTTR (mean time to recovery) → invest into good monitoring, good deployment (bluegreen, canaries), good rollback
- DevOps → devs need to deploy and operate their work; be paged when stuff goes down, etc.

Ancillary concerns (and what's needed to be done before service is considered production ready):

- architectural choices
- authN
- reporting
- security threat model
- license audit
- compliance audit
- dependencies: services, 3rd party services and libraries
- monitoring plan
- maintenance process
- backup and restore process
- secret management
- secret rotation
- on-call schedule
- capacity plan
- configuration management
- alerts
- log aggregation
- distributed tracing
- Ops and incident response runbooks
- API documentation
- CI pipeline (build, test, publish, integration test, contract test)
- Canary, Deploy, Post deploy test

<img src="/images/microservice-infrastructure.png" alt="microservice infrastructure" style="max-width: 710px;"/>

Onion model → wrap in a middleware, business logic is in the center

<img src="/images/microservice-onion-architecture.png" alt="onion model" style="max-width: 710px;"/>
