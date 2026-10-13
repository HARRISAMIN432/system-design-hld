# Chapter 2 — Monolith vs Microservices (and How to Migrate)

> Personal study notes. Goal: understand both architectures deeply enough to choose between them and to migrate from one to the other without breaking production.

---

## Table of Contents

1. [Running Example](#1-running-example)
2. [Monolithic Architecture](#2-monolithic-architecture)
3. [Problems with Monoliths](#3-problems-with-monoliths)
4. [Microservice Architecture](#4-microservice-architecture)
5. [How Microservices Solve Each Problem](#5-how-microservices-solve-each-problem)
6. [Database Strategy](#6-database-strategy)
7. [Common Misconceptions](#7-common-misconceptions)
8. [Migrating from Monolith to Microservices](#8-migrating-from-monolith-to-microservices)
9. [Traffic Shifting: Canary and Strangler Pattern](#9-traffic-shifting-canary-and-strangler-pattern)
10. [Distributed Transactions: The Saga Pattern](#10-distributed-transactions-the-saga-pattern)
11. [Data Consistency: The Outbox Pattern](#11-data-consistency-the-outbox-pattern)
12. [Trade-offs: When NOT to Use Microservices](#12-trade-offs-when-not-to-use-microservices)
13. [Interview Questions](#13-interview-questions)
14. [Quick Revision Sheet](#14-quick-revision-sheet)

---

## 1. Running Example

Imagine an online learning or e-commerce product with these modules:

- **Auth** — sign up, log in
- **Course / Catalog** — create and browse items
- **Cart**
- **Payment**
- **Order** — confirm bookings
- **Profile**, **Ratings & Reviews**, **Search**

How we package these modules defines the architecture.

---

## 2. Monolithic Architecture

All modules live in **one codebase**, are built **together**, run **together** and are deployed **together**.

```
┌──────────────────── Monolith (one process) ────────────────────┐
│  Auth │ Catalog │ Cart │ Payment │ Order │ Profile │ Reviews   │
│                                                                │
│   Handlers / Controllers  (all in one repo, one entry point)   │
└──────────────────────────────┬─────────────────────────────────┘
                               │
                      ┌────────▼────────┐
                      │ Shared Database │
                      │ users, orders,  │
                      │ payments, ...   │
                      └─────────────────┘
```

### Characteristics

- **Single codebase** and a **single entry point** (e.g. one `index.js`).
- A *controller* is simply a handler: auth handler, payment handler, profile handler, reviews handler, and so on.
- Modules are **tightly coupled**.
- Build, run, test and deploy happen for **all modules at once**.
- Typically one **shared database** with separate tables per module.
- An error anywhere can take down the entire application.

---

## 3. Problems with Monoliths

### 3.1 Single Point of Failure

Because everything runs as one unit, a bug or crash in one module brings everything down.

**Scenario:** the system works for 100, then 10,000, then 100,000 users. At 1 million users the **Payment** module crashes. Result:

- Payment is down (expected).
- Auth is also down, so no one can sign up or log in.
- Cart and Order are also down.

One failing module made the whole product unusable.

### 3.2 Deployment Bottleneck

Suppose we only want to add **credit card support** to Payment. That is a small change in one module, yet:

- The entire application must be rebuilt, retested and redeployed.
- Releases become large, risky and slow.
- Many teams end up waiting on one release pipeline.

### 3.3 No Individual Scaling

Suppose Payment receives **20 million requests a day** while Auth, Cart and the rest together receive less than 2 million. Only Payment needs more capacity, but in a monolith:

- You cannot scale one module in isolation.
- You must scale the **whole** application, including modules that do not need it.
- This forces larger or more machines and **raises cost** unnecessarily.

---

## 4. Microservice Architecture

Each module becomes its own **service**. Services are **loosely coupled**: they do not depend on each other's code or runtime.

```
        ┌────────┐   ┌────────┐   ┌─────────┐   ┌─────────┐
        │  Auth  │   │  Cart  │   │  Order  │   │ Payment │
        │Service │   │Service │   │ Service │   │ Service │
        └───┬────┘   └───┬────┘   └────┬────┘   └────┬────┘
            │            │             │             │
        ┌───▼───┐    ┌───▼───┐     ┌───▼───┐     ┌───▼───┐
        │  DB 1 │    │  DB 2 │     │  DB 3 │     │  DB 4 │
        └───────┘    └───────┘     └───────┘     └───────┘
```

Each service can be **built, run, tested and deployed independently**. In code, every service has its own entry file (its own `index.js`) and runs on its own port. For example, a user service might run on port 4002 while another runs on a different port.

---

## 5. How Microservices Solve Each Problem

| Monolith problem | Microservice answer |
|---|---|
| Single point of failure | If the Payment service crashes, the User/Auth service is unaffected, because they are independent. Failures are **isolated**. |
| Deployment bottleneck | Adding credit card support means redeploying **only** the Payment service. |
| No individual scaling | If Payment gets 20 million requests, add more machines **for Payment only**. |

> Note: isolation is not automatic. If Order *calls* Payment synchronously and Payment is down, Order is affected unless you design for it (timeouts, retries, circuit breakers, queues). Loose coupling reduces blast radius; it does not remove the need for resilience patterns.

---

## 6. Database Strategy

| | Monolith | Microservices |
|---|---|---|
| Database | Usually one **shared** database | Each service has its **own dedicated** database |
| Layout | Separate tables per module in one DB | Separate DB per service |
| Technology | One choice for everything | Pick the best fit per service |

**Choosing per service:**

- A **User** service managing structured user data → a relational database (RDBMS, e.g. PostgreSQL) with an ORM such as Prisma.
- A **Cart** service where each item (e.g. each course) has different attributes → a **NoSQL key-value / document** store.

This is called *polyglot persistence*.

---

## 7. Common Misconceptions

**Misconception 1 — "A monolith runs on a single machine."**
False. When you need to scale a monolith you run **multiple copies** of the whole app on multiple servers (App Server 1, App Server 2, …). What you *cannot* do easily is scale one **module** alone.

**Misconception 2 — "Every microservice runs on its own tiny machine."**
False. If traffic is low, many microservices can share one machine. You separate them onto different machines **only when you need to scale** a particular service.

---

## 8. Migrating from Monolith to Microservices

Three golden rules:

1. **It is not a one-day activity.** Migration is gradual.
2. **Do not migrate the entire system in one go.** Move module by module.
3. **Do not shift 100% of traffic at once**, even after the new service is ready.

### Step-by-step process

| Step | Action | Example (Amazon-like system) |
|---|---|---|
| 1 | **Understand your monolith** | List modules: Auth, Search, Cart, Payment, Order |
| 2 | **Identify high-impact areas** | Payment gets heavy traffic and may crash soon, so extract it first |
| 3 | **Build API contracts and set up communication** | Decide synchronous (REST / gRPC) vs asynchronous (Kafka, RabbitMQ) |
| 4 | **Migrate module by module** | Pick one small module, e.g. Payment, and work on it |
| 5 | **Monitoring and observability** | Verify it works, handles traffic, and does not crash |
| 6 | **Scaling and optimization** | Add instances, give Payment more CPU, etc. |

### Code migration vs service migration

These are different:

- **Code migration:** create the new service repo, move the code, configure the new database, set up Docker, define the folder structure and entry point. Then test and deploy.
- **Service/module migration:** complete only when **100% of traffic** is routed to the new microservice.

### When to remove the old module

Do **not** delete the Payment module from the monolith as soon as the microservice exists. Decommission it only after 100% of traffic is flowing to the microservice and it is proven stable.

---

## 9. Traffic Shifting: Canary and Strangler Pattern

### Canary deployment

Route a **small subset of users** to the new version and widen gradually:

```
0.1%  ──►  1%  ──►  5%  ──►  10%  ──►  50%  ──►  100%
```

If errors appear at any stage, you stop or reduce the percentage.

### Strangler pattern

A **controller / router** sits in front and decides where each API request goes.

```
                    ┌────────────────────┐
  100 requests ───► │  Router/Controller │
                    └─────┬────────┬─────┘
                       90 │        │ 10
                  ┌───────▼──┐  ┌──▼────────────┐
                  │ Monolith │  │ Payment       │
                  │          │  │ Microservice  │
                  └──────────┘  └───────────────┘
```

- Start at 90% monolith / 10% microservice.
- If the microservice fails to handle even the 10%, set it back to 0% and send everything to the monolith (**rollback**).
- If it works, increase gradually until the monolith's version of that module is "strangled" and can be removed.

Canary tells you **how much** traffic to move; the strangler pattern provides the **routing mechanism** to do it.

---

## 10. Distributed Transactions: The Saga Pattern

**Problem:** with a database per service, one business operation may span several services.

Example: Order service must update 5 tables, User service 6 tables, Payment service 8 tables. Suppose User and Order succeed, but Payment only manages 2 of 8 and fails. In a single database you would simply roll back. Across several databases, a normal rollback is no longer available.

**Solution: Saga pattern.** A saga splits the transaction into a sequence of local transactions. If a step fails, **compensating transactions** undo the earlier steps (e.g. cancel the order, release the reserved item).

Two common styles:

| Style | How it works |
|---|---|
| **Choreography** | Services react to each other's events, with no central coordinator. |
| **Orchestration** | A central orchestrator tells each service what to do next and triggers compensations on failure. |

---

## 11. Data Consistency: The Outbox Pattern

**Problem:** during migration, both the monolith database and the new service's database hold payment data. They must stay consistent (no case where `x = 5` in one and `x = 4` in the other).

**Solution: Outbox pattern.**

```
 User pays
     │
     ▼
 Monolith DB ── write payment ──► payments table
     │                        └─► outbox table  (same transaction)
     ▼
 Relay reads outbox ─► publishes event to Kafka
                                  │
                                  ▼
                       Payment microservice consumes event
                                  │
                                  ▼
                       Payment microservice DB updated
```

Because the payment record and the outbox record are written in **one local transaction**, you never end up with "payment saved but event lost". The event is then delivered through a message broker such as Kafka.

---

## 12. Trade-offs: When NOT to Use Microservices

| Microservices add | Cost |
|---|---|
| Network calls between services | Latency, partial failures |
| Many databases | Harder joins and reporting |
| Many deployables | Operational complexity (CI/CD, monitoring, tracing) |
| Distributed transactions | Saga and outbox complexity |

**Rule of thumb:** a small team with a young product usually does better with a **well-structured monolith**, moving to microservices when specific pain points (scaling, release speed, team independence) appear.

---

## 13. Interview Questions

1. What is a monolith, and what does "tightly coupled" mean in practice?
2. Name three problems of a monolith and how microservices address each.
3. Can a monolith be scaled? How? What can't be scaled?
4. Must each microservice run on its own machine?
5. Why does each microservice get its own database? What are the downsides?
6. How would you migrate a payments module out of a monolith with zero downtime?
7. Explain canary deployment and the strangler pattern. How do they relate?
8. When do you decommission the old module?
9. How do you handle a transaction that spans three services?
10. How do you keep two databases consistent during migration?

---

## 14. Quick Revision Sheet

- **Monolith:** one codebase, tightly coupled, build/run/test/deploy together, shared DB.
- **Microservices:** many small services, loosely coupled, independent lifecycle, DB per service.
- **Monolith pains:** single point of failure, deployment bottleneck, no individual scaling (higher cost).
- **Migration rules:** gradual, module by module, never 100% traffic at once.
- **Steps:** understand → identify high-impact → API contracts and communication → migrate module → monitor → scale.
- **Traffic tools:** canary (how much) + strangler (where it goes).
- **Transactions:** Saga. **Consistency:** Outbox + message broker.
- **Remove old code only after** 100% traffic is on the new service.

---

**Previous:** [Chapter 1 — Roadmap](../Chapter-01-Course-Roadmap/README.md) · **Next:** [Chapter 3 — API Gateway vs Load Balancer](../Chapter-03-API-Gateway-vs-Load-Balancer/README.md)
