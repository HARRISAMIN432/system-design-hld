# Chapter 2 — Monolith vs. Microservices: Architecture, Trade-offs, and Migration

> **Chapter objective:** Understand monolithic and microservice architectures from first principles, evaluate their trade-offs, design reliable service boundaries, and migrate a production system incrementally without unnecessary downtime or data inconsistency.

## Learning Objectives

By the end of this chapter, you should be able to:

- Explain how application architecture differs from code organization, deployment, and runtime architecture.
- Understand monolithic, modular monolithic, and microservice architectures.
- Analyze coupling, cohesion, scalability, availability, and fault isolation.
- Design service boundaries and define clear ownership of data.
- Understand synchronous communication, asynchronous messaging, and distributed transactions.
- Explain the Strangler Fig pattern, canary deployments, the Saga pattern, and the Transactional Outbox pattern.
- Plan an incremental migration from a monolith to microservices.
- Identify when microservices are justified and when a monolith is the better engineering decision.

---

## Table of Contents

1. [Introduction: What Is Software Architecture?](#1-introduction-what-is-software-architecture)
2. [Our Running Example](#2-our-running-example)
3. [Monolithic Architecture](#3-monolithic-architecture)
4. [Modular Monolith: The Important Middle Ground](#4-modular-monolith-the-important-middle-ground)
5. [Problems That Can Emerge in Monoliths](#5-problems-that-can-emerge-in-monoliths)
6. [Microservice Architecture](#6-microservice-architecture)
7. [Monolith vs. Microservices: A Detailed Comparison](#7-monolith-vs-microservices-a-detailed-comparison)
8. [Service Boundaries and Domain-Driven Design](#8-service-boundaries-and-domain-driven-design)
9. [Database Architecture and Data Ownership](#9-database-architecture-and-data-ownership)
10. [Communication Between Microservices](#10-communication-between-microservices)
11. [Reliability, Failure Isolation, and Scalability](#11-reliability-failure-isolation-and-scalability)
12. [Migration Strategies and the Strangler Fig Pattern](#12-migration-strategies-and-the-strangler-fig-pattern)
13. [Canary Releases, Traffic Shifting, and Rollback](#13-canary-releases-traffic-shifting-and-rollback)
14. [Distributed Transactions: The Saga Pattern](#14-distributed-transactions-the-saga-pattern)
15. [Data Consistency: The Transactional Outbox Pattern](#15-data-consistency-the-transactional-outbox-pattern)
16. [A Practical Migration Walkthrough](#16-a-practical-migration-walkthrough)
17. [Observability and Operational Readiness](#17-observability-and-operational-readiness)
18. [Common Misconceptions](#18-common-misconceptions)
19. [When Should You Choose Each Architecture?](#19-when-should-you-choose-each-architecture)
20. [Architecture Decision Checklist](#20-architecture-decision-checklist)

---

# 1. Introduction: What Is Software Architecture?

Before comparing monoliths and microservices, we need to understand what software architecture actually describes.

Software architecture is the organization of a software system: its components, responsibilities, relationships, communication mechanisms, deployment units, and important design decisions.

Consider an online learning platform. It might contain:

- Authentication and authorization
- Course catalog
- Shopping cart
- Payment processing
- Order management
- Student profiles
- Reviews and ratings
- Search

These are **business capabilities**. They describe what the application does, not how its code must be deployed.

The same capabilities can be implemented using different architectures.

### 1.1 Four concepts that must not be confused

| Concept | Meaning | Example |
|---|---|---|
| Codebase | Where the source code is maintained | One Git repository |
| Module | A logical unit of functionality | Payment module |
| Process | A running instance of a program | A Node.js application |
| Deployment unit | What can be released independently | One application or one service |

A monolithic application might have one repository, several internal modules, and multiple running instances. A microservice system might use several repositories, but it can also use a monorepo containing many independently deployable services.

Therefore, **one repository does not necessarily mean one service, and multiple repositories do not necessarily mean microservices.**

### 1.2 Architecture is about boundaries and dependencies

Suppose the Payment module needs information about an order.

There are several possible designs:

**Design A: Direct in-process call**

```js
const order = orderModule.getOrder(orderId);
```

Both modules run within the same application process. The call is local and does not require network communication.

**Design B: HTTP request**

```js
const response = await fetch(
  `http://order-service/orders/${orderId}`
);

const order = await response.json();
```

Now Payment communicates with another running service over a network.

The business requirement is the same, but the operational characteristics differ.

The HTTP version introduces network latency, connection failures, timeouts, authentication between services, and potentially independent deployments. The in-process version avoids those network costs but usually shares the application's runtime and release lifecycle.

These differences are central to architectural design.

### 1.3 Coupling and cohesion

Two concepts help us reason about software architecture.

**Coupling** describes how strongly one component depends on another.

High coupling occurs when changing one module frequently requires changes to other modules. For example, if Payment directly modifies several internal tables owned by Order, the two modules are strongly coupled.

**Cohesion** describes how closely related the responsibilities within a component are.

High cohesion means a module contains functionality that belongs together. For example, payment authorization, refunds, and payment reconciliation may form a cohesive payment domain.

A useful architectural objective is:

- High cohesion within each module or service.
- Low coupling between modules or services.
- Explicit, stable interfaces between independently managed components.

Microservices do not automatically achieve these objectives. Poorly designed microservices can be more tightly coupled than a well-structured monolith.

---

# 2. Our Running Example

Throughout this chapter, we will use an online learning platform called **LearnHub**.

A student can browse courses, add them to a cart, pay for an order, and access purchased courses.

## 2.1 Business capabilities

| Module | Responsibility | Example operation |
|---|---|---|
| Auth | Authentication and identity | Log in |
| Catalog | Course information | Browse courses |
| Cart | Temporary selections | Add a course |
| Payment | Payment lifecycle | Authorize a payment |
| Order | Purchase lifecycle | Create an order |
| Profile | Student information | Update profile |
| Reviews | Feedback and ratings | Submit a review |
| Search | Course discovery | Search by keyword |

A simplified purchase workflow looks like this:

```text
Student
   |
   v
Browse Catalog
   |
   v
Add Course to Cart
   |
   v
Create Order
   |
   v
Process Payment
   |
   v
Confirm Order
   |
   v
Grant Course Access
```

The workflow expresses the business process. It does not imply that every step must be implemented as a separate microservice.

## 2.2 Why use the same example throughout?

Architecture decisions become easier to understand when we compare the same business operation under different designs.

For example:

- In a monolith, Order can call Payment directly.
- In a modular monolith, Order calls a defined Payment interface.
- In a microservice architecture, Order may call Payment over HTTP or communicate through events.
- In a distributed workflow, a Saga can coordinate payment, order confirmation, and course access.

The business objective remains the same; the architectural responsibilities and failure modes change.

---

# 3. Monolithic Architecture

## 3.1 Definition

A **monolithic architecture** organizes an application as a single deployable application.

Its business modules may be separated in source code, but they are normally built and released as one application unit.

For example, a Node.js application might contain authentication, catalog, cart, payment, and order functionality in one deployable process.

```text
                  LearnHub Application
       ┌───────────────────────────────────┐
       │                                   │
       │  Auth       Catalog      Cart     │
       │                                   │
       │  Payment    Order        Profile  │
       │                                   │
       │  Reviews    Search                │
       │                                   │
       └─────────────────┬─────────────────┘
                         |
                         v
                 Shared Database
```

The modules may use different classes, folders, controllers, and services. The defining characteristic is that they belong to the same application deployment unit.

## 3.2 Example project structure

```text
learnhub/
├── src/
│   ├── auth/
│   │   ├── auth.controller.js
│   │   ├── auth.service.js
│   │   └── auth.repository.js
│   ├── catalog/
│   │   ├── catalog.controller.js
│   │   └── catalog.service.js
│   ├── cart/
│   ├── payment/
│   ├── orders/
│   ├── profiles/
│   ├── reviews/
│   ├── app.js
│   └── server.js
├── tests/
├── package.json
└── Dockerfile
```

The exact folder structure is not important. What matters is that the application is built and deployed as one unit.

## 3.3 How a request flows through a monolith

Suppose a student requests:

```http
GET /courses/101
```

The request might travel through the following layers:

```text
HTTP Request
     |
     v
Course Controller
     |
     v
Course Service
     |
     v
Course Repository
     |
     v
Database
     |
     v
HTTP Response
```

A simplified implementation:

```js
// catalog.controller.js
async function getCourse(req, res) {
  const course = await catalogService.getCourse(
    req.params.id
  );

  if (!course) {
    return res.status(404).json({
      error: "Course not found"
    });
  }

  res.json(course);
}
```

```js
// catalog.service.js
async function getCourse(id) {
  return catalogRepository.findById(id);
}
```

These functions may run inside the same Node.js process. Calling one module from another generally involves an ordinary function call rather than a network request.

## 3.4 Characteristics of a monolith

A typical monolithic application has these properties:

1. **One deployment unit:** The application is released as a whole.
2. **In-process communication:** Internal modules commonly call each other directly.
3. **Shared runtime:** Modules typically use the same application runtime and process resources.
4. **Potentially shared database:** Modules may use separate tables in one database.
5. **Unified development lifecycle:** Changes often pass through one build and release pipeline.
6. **Simpler local transactions:** When modules share a transactional database, one database transaction can update multiple tables atomically.

These are common characteristics, not universal requirements. A monolith can use multiple databases, run on several machines, or publish events to external systems.

## 3.5 Advantages

### A. Simpler development

A developer can often run the application and its dependencies locally without starting numerous independently managed services.

### B. Straightforward debugging

A request can be traced through function calls in one process. Standard debuggers can inspect variables and stack traces directly.

### C. Lower operational overhead

One application may require fewer deployment pipelines, network policies, dashboards, and service-discovery mechanisms than dozens of independently deployed services.

### D. Easier local transactions

Consider creating an order and reserving its payment record in the same relational database.

```sql
BEGIN;

INSERT INTO orders (id, user_id, status)
VALUES (101, 7, 'pending');

INSERT INTO payments (order_id, status)
VALUES (101, 'pending');

COMMIT;
```

If both writes belong to the same transactional database, the database can commit or roll back the changes atomically.

This is much more straightforward than coordinating independent databases.

## 3.6 Disadvantages

A monolith can become difficult to maintain when its internal boundaries deteriorate.

For example, the Order module might directly access payment tables, the Cart module might modify order records, and controllers might contain business rules that should belong to services.

As the system grows, a small change can have unexpected effects across the application.

However, these problems are not inevitable. A well-designed modular monolith can maintain strong internal boundaries.

---

# 4. Modular Monolith: The Important Middle Ground

A common architectural mistake is assuming that the only choices are a tightly coupled monolith and microservices.

There is an important middle ground: the **modular monolith**.

## 4.1 Definition

A modular monolith is a single deployable application organized into clearly separated modules with explicit interfaces and controlled dependencies.

```text
              Modular Monolith
       ┌────────────────────────────┐
       │                            │
       │  ┌─────────┐ ┌──────────┐  │
       │  │ Catalog │ │  Orders  │  │
       │  └────┬────┘ └────┬─────┘  │
       │       │           │        │
       │  ┌────▼────┐ ┌────▼─────┐  │
       │  │ Catalog │ │ Payment  │  │
       │  │  Data   │ │  Module  │  │
       │  └─────────┘ └──────────┘  │
       │                            │
       └──────────────┬─────────────┘
                      |
                      v
               Shared Database
```

The diagram illustrates logical boundaries, not a mandatory folder or database design.

## 4.2 Why modularity matters

Compare these two implementations:

```js
// Tightly coupled
async function createOrder(orderData) {
  await db.query(
    "INSERT INTO orders (...) VALUES (...)"
  );

  await db.query(
    "INSERT INTO payments (...) VALUES (...)"
  );
}
```

```js
// Explicit module boundary
async function createOrder(orderData) {
  const order = await orderRepository.create(
    orderData
  );

  await paymentModule.createPendingPayment(order);

  return order;
}
```

The second example makes the dependency more visible. A production implementation still needs to consider transaction boundaries, error handling, and whether both operations can be committed atomically.

An interface alone does not guarantee architectural independence. The underlying modules must also respect their ownership rules.

## 4.3 Rules for a healthy modular monolith

- Each module owns a clearly defined business responsibility.
- Modules expose deliberate public interfaces.
- Other modules do not bypass interfaces to manipulate internal data.
- Dependencies follow a documented direction.
- Tests verify important module boundaries.
- Database tables have clear ownership, even if they share one physical database.

## 4.4 Why start here?

A modular monolith lets a team improve code boundaries without immediately taking on distributed-systems complexity.

Later, if a module needs independent scaling, deployment, or team ownership, a well-isolated module can become a candidate for extraction.

**Key principle:** Establish good boundaries before introducing network boundaries.

---

# 5. Problems That Can Emerge in Monoliths

A monolith does not automatically suffer from every problem below. These problems emerge when the application's architecture, workload, or operational requirements outgrow its current design.

## 5.1 Deployment bottleneck

Imagine the team wants to add a new card processor to Payment.

Even though the change is limited to payment functionality, the deployment process may require rebuilding, testing, and releasing the entire application.

Potential consequences include:

- Longer release cycles.
- Larger regression-testing requirements.
- Coordination between teams that share one release process.
- A greater chance that an unrelated change affects the release.

For a small application, one release process may be an advantage. It becomes a bottleneck when independent teams need to release frequently without coordinating every change.

## 5.2 Limited independent scaling

Suppose Payment uses 70% of the application's CPU time, while Catalog uses 10%.

If both modules run inside the same application process, adding application instances usually replicates both modules.

```text
                Load Balancer
                 /          \
                v            v
         Application A   Application B
         Auth             Auth
         Catalog          Catalog
         Payment          Payment
         Orders           Orders
```

If Payment needs more compute capacity, the team may have to replicate the entire application.

This can increase infrastructure costs because every instance also runs functionality that may not need additional capacity.

A monolith can still scale horizontally. Its limitation is that the internal modules are not normally scaled independently.

## 5.3 Shared resource contention

Modules may compete for:

- CPU and memory.
- Database connections.
- Database locks.
- Thread pools or worker capacity.
- File descriptors.
- External API rate limits.

For example, an expensive search operation may consume so much CPU that login requests become slower, even though the authentication code is functioning correctly.

Resource isolation can mitigate these problems, but it may require additional application-level or infrastructure-level engineering.

## 5.4 Failure blast radius

Suppose an unbounded loop in a search operation consumes all available memory.

If Search runs inside the same process as Auth and Payment, the runtime may terminate the entire process. Other modules in that process can become unavailable too.

This is a possible failure mode, not a defining property of monoliths. A monolith with multiple replicas, health checks, resource limits, and careful coding can remain highly available.

The architectural concern is that modules sharing a process and resources may share a failure domain.

## 5.5 Tight coupling and regression risk

Consider the following dependency chain:

```text
Payment
   |
   v
Orders
   |
   v
Cart
   |
   v
Catalog
```

If Payment depends on internal behavior in Orders, and Orders depends on internal behavior in Cart, a change to Catalog may eventually affect Payment.

The real problem is not the number of modules. It is the number and strength of their dependencies.

## 5.6 Organizational bottlenecks

As multiple teams work in the same codebase, they may compete over:

- Shared release schedules.
- Database schema changes.
- Centralized testing pipelines.
- Shared ownership of business logic.
- Conflicting changes to common modules.

A modular monolith can reduce these problems through ownership rules and automation. Microservices can provide more independence, but only if teams also own their service's deployment and operational responsibilities.

### Summary

| Problem | Underlying cause | Possible mitigation |
|---|---|---|
| Slow releases | Shared deployment lifecycle | Modular boundaries and automated CI/CD |
| Scaling inefficiency | Shared deployment and resource units | Optimize workloads or extract hot modules |
| Failure propagation | Shared process or resources | Isolation, limits, replicas, fault containment |
| Tight coupling | Uncontrolled dependencies | Explicit interfaces and module ownership |
| Team bottlenecks | Shared ownership and release coordination | Clear ownership and independent delivery where justified |

---

# 6. Microservice Architecture

## 6.1 Definition

A microservice architecture organizes an application as a collection of independently deployable services, each responsible for a cohesive business capability.

Each service typically has:

- A defined business responsibility.
- A contract through which other components interact with it.
- Its own deployment and release lifecycle.
- Clear ownership of its data.
- Independent operational and scaling characteristics.

For LearnHub, the architecture might look like this:

```text
                    Clients
                       |
                       v
                API Gateway / BFF
                       |
          ┌────────────┼────────────┐
          |            |            |
          v            v            v
       Catalog       Orders       Auth
       Service       Service     Service
          |            |            |
          v            v            v
       Catalog      Orders DB     Auth DB
         DB
                       |
                       v
                 Payment Service
                       |
                       v
                   Payments DB
```

The diagram is simplified. A real architecture may route each service through a gateway, use internal service discovery, and communicate through synchronous APIs or asynchronous events.

## 6.2 Independent deployment

Suppose Payment introduces a new payment provider.

In a microservice architecture, the team can deploy the updated Payment service without rebuilding Catalog or Auth, provided the change does not require incompatible changes to their contracts or infrastructure.

This reduces the coordination required for independent releases.

However, a service can still depend on coordinated database migrations, event schemas, authentication configuration, and external systems. Independent deployment is a capability that must be designed and maintained, not a guarantee that every change is isolated.

## 6.3 Independent scaling

Suppose traffic looks like this:

| Service | Daily requests |
|---|---:|
| Catalog | 5 million |
| Auth | 1 million |
| Payment | 20 million |
| Reviews | 300,000 |

The figures are illustrative and do not by themselves determine capacity requirements.

Payment may require more instances because it handles more traffic or more expensive operations.

```text
                    Load Balancer
                         |
              ┌──────────┼──────────┐
              v          v          v
          Payment 1  Payment 2  Payment 3
```

Catalog can have a different number of instances. The services can be scaled according to their own workload, resource consumption, latency targets, and dependency limits.

Daily request volume alone is insufficient for capacity planning. Peak requests per second, request complexity, database usage, and external provider limits also matter.

## 6.4 Service-to-service communication

Payment and Orders might communicate through HTTP:

```text
Order Service
     |
     | POST /payments
     v
Payment Service
     |
     v
Payments Database
```

Or Payment might publish an event:

```text
Order Service
     |
     v
Message Broker
     |
     v
Payment Service
```

These approaches have different latency, consistency, failure-handling, and coupling characteristics. They are explored in Section 10.

## 6.5 Why a service usually owns its data

In a mature microservice design, the Payment service owns payment records and payment state transitions. Orders owns order records and order state transitions.

Other services should not normally modify those records directly.

For example:

```text
Orders Service ── API / Events ──► Payment Service
                                         |
                                         v
                                    Payments DB
```

This ownership rule prevents other services from depending on the Payment database's internal schema.

## 6.6 Advantages

- Independent deployment and release cadence.
- Independent scaling of suitable workloads.
- Better alignment between business capabilities and team ownership.
- Potentially smaller failure domains.
- Freedom to choose different technologies where there is a justified need.
- Clearer ownership of data and business behavior.

## 6.7 Costs

Microservices introduce additional engineering responsibilities:

- Network communication and latency.
- Partial failures and timeouts.
- Distributed tracing and centralized observability.
- Service discovery and identity management.
- Data consistency across services.
- Schema and API versioning.
- More deployment pipelines and runtime components.
- More complex integration testing.
- Additional infrastructure and operational costs.

Microservices exchange some forms of local complexity for distributed-systems complexity.

---

# 7. Monolith vs. Microservices: A Detailed Comparison

| Dimension | Monolithic application | Microservice architecture |
|---|---|---|
| Deployment | Usually one application unit | Multiple independently deployable services |
| Internal communication | Often direct function calls | Network APIs and/or messaging |
| Scaling | Usually scales the whole application | Services can scale independently |
| Data access | Often a shared database | Explicit data ownership per service |
| Transactions | Local database transactions are often straightforward | Cross-service transactions require coordination |
| Fault isolation | Modules may share a process and resources | Service boundaries can limit some failures |
| Testing | Many tests can run in one application context | Unit, contract, integration, and workflow tests |
| Observability | Often simpler request tracing | Requires correlation across services |
| Technology choices | Usually fewer technology stacks | Different technologies can be used where justified |
| Deployment overhead | Generally lower | Generally higher |
| Team ownership | Often shared application ownership | Can align with independently owned services |
| Best fit | Many small and medium systems | Systems with justified independence requirements |

## 7.1 Does a monolith mean one server?

No.

A monolithic application can run several identical instances behind a load balancer:

```text
                 Load Balancer
                  /         \
                 v           v
          Monolith A     Monolith B
```

This is horizontal scaling. It improves capacity and can improve availability when instances and their dependencies are configured correctly.

The distinction is that each instance normally contains all application modules.

## 7.2 Does a microservice mean one server?

No.

A service can run multiple replicas, and several different services can share the same physical machine.

The deployment boundary is logical and operational; it does not have to correspond to one physical server.

## 7.3 Which architecture is faster?

Neither architecture is universally faster.

A monolith can make direct function calls without network serialization or network latency. Microservices introduce network hops, but independent scaling, caching, and isolation can improve end-to-end performance under appropriate workloads.

A poorly optimized monolith may be slow. A poorly designed microservice system may be even slower because a single user request crosses many services.

## 7.4 Which architecture is more reliable?

Reliability depends on the failure model and the system's design.

Microservices can isolate certain failures, but a critical request may depend on multiple services. If any required dependency is unavailable, the operation may fail.

A monolith can achieve high availability with multiple replicas and resilient dependencies.

**Architectural separation creates opportunities for isolation; it does not automatically create reliability.**

---

# 8. Service Boundaries and Domain-Driven Design

One of the hardest problems in microservice architecture is deciding where a service should begin and end.

A poor boundary can create a distributed monolith: several services that are deployed separately but must constantly coordinate and change together.

## 8.1 Do not create one service per database table

Suppose the database contains:

```text
users
courses
orders
order_items
payments
refunds
reviews
```

It does not follow that seven microservices should be created.

A table is a storage structure. A service represents a cohesive business responsibility.

For example, Payments and Refunds may belong to the same payment domain because refunds depend on payment lifecycle rules.

## 8.2 Business capability boundaries

A useful starting point is to identify distinct business capabilities:

- Catalog manages course information.
- Orders manages the purchase lifecycle.
- Payments manages authorization, capture, refunds, and reconciliation.
- Identity manages authentication and identity-related responsibilities.

These boundaries reduce the need for unrelated teams to coordinate every change.

## 8.3 Bounded contexts

Domain-Driven Design introduces the concept of a **bounded context**: a boundary within which a particular domain model and its terminology have a consistent meaning.

For example, the term "customer" may mean:

- An authenticated account in Identity.
- A purchaser with an order history in Orders.
- A payer associated with a transaction in Payments.

These models are related, but they are not necessarily identical. Each context should own the model that serves its business responsibilities.

## 8.4 A practical boundary test

Before extracting a service, ask:

1. Does it represent a distinct business capability?
2. Can its responsibilities be described clearly?
3. Can its data ownership be defined?
4. Can other components interact through a stable contract?
5. Can the service be deployed without routinely coordinating changes to every dependent service?
6. Is there a real scaling, reliability, organizational, security, or delivery reason to separate it?

If most answers are unclear, begin by improving the module boundary inside the monolith.

---

# 9. Database Architecture and Data Ownership

Database design is one of the most important differences between a conventional monolith and a well-designed microservice system.

## 9.1 Shared database in a monolith

A monolith might use one relational database:

```text
                 LearnHub
                     |
                     v
               PostgreSQL
          ┌──────────┼──────────┐
          v          v          v
       courses     orders     payments
```

Different modules can use separate tables while sharing the same database engine.

Advantages include straightforward joins, database-level constraints, and local transactions across tables.

The danger appears when modules directly modify each other's data without clear ownership.

## 9.2 Database per service

The database-per-service principle means that each service controls its own persistent data and other services access that data through defined contracts.

```text
Catalog Service ─────► Catalog DB

Orders Service ──────► Orders DB

Payment Service ─────► Payments DB
```

This is primarily an ownership and access-boundary principle. The databases do not necessarily need to run on separate physical machines or use different database products.

A team can enforce logical separation using schemas, permissions, and controlled access while sharing infrastructure.

## 9.3 Why not let Orders query the Payments database directly?

Consider this query:

```sql
SELECT o.id, p.status
FROM orders o
JOIN payments p ON p.order_id = o.id;
```

In a shared-database monolith, this may be perfectly reasonable.

In a microservice architecture where Payments owns its data, allowing Orders to query the Payments database directly introduces coupling:

- Orders depends on the internal schema of Payments.
- Payment migrations can break Orders.
- Database access permissions become harder to manage.
- Independent deployment becomes less reliable.
- Ownership becomes ambiguous.

A better design may expose a Payment API or publish payment status events.

For read-heavy workflows, a reporting database or materialized read model may be appropriate.

## 9.4 How does a service get another service's data?

There are several options.

### Option A: Request the data

Orders calls Payment when it needs a current payment status.

Advantage: it can obtain an up-to-date response from the service.

Disadvantage: the request depends on Payment being reachable and responsive.

### Option B: Maintain a local projection

Payment publishes events such as:

```json
{
  "eventId": "evt-1001",
  "type": "PaymentSucceeded",
  "orderId": "ord-101",
  "paymentId": "pay-501"
}
```

Orders consumes the event and updates its own local view.

Advantage: many reads can happen without calling Payment.

Disadvantage: the local view may temporarily lag behind the source of truth.

This is a form of event-driven integration. The local projection should be treated as a derived representation, not as permission to modify the original payment record.

## 9.5 Polyglot persistence

**Polyglot persistence** means choosing storage technologies according to a service's actual requirements rather than forcing every service to use the same database.

Examples:

| Requirement | Potential choice |
|---|---|
| Relational integrity and transactional records | PostgreSQL |
| Flexible document-shaped data | MongoDB |
| High-volume caching and short-lived data | Redis |
| Full-text search | Elasticsearch or OpenSearch |
| Analytical queries over large datasets | A warehouse or analytical database |

These are examples, not mandatory mappings. PostgreSQL can handle many workloads that might otherwise be assigned to NoSQL systems.

For example, a Cart service does not automatically require a document database merely because cart items have different attributes. The decision should be based on query patterns, consistency requirements, operational skills, and measured workload.

**Rule:** Introduce a new database technology to solve a concrete problem, not simply because microservices make it possible.

## 9.6 The cost of database separation

Database-per-service design can make independent ownership easier, but it complicates:

- Cross-service joins.
- Reporting.
- Referential integrity across databases.
- Transactions spanning services.
- Data migrations and backfills.
- Reconciliation and consistency checks.

A system must account for these costs before committing to distributed data ownership.

---

# 10. Communication Between Microservices

Services generally communicate through synchronous requests, asynchronous messages, or a combination of both.

## 10.1 Synchronous communication

The caller sends a request and waits for a response.

Common mechanisms include REST over HTTP and gRPC.

```text
Client
  |
  v
Orders Service
  |
  | POST /payments
  v
Payment Service
  |
  v
Response: payment authorized
```

Example:

```js
async function requestPayment(orderId, amount) {
  const response = await fetch(
    "http://payment-service/payments",
    {
      method: "POST",
      headers: {
        "content-type": "application/json"
      },
      body: JSON.stringify({
        orderId,
        amount
      }),
      signal: AbortSignal.timeout(3000)
    }
  );

  if (!response.ok) {
    throw new Error(
      `Payment service returned ${response.status}`
    );
  }

  return response.json();
}
```

The example demonstrates a timeout and response check. Production payment APIs also need authentication, validation, safe retry behavior, structured error handling, and idempotency.

### Advantages

- Simple request-response model.
- Immediate feedback to the caller.
- Suitable for operations requiring a result before continuing.

### Disadvantages

- The caller depends on the callee being available.
- Network latency increases response time.
- Timeouts can leave the caller uncertain whether an operation succeeded.
- Chains of synchronous dependencies can amplify failures.

## 10.2 Asynchronous communication

The producer publishes a message or event without requiring the consumer to finish the work before the producer continues.

Common mechanisms include Kafka and RabbitMQ.

```text
Orders Service
      |
      v
 Message Broker
      |
      v
Payment Service
```

An event could look like this:

```json
{
  "eventId": "evt-1002",
  "type": "OrderPlaced",
  "orderId": "ord-102",
  "occurredAt": "2026-10-13T18:00:00Z"
}
```

The producer and consumer may run at different times, provided the message is retained and delivered according to the broker's guarantees.

### Advantages

- Producer and consumer do not need to be active simultaneously.
- Bursts of work can be buffered.
- Consumers can process work asynchronously.
- Event-driven workflows can reduce direct runtime dependencies.

### Disadvantages

- Messages may be delayed or delivered more than once.
- Consumers need idempotent processing.
- Debugging requires correlation across components.
- Event schemas and ordering require deliberate design.
- Eventual consistency may be visible to users.

## 10.3 Choosing the communication style

| Requirement | Likely approach |
|---|---|
| Validate a request and return a result | Synchronous API |
| Read current account information | Synchronous API |
| Notify consumers that an order was placed | Asynchronous event |
| Process a background task | Queue or event |
| Coordinate a long-running business workflow | Saga using APIs, events, or both |

These are guidelines rather than absolute rules.

## 10.4 Timeouts, retries, and idempotency

Distributed requests can fail in ambiguous ways.

Suppose Orders sends a payment request. Payment processes it successfully, but the response is lost. Orders cannot infer from the timeout alone that no payment occurred.

Blindly retrying could create a second charge.

An **idempotency key** helps make repeated requests represent the same logical operation.

```http
POST /payments
Idempotency-Key: order-102-payment
Content-Type: application/json
```

Payment records the key and the result. When the same logical request is repeated, the service returns the existing result rather than creating another payment.

The key must be associated with a correctly defined operation and request payload. Production implementations also need concurrency control and a suitable retention policy.

Retries should use bounded attempts and backoff. They should not be applied indiscriminately to operations that may have irreversible side effects.

---

# 11. Reliability, Failure Isolation, and Scalability

A major reason to consider microservices is the ability to control the effects of failures and allocate resources independently.

However, independent services create distributed failure modes that must be addressed explicitly.

## 11.1 Availability and dependency chains

Suppose an endpoint requires all three services:

```text
Request
   |
   v
Orders ──► Payment ──► Identity
```

If every dependency is mandatory for every request, a failure in any required dependency can cause the whole operation to fail.

For illustration, if three independent services each have 99.9% availability, and the request requires all three to be available, the approximate combined availability is:

\[
0.999^3 \approx 0.997
\]

That is approximately 99.7% availability for this simplified dependency model.

Real systems have correlated failures, caching, fallbacks, varying request paths, and different definitions of availability, so this calculation is illustrative rather than a production prediction.

The lesson is that adding service boundaries can increase the number of dependencies a request must survive.

## 11.2 Circuit breakers

A circuit breaker prevents repeated calls to a dependency that is failing.

A simplified circuit breaker has three states:

- **Closed:** Requests are allowed; failures are measured.
- **Open:** Calls are blocked temporarily.
- **Half-open:** A limited number of test requests determine whether recovery has occurred.

```text
             Failures exceed threshold
       ┌─────────────────────────────────┐
       │                                 v
   CLOSED ─────────────────────────────► OPEN
      ^                                    |
      |                                    |
      |           Recovery test            |
      └──────────── HALF-OPEN ◄─────────────┘
```

A circuit breaker can reduce wasted work and prevent an unhealthy dependency from being overwhelmed. It does not repair the dependency or replace appropriate timeout and error-handling policies.

## 11.3 Bulkheads

A bulkhead isolates resources so one workload cannot consume all capacity.

For example, a service might reserve separate concurrency limits for:

- Payment-provider calls.
- Catalog lookups.
- Background reconciliation.

This prevents one slow dependency from occupying every available request slot.

## 11.4 Horizontal scaling

A service can run multiple replicas:

```text
                  Load Balancer
                  /     |     \
                 v      v      v
                P1     P2      P3
```

This works best when requests can be distributed across replicas without relying on unshared local state.

The database and external dependencies must also support the resulting load. Adding application replicas does not solve a database bottleneck automatically.

## 11.5 Graceful degradation

Not every dependency must be available for every feature.

For example, if Reviews is unavailable, LearnHub may still allow students to browse and purchase courses.

```text
Catalog available
Payment available
Orders available
Reviews unavailable
```

A resilient application should preserve essential functionality where possible rather than treating every failure as a reason to reject all requests.

Payment processing itself requires stricter handling. A failed payment dependency should not be interpreted as successful payment, and an ambiguous payment outcome must be reconciled safely.

---

# 12. Migration Strategies and the Strangler Fig Pattern

Migrating a monolith to microservices is an architectural transition, not simply moving files into separate repositories.

The safest approach is usually incremental: preserve the existing system, introduce a new boundary, transfer responsibility gradually, and remove the old implementation only after the new one is proven reliable.

## 12.1 Why a big-bang migration is risky

Suppose LearnHub has years of working functionality.

Rebuilding Auth, Catalog, Cart, Payment, and Orders simultaneously introduces many risks:

- Existing business rules may be missed.
- Data migration may fail.
- New services may behave differently under real traffic.
- Several independent failure modes appear at once.
- Rollback becomes difficult if the data models have diverged.

A gradual migration limits the amount of change introduced at any one time.

## 12.2 Step 1: Understand the existing system

Before extracting anything, identify:

- Modules and their responsibilities.
- Dependencies between modules.
- Tables each module reads and writes.
- Scheduled jobs and background workers.
- External integrations.
- Business invariants and transaction boundaries.
- Traffic patterns and performance bottlenecks.
- Monitoring gaps and current failure rates.

A dependency graph can reveal modules that are difficult to separate:

```text
Catalog ─────► Orders
Cart ─────────► Orders
Orders ───────► Payment
Orders ───────► Profiles
Payment ──────► External Provider
```

Do not select a service boundary based on folder size alone.

## 12.3 Step 2: Select a suitable extraction candidate

A suitable first candidate often has:

- A clearly defined responsibility.
- Limited dependencies on internal monolith behavior.
- An understandable data model.
- A measurable reason to separate it.
- A safe way to route requests and observe outcomes.

Payment may be a candidate because it has distinct security and integration requirements, but it can also be a difficult first extraction due to financial correctness and reconciliation requirements.

A lower-risk module may be a better learning opportunity.

The correct choice depends on the system, not on a universal extraction order.

## 12.4 Step 3: Define the contract

Before moving the implementation, define the interface between the monolith and the new service.

For example:

```http
POST /payments
GET /payments/{paymentId}
POST /payments/{paymentId}/refunds
```

Document:

- Request and response schemas.
- Authentication and authorization.
- Error codes.
- Timeout behavior.
- Idempotency requirements.
- Versioning rules.
- Ownership of payment state.
- Compatibility expectations during rollout.

A stable contract lets the old application and new service evolve with less coordination.

## 12.5 Step 4: Separate the data ownership

Initially, the monolith may still own the authoritative data. The extraction must establish how data will move to the new service and how competing writes will be prevented.

Possible approaches include:

- Moving ownership after a controlled data migration.
- Capturing changes through an outbox or change-data-capture mechanism.
- Using a transitional adapter while the old interface remains available.
- Backfilling data and validating it before switching writes.

The correct strategy depends on whether the data must remain writable throughout the transition and how much downtime is acceptable.

**Do not casually allow both implementations to write the same business records.** Dual writes create conflicts, duplicated side effects, and ambiguous ownership unless a deliberate consistency strategy exists.

## 12.6 Step 5: Deploy the new service without sending production traffic

First verify the new service in a controlled environment.

Check:

- Unit and integration tests.
- Database migrations.
- Authentication and authorization.
- Contract compatibility.
- Resource consumption.
- Logging, metrics, and tracing.
- Dependency failure handling.
- Rollback and recovery procedures.

Deploying the service does not require routing real user requests to it immediately.

## 12.7 Step 6: Route a controlled amount of traffic

Use a routing layer to direct selected requests to the new implementation while the existing implementation remains available.

Monitor the new service against the old system using suitable metrics, such as:

- Error rate.
- Latency percentiles.
- Payment success and failure rates.
- Duplicate operation rates.
- Database consistency checks.
- External provider responses.

Increase traffic only when the results meet predefined acceptance criteria.

## 12.8 Step 7: Transfer ownership and remove the old implementation

Once the new service handles the relevant operations reliably:

1. Route all intended traffic to the new service.
2. Verify background jobs and non-HTTP entry points.
3. Ensure the old implementation no longer receives writes.
4. Reconcile outstanding records and pending operations.
5. Remove old database access paths when safe.
6. Delete the obsolete module after the rollback and recovery requirements are satisfied.

The final step is important: an extraction is incomplete if the new service is active but the monolith still independently owns or modifies the same business state.

---

# 13. Canary Releases, Traffic Shifting, and Rollback

Two concepts are often confused: canary deployment and the Strangler Fig pattern.

They can work together, but they solve different problems.

## 13.1 Canary deployment

A canary deployment gradually exposes a new version or implementation to a subset of traffic.

For example:

```text
Version 1: 100% of traffic
             |
             v
Version 2:   1% of traffic
             |
             v
Version 2:   5% of traffic
             |
             v
Version 2:  10% of traffic
             |
             v
Version 2:  50% of traffic
             |
             v
Version 2: 100% of traffic
```

The percentages are illustrative, not a required sequence.

At each stage, evaluate error rates, latency, business outcomes, and dependency health.

If the canary fails the acceptance criteria, stop the rollout or shift traffic back to the known-good implementation.

A canary can be used to release a new version of a single service or to introduce an extracted service into an existing application.

## 13.2 The Strangler Fig pattern

The Strangler Fig pattern is an incremental modernization strategy in which new functionality gradually takes over responsibilities previously handled by a legacy system.

A routing layer determines whether a request is handled by the old implementation or the new one.

```text
                    Incoming Requests
                           |
                           v
                    Routing Layer
                      /       \
                     v         v
                Monolith   New Service
```

At the beginning, nearly all requests may go to the monolith. As functionality is extracted, the new service handles more of the relevant operations.

Eventually, the old implementation is no longer required for that capability.

## 13.3 How do the two work together?

The Strangler Fig pattern describes the broader migration strategy. Canary deployment is one technique for controlling the rollout.

For example:

1. Keep the monolith as the default handler.
2. Deploy a Payment microservice.
3. Route a small, controlled subset of eligible requests to it.
4. Compare results against explicit safety criteria.
5. Increase traffic progressively.
6. Roll back routing if the new path fails.
7. Retire the old Payment implementation after ownership has fully transferred.

A percentage split is not always the safest strategy for stateful operations. Payments may need routing based on order ownership, account identity, or idempotency keys so repeated operations consistently reach the correct state owner.

## 13.4 A critical rollback limitation

Changing the router can restore the old request path, but it does not reverse database writes that have already happened.

If the new service has charged a customer or created a payment record, sending future requests to the monolith does not undo that operation.

Therefore, safe rollback requires:

- Compatible data models or a defined reconciliation strategy.
- Controlled write ownership.
- Idempotent operations.
- Clear handling of in-flight requests.
- Business-level compensation where necessary.

**Traffic rollback and data rollback are different operations.**

---

# 14. Distributed Transactions: The Saga Pattern

## 14.1 The problem

In a monolith using one transactional database, a purchase workflow may update several tables within a single transaction.

```text
BEGIN
   |
   v
Create Order
   |
   v
Reserve Required Resource
   |
   v
Record Payment State
   |
   v
COMMIT
```

If an operation fails before the commit, the database can roll back the local transaction.

Now imagine Orders, Inventory, and Payment each own separate databases.

```text
Orders Service ─────► Orders DB
Inventory Service ──► Inventory DB
Payment Service ────► Payments DB
```

A normal local database transaction cannot atomically commit changes across all three databases.

If the order is created and inventory is reserved but payment fails, the system must decide how to handle the earlier successful operations.

## 14.2 Definition of a Saga

A **Saga** is a sequence of local transactions that collectively implement a business workflow.

Each service commits its own local transaction. If a later step fails, the workflow may invoke compensating actions to address earlier completed steps.

For example:

| Step | Local transaction | Possible compensation |
|---|---|---|
| 1 | Create pending order | Cancel the order |
| 2 | Reserve inventory or entitlement | Release the reservation |
| 3 | Authorize payment | Void the authorization |
| 4 | Confirm order | Start an appropriate cancellation workflow if necessary |

The compensation depends on the operation. A completed card capture, for example, may require a refund rather than a simple rollback.

A compensation is a new business operation, not a database rollback. It may itself fail and require retries or manual intervention.

## 14.3 Saga choreography

In choreography, services react to events published by other services.

```text
Order Created
      |
      v
Inventory Service
      |
      v
Inventory Reserved
      |
      v
Payment Service
      |
      v
Payment Authorized
      |
      v
Order Confirmed
```

Each service decides what to do when it receives an event.

**Advantages**

- No central coordinator is required.
- Services can remain relatively autonomous.
- The design can be effective for straightforward event-driven workflows.

**Disadvantages**

- Workflow logic becomes distributed across event handlers.
- Understanding the entire process may be difficult.
- Complex branching and compensation paths can become hard to manage.

## 14.4 Saga orchestration

In orchestration, a coordinator explicitly manages the workflow.

```text
                 Saga Orchestrator
                  /      |      \
                 v       v       v
              Orders  Inventory Payment
```

The orchestrator tracks progress and decides which operation to execute next.

For example:

```text
Create Order
     |
     v
Reserve Inventory
     |
     +---- failure ----> Cancel Order
     |
     v
Authorize Payment
     |
     +---- failure ----> Release Inventory
     |                  Cancel Order
     |
     v
Confirm Order
```

This makes the business workflow easier to inspect centrally, but introduces the need to operate and recover the orchestrator.

## 14.5 Important Saga guarantees and limitations

A Saga does not provide the same isolation and atomicity as a single ACID transaction across all participants.

Other requests may observe intermediate states, and compensating actions may not restore the exact previous world state.

For example, if an order has already triggered an email notification, cancelling the order cannot make the email unsent.

A robust Saga needs:

- Durable workflow state.
- Idempotent steps and compensations.
- Timeouts and retry policies.
- Handling for delayed or duplicated events.
- Recovery after process restarts.
- Reconciliation for ambiguous outcomes.
- Explicit business states such as `PENDING`, `CONFIRMED`, and `CANCELLED`.

The Saga pattern is useful when a business workflow spans services and can tolerate coordinated, potentially eventual completion.

---

# 15. Data Consistency: The Transactional Outbox Pattern

## 15.1 The dual-write problem

Suppose the Orders service must save an order and publish an `OrderCreated` event.

A naive implementation might do this:

```js
await db.orders.insert(order);

await kafka.publish({
  type: "OrderCreated",
  orderId: order.id
});
```

Two independent systems are involved: the database and the message broker.

This creates two dangerous failure cases.

**Case A: Database succeeds, message publication fails**

```text
Order saved
    |
    v
Process crashes
    |
    v
Event never published
```

Other services never learn about the order.

**Case B: Message publication succeeds, database write fails**

```text
Event published
    |
    v
Database transaction fails
    |
    v
Consumers see an order that does not exist
```

This is known as the **dual-write problem**.

## 15.2 Definition of the Transactional Outbox pattern

The Transactional Outbox pattern stores the business change and the event record in the same local database transaction.

```text
                Order Service
                     |
                     v
              Database Transaction
                /            \
               v              v
         Orders Table     Outbox Table
               \              /
                \            /
                   COMMIT
                     |
                     v
               Outbox Relay
                     |
                     v
                Message Broker
                     |
                     v
                Event Consumer
```

Example schema:

```sql
CREATE TABLE outbox (
    event_id UUID PRIMARY KEY,
    event_type TEXT NOT NULL,
    aggregate_id TEXT NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    published_at TIMESTAMPTZ
);
```

Example transaction:

```sql
BEGIN;

INSERT INTO orders (id, user_id, status)
VALUES ('ord-103', 7, 'pending');

INSERT INTO outbox (
    event_id,
    event_type,
    aggregate_id,
    payload
)
VALUES (
    '00000000-0000-0000-0000-000000000103',
    'OrderCreated',
    'ord-103',
    '{"orderId":"ord-103","status":"pending"}'
);

COMMIT;
```

The example uses a fixed UUID for illustration; a production application should generate unique event identifiers.

Because both inserts belong to the same database transaction, either both commit or both roll back.

## 15.3 How does the event reach the broker?

An outbox relay reads unpublished records and publishes them to the broker.

Possible implementations include:

- A polling worker that reads the outbox table.
- Change data capture that reads database changes from a transaction log.
- A specialized connector that forwards committed outbox records.

The relay marks records as published or advances its processing position according to its implementation.

If the relay crashes after publishing an event but before recording that publication succeeded, it may publish the same event again.

Therefore, the Outbox pattern generally requires consumers to tolerate duplicate delivery.

## 15.4 Idempotent consumers

A consumer should avoid applying the same logical event twice.

One approach is to track processed event IDs:

```sql
CREATE TABLE processed_events (
    event_id UUID PRIMARY KEY,
    processed_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

A consumer can process the event and record its ID in the same local transaction as its business changes.

Conceptually:

```text
Receive Event
     |
     v
Begin Local Transaction
     |
     v
Check Event ID
     |
     +---- Already processed ----> No duplicate effect
     |
     v
Apply Business Change
     |
     v
Record Event ID
     |
     v
Commit
```

The precise implementation must handle concurrent deliveries and ensure the deduplication record and business effect are atomic.

## 15.5 What the Outbox pattern guarantees

The pattern solves a specific problem: ensuring that the intent to publish an event is committed atomically with the corresponding local business change.

It does not guarantee:

- Instant delivery.
- Exactly-once processing across the entire distributed system.
- Automatic consistency between two arbitrary databases.
- Successful processing by every consumer.
- Correct ordering without additional design.
- That a failed consumer will recover without monitoring and retries.

The usual outcome is reliable eventual propagation, subject to the relay, broker, and consumer recovery mechanisms.

## 15.6 Outbox during monolith-to-microservice migration

Suppose the monolith owns payment data initially, and a new service needs a reliable stream of payment changes.

The monolith can record its payment update and an outbox event in the same local transaction.

A relay publishes the event, and the new service updates its own database.

```text
Monolith Database
  ├── Payments Table
  └── Outbox Table
          |
          v
      Event Relay
          |
          v
      Message Broker
          |
          v
   Payment Service
          |
          v
   Payment Service DB
```

During this transition, the two databases may temporarily contain different states because event delivery is asynchronous.

The migration needs a defined source of truth, replay and backfill procedures, duplicate handling, and reconciliation checks.

Once the new service becomes authoritative, the monolith must stop making independent writes to the payment records it no longer owns.

---

# 16. A Practical Migration Walkthrough

This section combines the architectural concepts into a single example.

Assume LearnHub has a working monolith and wants to extract Payment.

## Phase 1: Establish a baseline

Before making changes, measure the existing system.

Record:

- Payment endpoint latency.
- Error rate.
- Payment success and decline rates.
- Database performance.
- External provider failure rate.
- Deployment frequency and recovery time.

Without a baseline, the team cannot determine whether the migration improved anything.

## Phase 2: Isolate payment logic

Refactor payment responsibilities behind an explicit interface in the monolith.

```js
async function createPayment(input) {
  // Validate payment request.
  // Enforce idempotency.
  // Call the payment provider.
  // Persist the payment state.
  // Return a defined result.
}
```

Ensure unrelated modules no longer manipulate payment internals directly.

This step improves the architecture even before a separate service exists.

## Phase 3: Define the external contract

For example:

```http
POST /payments
GET /payments/{paymentId}
POST /payments/{paymentId}/refunds
```

Define request schemas, error responses, authentication, idempotency, and payment-state semantics.

The monolith should call the new interface through a controlled adapter rather than depending on implementation details.

## Phase 4: Create and validate the new service

Deploy Payment independently with its own configuration, observability, database access, and integration tests.

Initially, production requests can continue to use the existing implementation.

Avoid introducing a second uncontrolled writer for payment data.

## Phase 5: Plan data ownership

Choose a cutover strategy based on the system's constraints.

A possible strategy is:

1. Backfill existing payment records.
2. Capture subsequent changes through a reliable event mechanism.
3. Validate that the new database matches the expected source state.
4. Establish a controlled write cutover.
5. Verify that only the intended owner can change authoritative payment records.

A backfill alone is insufficient if writes continue during the copy. The team needs a way to account for changes occurring while the migration is running.

## Phase 6: Introduce traffic gradually

Use a routing layer and a documented rollout plan.

For each stage, define:

- The eligible requests.
- The percentage or selection rule.
- The maximum acceptable error rate.
- Latency thresholds.
- Business correctness checks.
- The rollback trigger.

For financial operations, compare authoritative transaction records and business outcomes rather than relying only on HTTP status codes.

## Phase 7: Test failure scenarios

Deliberately test cases such as:

- Payment service unavailable.
- Database connection lost.
- Payment provider times out after processing the request.
- Duplicate request arrives.
- Relay publishes an event twice.
- Consumer is offline.
- Service restarts with pending operations.
- A rollback occurs while requests are in flight.

A migration is not production-ready merely because the happy path succeeds.

## Phase 8: Complete the ownership transfer

When the new implementation is stable:

1. Route all intended payment operations to the new service.
2. Stop old payment writes.
3. Verify that scheduled jobs and administrative operations use the new owner.
4. Reconcile pending payments and refunds.
5. Remove obsolete payment tables or access paths when safe.
6. Retire the monolith's old payment implementation.

The system should have one clear authority for each payment state.

## Phase 9: Evaluate the outcome

Compare the baseline with the new architecture.

Questions include:

- Did deployment become easier?
- Did payment latency improve or worsen?
- Did infrastructure cost increase?
- Is the service easier to operate?
- Are failures more contained?
- Did data consistency become harder to maintain?
- Was the added operational complexity justified?

An extraction is successful when it improves the relevant business or engineering outcome—not merely when a module runs in a different process.

---

# 17. Observability and Operational Readiness

Independent services create more places where failures can occur. Monitoring must cover the complete business operation, not just individual services.

## 17.1 The three pillars of observability

### Logs

Logs record events and contextual details.

Example:

```json
{
  "level": "error",
  "service": "payment-service",
  "operation": "authorize-payment",
  "orderId": "ord-104",
  "traceId": "trace-123",
  "message": "Payment provider timed out"
}
```

Production logs should avoid exposing payment credentials, secrets, or unnecessary sensitive data.

### Metrics

Metrics aggregate numeric measurements over time.

Useful examples include:

- Request rate.
- Error rate.
- Latency percentiles.
- CPU and memory utilization.
- Queue depth.
- Outbox backlog.
- Database connection saturation.
- Payment authorization success rate.

### Distributed traces

A trace follows a logical request through multiple services.

```text
Trace: Create Order
  ├── Orders API
  ├── Inventory API
  ├── Payment API
  └── Order Confirmation
```

Trace context or correlation identifiers help engineers determine where latency or failure occurred.

## 17.2 Health checks

A service can expose different health signals.

- **Liveness:** Is the process functioning sufficiently to continue running?
- **Readiness:** Is the instance prepared to receive the traffic assigned to it?

A readiness check should reflect the service's actual ability to handle requests. An overly strict dependency check can cause every replica to become unready during a temporary downstream outage, potentially worsening the incident.

## 17.3 Define service-level objectives

A service-level objective (SLO) is a measurable reliability target.

For example:

- 99.9% of eligible requests succeed over a defined measurement window.
- 95% of catalog requests complete within a specified latency.
- Payment outcomes are reconciled within a defined period.

The numbers should come from business requirements and operational evidence, not from arbitrary assumptions.

## 17.4 Operational readiness checklist

Before exposing a new service to production traffic, verify:

- [ ] Logs, metrics, and traces are available.
- [ ] Alerts identify actionable failures.
- [ ] Authentication and authorization are enforced.
- [ ] Timeouts and bounded retries are configured.
- [ ] Duplicate requests are handled safely.
- [ ] Database migrations and recovery procedures are documented.
- [ ] Backups and restore procedures are tested where applicable.
- [ ] Rollback criteria are defined.
- [ ] Dependency failure scenarios have been tested.
- [ ] The team knows how to investigate and recover from an incident.

---

# 18. Common Misconceptions

## Misconception 1: A monolith runs on one machine.

**Incorrect.** A monolith can have many replicas across multiple machines. The key distinction is that the application is deployed as one unit rather than as independently deployable business services.

## Misconception 2: Every microservice needs its own physical server.

**Incorrect.** Multiple services can share a machine, and each service can have multiple replicas. Logical deployment boundaries do not require a one-to-one mapping to physical hardware.

## Misconception 3: Microservices automatically solve single points of failure.

**Incorrect.** A service may have multiple replicas but still depend on one database, one external provider, or one critical broker. Reliability requires redundancy, suitable recovery procedures, and deliberate dependency design.

## Misconception 4: Microservices are always faster.

**Incorrect.** Network communication introduces latency and additional failure modes. Independent scaling and optimization can help under appropriate workloads, but they do not guarantee lower latency.

## Misconception 5: A microservice must use a different programming language.

**Incorrect.** Services can use the same language and framework. Polyglot development is an option, not a requirement.

## Misconception 6: Every module should become a microservice.

**Incorrect.** Splitting tightly related components can create unnecessary network calls, deployment overhead, and distributed transactions.

## Misconception 7: Database per service means one physical database server per service.

**Incorrect.** It means each service owns its data and controls access to it. Physical infrastructure can be shared.

## Misconception 8: The Saga pattern provides a distributed database rollback.

**Incorrect.** Sagas coordinate local transactions and compensating business actions. Compensation may be incomplete, delayed, or require manual resolution.

## Misconception 9: The Outbox pattern guarantees exactly-once delivery.

**Incorrect.** It reliably records the intent to publish alongside a local database change. Relays can publish duplicates, so consumers need idempotent processing.

## Misconception 10: Canary deployment and the Strangler Fig pattern are the same thing.

**Incorrect.** Canary deployment controls exposure to a new version. The Strangler Fig pattern gradually replaces legacy functionality. Canary deployment can be used during a Strangler migration.

## Misconception 11: Moving code to another repository completes a migration.

**Incorrect.** A complete migration must address routing, data ownership, communication, operations, and retirement of the old implementation.

## Misconception 12: A well-designed monolith is merely a temporary architecture.

**Incorrect.** A modular monolith can be a long-term architecture. Microservices are justified when their benefits outweigh their operational and distributed-systems costs.

---

# 19. When Should You Choose Each Architecture?

The right choice depends on actual requirements, not on which architecture is more fashionable.

## Choose a monolith when

- The product is new and requirements are changing quickly.
- The team is small.
- Modules have closely related business rules.
- A shared transaction model is useful.
- Deployment simplicity matters more than independent releases.
- Traffic can be handled efficiently by scaling the application.
- The team does not yet have the operational maturity to manage many services.

A modular monolith is often a strong default because it preserves simple deployment while establishing clear internal boundaries.

## Consider microservices when

- Some capabilities have substantially different scaling requirements.
- Teams need independently owned release cycles.
- Specific capabilities require stronger runtime or security isolation.
- Distinct business domains have stable boundaries.
- A module has operational requirements that differ significantly from the rest of the application.
- The organization can support service observability, deployment automation, data ownership, and incident response.

The decision should be supported by evidence such as deployment bottlenecks, measured resource contention, reliability requirements, or organizational constraints.

## A simple decision framework

```text
                 Start with requirements
                          |
                          v
             Are boundaries and ownership clear?
                     /             \
                   No               Yes
                   |                 |
                   v                 v
          Improve modularity    Is independent
          inside the monolith   deployment/scaling
                                materially valuable?
                                  /         \
                                No           Yes
                                |             |
                                v             v
                         Modular monolith  Evaluate
                                          microservices
```

This is a decision aid, not a universal rule. Security, regulatory constraints, organizational structure, and existing infrastructure may change the result.

---

# 20. Architecture Decision Checklist

Use this checklist before starting a migration.

## Business justification

- [ ] What measurable problem are we solving?
- [ ] Is the problem caused by architecture, implementation, or infrastructure?
- [ ] Can it be addressed through refactoring or better resource allocation?
- [ ] What outcome would justify the added complexity?

## Service boundaries

- [ ] Is the business responsibility clearly defined?
- [ ] Are dependencies understood?
- [ ] Is the service's data ownership explicit?
- [ ] Can the interface evolve without coordinated releases for every change?

## Data and consistency

- [ ] What is the authoritative source of truth?
- [ ] Which operations require atomicity?
- [ ] Can temporary inconsistency be tolerated?
- [ ] Are duplicate messages and requests handled safely?
- [ ] Is reconciliation available for ambiguous outcomes?

## Reliability

- [ ] What happens if the service is unavailable?
- [ ] Are timeouts and retry policies appropriate?
- [ ] Can failures cascade through dependencies?
- [ ] Are business-critical operations idempotent?
- [ ] Is there a tested recovery procedure?

## Migration and deployment

- [ ] Can the new implementation be tested without production traffic?
- [ ] Can traffic be routed gradually?
- [ ] Are rollback conditions explicit?
- [ ] Can old and new data coexist safely during the transition?
- [ ] Is there a clear condition for retiring the old implementation?

## Operations

- [ ] Can the team monitor the complete business workflow?
- [ ] Are deployment and database migrations automated?
- [ ] Are service ownership and incident responsibilities clear?
- [ ] Does the expected benefit justify the additional operating cost?

---
