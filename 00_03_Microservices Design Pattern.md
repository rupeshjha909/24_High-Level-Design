# Microservices Design Patterns: A Ground-Up Guide

> A practical reference on microservices architecture and the patterns that make it work — decomposition, Saga, Strangler, CQRS, and the operational patterns (API Gateway, Service Discovery, Circuit Breaker, and more) — built from first principles with diagrams, trade-offs, and interview prep.

---

## Table of Contents

1. [What Are Microservices, and Why?](#1-what-are-microservices-and-why)
2. [The Pros and Cons (and the Honest Trade-off)](#2-the-pros-and-cons-and-the-honest-trade-off)
3. [Decomposition Patterns](#3-decomposition-patterns)
4. [The Strangler Pattern (Migration)](#4-the-strangler-pattern-migration)
5. [The Saga Pattern (Distributed Transactions)](#5-the-saga-pattern-distributed-transactions)
6. [CQRS & Event Sourcing](#6-cqrs--event-sourcing)
7. [Operational Patterns Every System Needs](#7-operational-patterns-every-system-needs)
8. [How the Patterns Fit Together](#8-how-the-patterns-fit-together)
9. [Interview-Ready Insights](#9-interview-ready-insights)
10. [Quick Glossary](#10-quick-glossary)

---

## 1. What Are Microservices, and Why?

A **microservice architecture** structures an application as a collection of **small, independently deployable services**, each owning **a single business capability** and **its own data**.

To understand why this exists, picture the alternative it reacts against: the **monolith**. In a monolith, all the code — orders, inventory, payments, users — lives in one codebase, deploys as one unit, and shares one database. That's simple at first, but as it grows: one tiny change forces a redeploy of *everything*; one team's bug can take down unrelated features; you can't scale just the busy part; and the whole thing is locked to one technology stack.

Microservices break that single block into independent pieces. The two defining rules are worth dwelling on, because everything else follows from them:

- **"Independently deployable"** — each service can be built, tested, and released on its own schedule without coordinating a giant lockstep deploy. This is the property that delivers most of the organizational benefit.
- **"Owns its data"** — each service has its *own* database, and no other service is allowed to reach into it directly. They communicate only through APIs/events. This is what keeps services truly decoupled — and it's also the source of the hardest problems (distributed transactions, consistency), which is exactly why patterns like Saga and CQRS exist.

The catch — and the theme of this whole guide — is that **you've traded a complicated codebase for a complicated *system*.** The complexity didn't vanish; it moved from inside one process out onto the network between many. Most of the patterns below are tools for managing that relocated complexity.

---

## 2. The Pros and Cons (and the Honest Trade-off)

| Pros                          | Cons                                  |
|-------------------------------|---------------------------------------|
| Independent scaling           | Distributed-system complexity         |
| Independent deployments       | Network latency overhead              |
| Tech-stack flexibility        | Data consistency is hard              |
| Fault isolation               | Debugging across services is hard     |

**Reading the trade-off honestly:** every pro on the left has a cost on the right that comes from the *same* underlying cause — the services are separated by a network instead of a function call.

- You can **scale** the payment service independently of the catalog (pro) — but now a request that used to be one in-process call is several network hops, each adding **latency** and a chance of failure (con).
- You can **deploy** one service without the others (pro) — but a bug now hides somewhere across a fleet of services, so **debugging** means tracing a request through many hops (con). This is why distributed tracing and centralized logging are mandatory, not optional.
- Each service **owns its data** for isolation (pro) — but a single business operation may span several databases, so you can't wrap it in one ACID transaction; **consistency becomes hard** (con), which is the entire reason Saga exists.

The practical wisdom: **microservices are a solution to an organizational scaling problem, not a default.** Small teams and early products are usually better off with a well-structured monolith, then decomposing later (see Strangler) when the monolith's coupling genuinely hurts.

---

## 3. Decomposition Patterns

The first hard question is: *where do you draw the lines between services?* Cut wrong and you get "microservices" that are so chatty and interdependent they're just a monolith spread painfully across a network (a "distributed monolith" — the worst of both worlds).

### Decompose by Business Capability

Identify what the business *does* — Orders, Inventory, Payments, Shipping — and make **one service per capability**. This aligns services with how the business is actually organized, which tends to keep them stable (business capabilities change less often than technical details).

### Decompose by Subdomain (Domain-Driven Design)

A more rigorous version using **DDD's bounded contexts**. A *bounded context* is a boundary within which a particular model and its terms have one precise meaning. "Customer" might mean something different in the Sales context than in the Support context — DDD makes those boundaries explicit, and each bounded context becomes a natural service boundary. The goal is **high cohesion inside a service, loose coupling between services.**

### The guiding heuristic

A good boundary is one where the service can change and deploy on its own most of the time. If two "services" always have to be deployed together, the line is in the wrong place.

---

## 4. The Strangler Pattern (Migration)

You rarely build microservices from scratch — you usually have a monolith you need to migrate *without* a risky big-bang rewrite. The **Strangler Pattern** (named after the strangler fig vine that grows around a tree and gradually replaces it) does this incrementally.

You place a **façade/proxy** in front of the monolith that intercepts all incoming requests. Initially it routes everything to the legacy app. Then, one capability at a time, you build a new microservice and reroute *just that endpoint* to it — leaving everything else untouched. Over time, more and more traffic shifts to new services until the monolith is "strangled" and can be retired.

```
[Client] ──► [Facade / Proxy] ──┬──► [Legacy Monolith]   (shrinking over time)
                                └──► [New Microservice]   (growing over time)
```

### Why it's the standard approach

- **Low risk** — you migrate in small, reversible steps instead of one terrifying cutover. If a new service misbehaves, route that endpoint back to the monolith.
- **Continuous delivery** — the system stays live and shippable throughout the migration.
- **Learn as you go** — early extractions teach you where the real boundaries are before you commit to the harder ones.

Essentially every team moving off a monolith uses some form of this.

---

## 5. The Saga Pattern (Distributed Transactions)

### The problem

In a monolith, a business operation that touches orders, inventory, and payments can be wrapped in a single **ACID transaction**: either it *all* commits or it *all* rolls back. In microservices, those three live in **three separate databases**, so there is no global transaction to wrap around them. If payment fails *after* you've already reserved inventory, nothing automatically undoes the reservation.

### The solution

A **Saga** is a **sequence of local transactions**. Each service does its own local (ACID) transaction and then emits an event/message that triggers the next step. If a step fails, the saga runs **compensating transactions** — explicit "undo" operations — to reverse the steps already completed. Sagas give you *eventual* consistency, not the instant atomicity of a single ACID transaction.

> Key mental shift: there's no automatic rollback. *You* must write the compensating action for each step (e.g., "release the inventory you reserved", "refund the payment"). Compensation is your responsibility, by design.

### Example: e-commerce order

```
Reserve Inventory  ──►  Charge Payment  ──►  Ship Order
       │                      │
       │   if payment fails:  │
       └──────────────────────┘
        release inventory, cancel order   (compensating transactions)
```

### Two coordination styles

**1. Choreography** — *no central coordinator.* Each service listens for events and publishes its own. Order service emits `OrderCreated`; Inventory hears it, reserves stock, emits `InventoryReserved`; Payment hears that, charges, and so on.

- *Pros:* simple for short flows, no single point of control, loosely coupled.
- *Cons:* the overall flow is implicit and scattered across services — hard to see the whole picture or debug as it grows. Risk of cyclic event dependencies.

**2. Orchestration** — *a central orchestrator* explicitly tells each service what to do and tracks progress.

- *Pros:* the workflow lives in one place — easy to understand, monitor, and modify; centralized failure handling.
- *Cons:* the orchestrator is extra infrastructure and can become a point of coupling/failure if overloaded with logic.

**Rule of thumb:** choreography for simple, short flows; orchestration once a workflow has many steps, branches, or needs clear observability.

---

## 6. CQRS & Event Sourcing

### CQRS — Command Query Responsibility Segregation

The core idea: **separate the model you write with from the model you read with.** "Commands" (writes that change state) and "Queries" (reads that return data) have fundamentally different needs, so you stop forcing one model to serve both.

```
                   ┌─►  [Write DB]  (normalized, transactional)
[Client] ──► API ──┤          │  replicates / projects
                   │          ▼
                   └─►  [Read DB(s)]  (denormalized, query-optimized)
```

- **Write model** — normalized and transactional; its job is to **enforce business rules** correctly.
- **Read model** — denormalized, pre-aggregated views shaped exactly for the queries the UI needs, so **reads are fast** and don't strain the write side.

The write side propagates changes to the read side (often asynchronously), which means the read model is typically **eventually consistent** — a deliberate trade-off you accept in exchange for read scalability and clean separation.

### When CQRS earns its complexity

It shines when **reads and writes have wildly different loads or shapes** — e.g., a system written to occasionally but read from constantly with complex queries. It is *overkill* for simple CRUD; the extra moving parts (two models, the sync between them, eventual-consistency handling) aren't worth it unless the read/write asymmetry is real.

### Event Sourcing (the common companion)

Instead of storing the *current state* and overwriting it, **store the full sequence of events** that led to that state ("OrderCreated", "ItemAdded", "OrderShipped"). The current state is *derived* by replaying events.

- You get a complete, immutable **audit log** for free, and can reconstruct state at any past point in time.
- It pairs naturally with CQRS: the event stream is the write side, and you build read models by projecting those events.
- Cost: more complexity, and querying current state requires building projections rather than a simple `SELECT`.

---

## 7. Operational Patterns Every System Needs

Once you have many services talking over a network, you need infrastructure-level patterns to keep the system reliable, discoverable, and secure. These are the patterns that turn "a pile of services" into "a system."

| Pattern             | Purpose                                                    | Examples              |
|---------------------|------------------------------------------------------------|-----------------------|
| **API Gateway**     | Single entry point: routing, auth, rate-limiting, aggregation | Kong, NGINX, AWS API GW |
| **Service Discovery** | Find dynamic service instances as they come and go        | Consul, Eureka        |
| **Circuit Breaker** | Stop cascading failures from a failing dependency          | Resilience4j, (Hystrix) |
| **Bulkhead**        | Isolate resource pools so one failure can't sink the rest  | (thread/connection pools) |
| **Sidecar**         | Run a helper alongside a service (logging, proxy, security) | basis of service mesh |
| **Service Mesh**    | Manage all service-to-service communication centrally       | Istio, Linkerd        |

### API Gateway

Clients shouldn't need to know about your dozens of internal services. The gateway is a **single front door**: it routes requests to the right service, handles **cross-cutting concerns** once (authentication, rate-limiting, TLS termination), and can **aggregate** several service calls into one client response. Without it, every client would have to know your internal topology and re-implement auth.

### Service Discovery

Service instances are **dynamic** — they scale up and down, restart, and get new IP addresses constantly. Hard-coding addresses is impossible. A **service registry** (Consul, Eureka) keeps a live list of healthy instances; services look up *who's available right now* instead of relying on fixed addresses.

### Circuit Breaker

If Service A calls Service B and B is failing, A's requests pile up waiting and time out — and whoever calls A then backs up too. This is a **cascading failure** that can take down the whole system. A circuit breaker watches the failure rate and, once it crosses a threshold, **"trips"** — failing fast (returning an error or fallback immediately) instead of waiting. After a cooldown it tests whether B has recovered. It's the same idea as an electrical breaker: cut the circuit to protect everything downstream.

```
CLOSED ──(failures exceed threshold)──► OPEN ──(after timeout)──► HALF-OPEN
   ▲                                                                  │
   └──────────────(test request succeeds)────────────────────────────┘
```

### Bulkhead

Named after the watertight compartments in a ship's hull: if one floods, the others keep the ship afloat. In software, you **isolate resources** (separate thread pools, connection pools) per dependency, so one misbehaving downstream can only exhaust *its own* pool — it can't consume every resource and starve unrelated work.

### Sidecar

Deploy a **helper process/container alongside** your main service that handles cross-cutting concerns — logging, monitoring, proxying, security — *without* baking that logic into the service's own code. The service focuses on business logic; the sidecar handles the plumbing. This is the building block of the service mesh.

### Service Mesh

Once every service has a sidecar proxy, you have a **mesh**: a dedicated infrastructure layer (Istio, Linkerd) that transparently manages **all service-to-service communication** — load balancing, retries, encryption (mTLS), traffic routing, and observability — *without changing application code*. It pulls networking concerns out of your services entirely and centralizes their control.

---

## 8. How the Patterns Fit Together

These patterns aren't a menu of unrelated options — they layer into a coherent system:

```
                          [ Clients ]
                               │
                        [ API Gateway ]        ← single entry, auth, routing
                               │
        ┌──────────────────────┼──────────────────────┐
        ▼                      ▼                       ▼
   [ Order Svc ]         [ Payment Svc ]        [ Inventory Svc ]
   (own DB, CQRS)        (own DB)               (own DB)
        │                      │                       │
        └────────── Saga (events / orchestrator) ──────┘   ← distributed txn
        
   Cross-cutting (every service):
   • Service Discovery — find each other
   • Circuit Breaker + Bulkhead — survive failures
   • Sidecar / Service Mesh — handle comms, security, observability
```

- **Decomposition** decides the boundaries.
- **Strangler** gets you there from a monolith.
- **Saga** keeps a multi-service operation consistent.
- **CQRS/Event Sourcing** handles read/write scale and auditability within a service.
- **Gateway, Discovery, Circuit Breaker, Bulkhead, Sidecar, Mesh** keep the whole thing reliable and operable.

---

## 9. Interview-Ready Insights

**Q: What problem do microservices actually solve?**
They solve an *organizational/scaling* problem: letting independent teams build, deploy, and scale parts of a system independently. They don't reduce complexity — they relocate it from inside one codebase onto the network between services.

**Q: When should you NOT use microservices?**
Early-stage products and small teams. A well-structured monolith is simpler and faster to build; decompose later (via Strangler) once coupling genuinely hurts. Premature microservices often produce a "distributed monolith" — the costs of both with the benefits of neither.

**Q: How do you handle a transaction across services without a global ACID transaction?**
The Saga pattern: a sequence of local transactions, each triggering the next, with compensating transactions to undo prior steps on failure. You trade instant atomicity for eventual consistency, and you write the rollbacks yourself.

**Q: Choreography vs. orchestration for Sagas?**
Choreography (event-driven, no coordinator) suits simple, short flows but scatters the logic. Orchestration (central coordinator) suits complex, multi-step flows where you need the workflow visible in one place and easier to monitor.

**Q: What does a circuit breaker prevent, and how?**
Cascading failures. When a dependency's failure rate crosses a threshold, the breaker trips and fails fast instead of letting requests pile up waiting on a dead service — protecting everything upstream until the dependency recovers.

**Q: Why CQRS, and what's the cost?**
Separating read and write models lets each be optimized independently — great when read and write loads/shapes differ sharply. The cost is added complexity and an eventually-consistent read model, so it's overkill for simple CRUD.

**Q: What's a service mesh and what does it replace?**
A dedicated infra layer (via sidecar proxies) that handles service-to-service comms — retries, load balancing, mTLS, observability — without app code changes. It pulls networking/reliability logic out of every service and centralizes it.

---

## 10. Quick Glossary

- **Monolith** — an application built and deployed as a single unit with one shared database.
- **Microservice** — a small, independently deployable service owning one capability and its own data.
- **Distributed monolith** — services so tightly coupled they must deploy together; an anti-pattern.
- **Bounded context (DDD)** — a boundary within which a model and its terms have one precise meaning.
- **Saga** — a sequence of local transactions with compensating actions, for cross-service consistency.
- **Compensating transaction** — an explicit "undo" for a completed saga step.
- **Choreography / Orchestration** — event-driven (no coordinator) vs. centrally-coordinated saga styles.
- **CQRS** — separating the write model (commands) from the read model (queries).
- **Event Sourcing** — storing the sequence of events rather than just current state.
- **API Gateway** — a single entry point handling routing and cross-cutting concerns.
- **Service Discovery** — a registry of live service instances so services can find each other.
- **Circuit Breaker** — fails fast when a dependency is unhealthy, preventing cascading failure.
- **Bulkhead** — resource isolation so one failure can't starve unrelated work.
- **Sidecar** — a helper process deployed alongside a service for cross-cutting concerns.
- **Service Mesh** — an infra layer managing all service-to-service communication transparently.
- **Eventual consistency** — replicas/views converge over time rather than instantly.

---

*Reference document. Contributions and corrections welcome.*
