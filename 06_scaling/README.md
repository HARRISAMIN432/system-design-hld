# Chapter 7 — Horizontal vs Vertical Scaling

> **Personal study notes**
>
> **Goal:** Understand why systems need scaling, how vertical and horizontal scaling work, their architectural trade-offs, and how production systems combine both approaches to achieve performance, reliability, and cost efficiency.

---

## Table of Contents

1. [What Is Scaling?](#1-what-is-scaling)
2. [Why Do Systems Need to Scale?](#2-why-do-systems-need-to-scale)
3. [The Pizza Shop Analogy](#3-the-pizza-shop-analogy)
4. [Vertical Scaling — Scale Up](#4-vertical-scaling--scale-up)
5. [Horizontal Scaling — Scale Out](#5-horizontal-scaling--scale-out)
6. [Detailed Comparison](#6-detailed-comparison)
7. [What Changes When We Scale?](#7-what-changes-when-we-scale)
8. [Why Load Balancing Matters](#8-why-load-balancing-matters)
9. [How Real-World Systems Scale](#9-how-real-world-systems-scale)
10. [Practical Considerations](#10-practical-considerations)
11. [Decision Guide](#11-decision-guide)
12. [Common Misconceptions](#12-common-misconceptions)
13. [Interview Questions and Answers](#13-interview-questions-and-answers)
14. [Quick Revision Sheet](#14-quick-revision-sheet)

---

## 1. What Is Scaling?

**Scaling is the process of increasing a system's capacity to handle additional workload while maintaining acceptable performance, availability, and reliability.**

Workload can increase in several ways:

- **More users:** More people use the application simultaneously.
- **More requests:** The application receives more HTTP requests, database queries, or background jobs.
- **More data:** The system stores and processes larger datasets.
- **More computationally expensive work:** Requests require more CPU, memory, or processing time.
- **Stricter performance requirements:** The system must handle the same workload with lower latency.

Two fundamental approaches are used to increase computational capacity.

| Approach | Also called | Core idea |
|---|---|---|
| **Vertical scaling** | Scale up | Increase the resources available to one machine. |
| **Horizontal scaling** | Scale out | Add more machines or application instances to share the workload. |

### 1.1 Vertical scaling at a glance

You have one application server with 4 CPU cores and 16 GB of RAM.

You upgrade it to 16 CPU cores and 64 GB of RAM.

The application still runs on one machine, but that machine has more resources.

```text
Before                         After

┌────────────────┐             ┌────────────────┐
│ Application    │             │ Application    │
│ Server         │   Scale up   │ Server         │
│                │ ──────────►  │                │
│ 4 CPU cores    │             │ 16 CPU cores   │
│ 16 GB RAM      │             │ 64 GB RAM      │
└────────────────┘             └────────────────┘

       One machine                  One machine
```

### 1.2 Horizontal scaling at a glance

Instead of making one machine bigger, you run the application on multiple machines.

```text
                         ┌─────────────────┐
                    ┌───►│ Application A   │
                    │    └─────────────────┘
┌────────────┐      │    ┌─────────────────┐
│   Clients  │─────►│ LB ├──► Application B│
└────────────┘      │    └─────────────────┘
                    │    ┌─────────────────┐
                    └───►│ Application C   │
                         └─────────────────┘

                  LB = Load Balancer
```

Each instance can process requests independently, assuming the application and its dependencies are designed appropriately.

### The key distinction

**Vertical scaling increases the capacity of a machine. Horizontal scaling increases the number of machines or instances participating in the system.**

Scaling is not limited to web servers. Databases, caches, message brokers, storage systems, and background workers can all be scaled using different strategies.

---

## 2. Why Do Systems Need to Scale?

Imagine an online shopping application.

Initially, it serves a few hundred requests per day. A single server handles the workload comfortably.

Then the business grows. Thousands of customers browse products, search the catalog, add items to carts, and place orders simultaneously.

The same infrastructure must now process substantially more work.

### 2.1 Every machine has finite resources

A server has limited resources, including:

- **CPU:** Executes application code and performs calculations.
- **RAM:** Holds application data, caches, and working memory.
- **Network bandwidth:** Determines how much network traffic the machine can handle.
- **Disk I/O:** Controls how quickly data can be read from or written to storage.
- **Connection capacity:** Limits may arise from sockets, file descriptors, database connections, or other operating-system and application constraints.

A server can become overloaded even when its CPU utilization is low. For example, it might be waiting on a slow database, running out of database connections, or exhausting available memory.

### 2.2 What happens when capacity is insufficient?

Consider an API receiving more requests than it can process.

```text
Incoming requests:       1,000 requests/second
Server capacity:           400 requests/second
                         ─────────────────────
Excess incoming load:      600 requests/second
```

This is an illustrative example, not a universal server capacity.

If incoming traffic consistently exceeds processing capacity, the system may experience:

1. Increasing request queues.
2. Higher response times.
3. Request timeouts.
4. Retries that add even more traffic.
5. Resource exhaustion.
6. Failed requests or service outages.

The exact outcome depends on queue limits, timeout policies, resource constraints, and how the application handles overload.

### 2.3 Capacity, latency, and throughput

These three concepts are related but different.

| Term | Meaning |
|---|---|
| **Throughput** | Amount of work completed per unit of time, such as requests per second. |
| **Latency** | Time required to complete one operation. |
| **Capacity** | Workload a component can sustain while meeting its performance and reliability requirements. |

Increasing capacity can improve throughput and reduce latency under load. However, adding servers does not guarantee lower latency for every request. Network communication, database bottlenecks, contention, and coordination can introduce additional delays.

### 2.4 Scaling and availability are different goals

Scaling primarily increases the system's ability to handle workload. Availability describes whether the system is operational and accessible when needed.

The two goals often overlap:

- More application instances can increase capacity.
- Multiple instances can also help the application survive an individual instance failure.
- However, if every instance depends on one failed database, the application may still become unavailable.

**Important:** Scaling out creates the possibility of greater resilience; it does not automatically create a highly available system.

---

## 3. The Pizza Shop Analogy

Imagine a pizza shop with one chef.

The chef can prepare two pizzas per hour, but the shop receives ten orders per hour.

The shop cannot sustainably process all incoming orders at its current capacity.

There are two fundamental options.

### Option A: Hire a faster chef

You replace the existing chef with someone or something capable of preparing ten pizzas per hour.

This resembles **vertical scaling**.

```text
Before                    After

┌──────────────┐          ┌──────────────┐
│ One chef     │          │ One chef     │
│ 2 pizzas/hr  │ ───────► │ 10 pizzas/hr │
└──────────────┘          └──────────────┘
```

You increase the capability of one worker.

### Option B: Hire more chefs

You hire additional chefs, each capable of preparing two pizzas per hour.

```text
                  ┌─────────────────┐
                  │ Incoming orders │
                  └────────┬────────┘
                           │
                     Order assignment
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
          ┌────────┐   ┌────────┐   ┌────────┐
          │ Chef 1 │   │ Chef 2 │   │ Chef 3 │
          │ 2/hr   │   │ 2/hr   │   │ 2/hr   │
          └────────┘   └────────┘   └────────┘
```

This resembles **horizontal scaling**.

The shop now has a theoretical combined preparation capacity of six pizzas per hour, assuming the chefs can work independently and have sufficient ingredients, ovens, and workspace.

### What does this teach us?

| Pizza shop | Software system |
|---|---|
| One chef | One application server |
| Faster chef | More powerful server |
| Additional chefs | Additional application instances |
| Assigning orders | Load balancing or work distribution |
| Shared ingredients | Shared databases, caches, and storage |
| Oven or ingredient shortage | A downstream bottleneck |

The analogy also reveals an important limitation: hiring more chefs does not help much if everyone must wait for the same oven.

Similarly, adding application servers will not solve a database bottleneck unless the database or the way it is accessed is addressed.

---

## 4. Vertical Scaling — Scale Up

**Vertical scaling means increasing the computational resources available to an existing machine or replacing it with a more powerful machine.**

It can involve upgrading:

- CPU cores or CPU performance.
- RAM.
- Storage performance.
- Network capacity.
- Other hardware or resource limits.

For virtual machines and cloud instances, vertical scaling often means changing the instance type to one with more allocated resources.

### 4.1 Example

Suppose a backend API runs on one server.

| Resource | Before | After |
|---|---:|---:|
| CPU cores | 4 | 16 |
| RAM | 16 GB | 64 GB |
| Application instances | 1 | 1 |

The machine has four times the CPU cores and four times the RAM. That does **not** necessarily mean the application becomes four times faster: performance depends on the workload, software architecture, resource contention, and whether the application can use the additional resources.

### 4.2 Advantages

**1. Simpler architecture**

You can often increase capacity without introducing additional application instances, inter-instance coordination, or a new load-balancing layer.

**2. Easier state management**

If the application stores state locally, there is no immediate requirement to distribute that state across multiple application machines.

**3. Fewer distributed-system concerns**

You may avoid introducing network communication and coordination between application instances.

**4. Potentially lower operational overhead**

Managing one larger server may be easier than operating many servers, depending on the platform and deployment environment.

**5. Useful for stateful workloads**

Some databases and legacy applications are easier to scale vertically because they rely on a single primary instance or have limited support for distributed operation.

### 4.3 Disadvantages

**1. Hardware limits**

A machine cannot grow indefinitely. CPU, memory, storage, and network capacity all have practical upper bounds.

**2. Potentially expensive upgrades**

The largest machine sizes may cost disproportionately more than smaller ones with comparable aggregate resources.

**3. Failure concentration**

If the application depends entirely on one machine, a machine failure can make that application unavailable.

**4. Upgrade and maintenance considerations**

Changing instance types or physical hardware may require a restart, migration, or maintenance window. Some cloud upgrades can be performed with limited downtime, but this is platform-dependent.

**5. Diminishing returns**

More resources help only when the workload can use them. If the application is blocked by a database, external API, or single-threaded execution path, adding CPU cores may have little effect.

### 4.4 When is vertical scaling useful?

Vertical scaling is often a good first step when:

- The application is small or moderately sized.
- Simplicity matters more than distributed-system complexity.
- The workload benefits from additional memory or CPU.
- The database is approaching resource limits and supports a larger instance.
- Measurements show that a larger machine will address the actual bottleneck.

Vertical scaling is not inherently a poor design. It is a practical engineering choice with a finite ceiling.

---

## 5. Horizontal Scaling — Scale Out

**Horizontal scaling means increasing capacity by adding more machines, virtual machines, containers, or application instances that can perform work in parallel.**

Rather than relying on one increasingly powerful machine, the system distributes workload across multiple instances.

### 5.1 Example

Suppose one application instance can sustainably handle 300 requests per second while meeting its latency target.

You deploy four equivalent instances.

```text
                         ┌─────────────────┐
                    ┌───►│ Instance 1      │
                    │    └─────────────────┘
┌────────────┐      │    ┌─────────────────┐
│   Clients  │─────►│ LB ├──► Instance 2   │
└────────────┘      │    └─────────────────┘
                    │    ┌─────────────────┐
                    └───►│ Instance 3      │
                         └─────────────────┘
                         ┌─────────────────┐
                         │ Instance 4      │
                         └─────────────────┘
```

With ideal workload distribution and no other bottlenecks, four instances could theoretically handle approximately 1,200 requests per second.

In practice, capacity is often lower because of:

- Load-balancing overhead.
- Uneven request costs.
- Shared database limitations.
- Cache contention.
- Network overhead.
- Resource coordination.
- Background work and other shared dependencies.

The capacity of a system is determined by its bottlenecks, not just the sum of its application-server capacities.

### 5.2 Advantages

**1. Greater aggregate capacity**

Multiple instances can process independent requests concurrently.

**2. A path to fault tolerance**

If instances are independent and the system routes traffic away from unhealthy instances, the remaining instances can continue serving requests.

**3. Flexible capacity changes**

Instances can often be added or removed without replacing the entire application server.

**4. Better use of commodity infrastructure**

A fleet of moderately sized machines may offer a better cost-performance balance than one very large machine.

**5. Support for elastic workloads**

Cloud infrastructure can add or remove instances in response to workload changes.

### 5.3 Disadvantages

**1. More operational complexity**

The system must deploy, configure, monitor, and maintain multiple instances.

**2. State management becomes more important**

An instance's local memory is not automatically shared with other instances.

**3. Shared dependencies may become bottlenecks**

A large application fleet can overload a database, cache, external API, or message broker.

**4. Network communication and coordination**

Distributed components communicate over networks, introducing latency and possible network failures.

**5. Fault tolerance requires deliberate design**

Multiple application instances do not protect against failures of shared infrastructure or common dependencies.

**6. Costs do not necessarily scale linearly**

Additional instances may require more database capacity, networking, observability, orchestration, and operational effort.

### 5.4 When is horizontal scaling useful?

It is particularly useful when:

- Traffic exceeds what a single machine can handle economically.
- High availability is required.
- Traffic fluctuates substantially.
- The application can process many requests independently.
- The infrastructure supports automated provisioning and health monitoring.

However, the application must be designed so that adding instances actually adds useful capacity.

---

## 6. Detailed Comparison

| Aspect | Vertical scaling | Horizontal scaling |
|---|---|---|
| Basic idea | Increase resources per machine | Add more machines or instances |
| Machine count | Usually unchanged | Increases |
| Capacity ceiling | Limited by machine size | Can grow across a fleet, subject to system limits |
| Architecture | Often simpler | Usually more complex |
| State management | Local state can remain local | Shared or distributed state often needs explicit handling |
| Communication | Can remain within one machine | Network communication may be required |
| Failure handling | One-machine dependency may create a single point of failure | Can tolerate individual instance failures if designed correctly |
| Load balancing | Not needed for one application instance | Commonly used, but not mandatory in every topology |
| Upgrade process | May require resizing, restart, or migration | Instances can often be replaced gradually |
| Cost | Larger machines may be expensive | Multiple machines and operational overhead add costs |
| Best fit | Simpler workloads and resource-bound machines | Parallel workloads, elastic capacity, and resilient application fleets |

### 6.1 Is horizontal scaling always more reliable?

No.

Consider these two architectures:

```text
Architecture A: One application instance

Clients ──► Application ──► Database
```

If the application instance fails, requests cannot be served by that instance.

Now consider:

```text
Architecture B: Multiple application instances

                    ┌──► Application A ──┐
Clients ──► Router ─┤                     ├──► Database
                    └──► Application B ──┘
```

If Application A fails, the router can send traffic to Application B.

But what happens if the database fails?

Both application instances may become unable to serve requests.

The second architecture improves resilience to an individual application-instance failure, but it still depends on the availability of the database, router, network, and other required components.

**Reliability depends on the failure domains and recovery mechanisms of the entire system.**

### 6.2 Is horizontal scaling always cheaper?

No.

For a low-traffic application, one appropriately sized server may be significantly cheaper and easier to operate.

At higher scale, several smaller machines may provide a better price-performance ratio. The result depends on hardware pricing, utilization, licensing, network costs, redundancy requirements, and operational overhead.

Choose based on measured workload and total cost, not on the number of servers alone.

---

## 7. What Changes When We Scale?

Understanding scaling requires looking beyond the number of machines.

### 7.1 Application state

Suppose a user logs in to Instance A. The application stores the session in that instance's local memory.

The next request reaches Instance B.

```text
Request 1:
Client ──► Instance A
             │
             └── Session stored in local memory

Request 2:
Client ──► Instance B
             │
             └── Session not found
```

The user may appear logged out because Instance B does not have Instance A's local session.

This is one reason horizontally scaled applications often externalize session state.

For example:

```text
                  ┌──► Instance A ──┐
Client ──► Router ├──► Instance B ──┼──► Shared Session Store
                  └──► Instance C ──┘
```

The session store could be a database, a distributed cache, or another suitable state-management system.

Another option is signed, self-contained tokens, where appropriate. These reduce the need to retrieve a session for every request, but introduce their own expiration, revocation, and security considerations.

Sticky sessions can also direct a client to the same instance, but they do not automatically provide state replication or reliable recovery after that instance fails.

### 7.2 Statelessness

A service is **stateless with respect to requests** when it does not depend on retaining request-specific session state exclusively in a particular application instance between requests.

A stateless service can usually process a request on any suitable instance, assuming the required data and dependencies are accessible.

This makes horizontal scaling easier because instances can be added or removed with fewer coordination requirements.

Stateless does not mean the application has no data. It means state is managed in a way that does not require every future request to return to the same process.

### 7.3 Data consistency

Vertical scaling does not automatically guarantee consistency. An application running on one machine can still contain race conditions, transaction errors, or stale caches.

Horizontal scaling adds additional consistency challenges when multiple instances share or replicate state.

For example:

- Instance A updates a product's inventory.
- Instance B reads a copy of that inventory.
- If the read comes from a lagging replica or stale cache, Instance B may see an older value.

The solution depends on the required consistency model, database architecture, transaction boundaries, caching strategy, and replication design.

A single primary database can serve many application instances while maintaining transactional guarantees for supported operations. Horizontal scaling does not necessarily require abandoning strong consistency.

### 7.4 Network communication

Communication within a process is usually cheaper than a network request because network communication involves additional protocol handling, serialization, scheduling, and potential transmission delays.

However, **vertical scaling does not automatically mean all communication is inter-process communication**. Two processes on the same machine may communicate through sockets or other mechanisms, while one process may perform work across multiple CPU cores.

Similarly, horizontal scaling does not mean every request must call another server. Each application instance can perform substantial work locally.

The important distinction is whether an operation requires communication across a process, machine, or network boundary.

### 7.5 Bottlenecks and Amdahl's law

Adding machines helps only when additional resources can perform useful work.

Imagine an application where most requests can be processed independently, but every request must also access a single database that has reached its maximum throughput.

Adding ten more application instances may increase database contention without increasing overall throughput.

This illustrates a fundamental rule:

> Scaling one component cannot eliminate a bottleneck in another component.

A useful related concept is **Amdahl's law**: the maximum speedup of a workload is limited by the portion that cannot be parallelized.

In practice, profile the system, identify the constrained resource, and estimate how much additional capacity the proposed change can actually deliver.

---

## 8. Why Load Balancing Matters

When an application has multiple instances, traffic must reach an appropriate instance.

A load balancer is a common solution.

```text
                         ┌────────────────┐
                         │ Application A  │
                    ┌───►└────────────────┘
                    │    ┌────────────────┐
┌────────────┐      │    │ Application B  │
│   Clients  │─────► Load └────────────────┘
└────────────┘      │ Balancer
                    │    ┌────────────────┐
                    └───►│ Application C  │
                         └────────────────┘
```

A load balancer may:

- Distribute requests among instances.
- Detect unhealthy instances using health checks.
- Stop routing traffic to instances that should not receive requests.
- Support connection management, TLS termination, and other features depending on the implementation.

Common algorithms include round robin, weighted round robin, and least connections. The appropriate algorithm depends on traffic patterns, connection behavior, request costs, and system requirements.

### 8.1 Is a load balancer mandatory for horizontal scaling?

Not always.

A load balancer is common for horizontally scaled web applications, but other systems distribute work differently.

Examples include:

- DNS-based routing.
- Client-side load balancing.
- Message queues that distribute jobs among workers.
- Partitioning work by key or data range.
- Orchestrators and service-discovery systems that direct traffic.
- Peer-to-peer or decentralized architectures.

The key requirement is not necessarily a dedicated load-balancer product. It is a mechanism for assigning work to the available instances.

### 8.2 What if the load balancer fails?

If all application traffic depends on one load-balancer instance, that instance can become a single point of failure.

Production systems can address this with redundant load balancers, managed highly available services, health checks, and resilient network routing.

**Adding application instances is not enough if the component in front of them remains a single point of failure.**

---

## 9. How Real-World Systems Scale

Most production architectures use a combination of vertical and horizontal scaling.

This is often called a **hybrid approach**.

```text
                         ┌───────────────────────┐
                         │        Clients        │
                         └───────────┬───────────┘
                                     │
                         ┌───────────▼───────────┐
                         │ Load Balancer / Edge  │
                         └───────────┬───────────┘
                                     │
                   ┌─────────────────┼─────────────────┐
                   ▼                 ▼                 ▼
             ┌───────────┐     ┌───────────┐     ┌───────────┐
             │ App A     │     │ App B     │     │ App C     │
             │ 4 CPU     │     │ 4 CPU     │     │ 4 CPU     │
             │ 16 GB RAM │     │ 16 GB RAM │     │ 16 GB RAM │
             └─────┬─────┘     └─────┬─────┘     └─────┬─────┘
                   └─────────────────┼─────────────────┘
                                     │
                           ┌─────────▼─────────┐
                           │ Database / Cache  │
                           └───────────────────┘
```

This is a simplified architecture. The database and cache may themselves require replication, partitioning, or other scaling strategies.

The architecture combines:

- **Horizontal scaling:** Multiple application instances share incoming workload.
- **Vertical scaling:** Each instance has sufficient CPU and memory to perform its work efficiently.
- **Shared infrastructure:** Instances access data and other dependencies through suitable services.
- **Load balancing:** Requests are directed to eligible instances.

### 9.1 Why not use only horizontal scaling?

Using many extremely small instances can introduce unnecessary scheduling, networking, and coordination overhead.

A reasonably sized instance may process its assigned workload more efficiently.

### 9.2 Why not use only vertical scaling?

Even a very powerful machine has finite resources and may remain a single point of failure.

Multiple instances can offer additional capacity and a path to surviving individual instance failures.

### 9.3 How databases fit into the picture

Databases require special consideration because they manage shared state.

A common progression is:

1. Optimize queries, indexes, schemas, and connection management.
2. Increase the database machine's resources if the workload benefits from it.
3. Add read replicas when read traffic and consistency requirements make them appropriate.
4. Partition or shard data when a single database node cannot handle the required workload.
5. Introduce additional coordination and operational mechanisms as needed.

This is not a universal sequence. Some systems begin with managed distributed databases or partitioned storage, while others remain comfortably on one database machine for years.

**Database scaling is not simply a choice between vertical and horizontal scaling.** Replication, partitioning, caching, query optimization, and workload isolation address different bottlenecks.

### 9.4 Auto-scaling

Cloud environments can automatically adjust the number of running instances based on metrics such as CPU utilization, request rate, queue depth, or response latency.

For example:

```text
Low traffic
    │
    ▼
2 application instances
    │
    │ Traffic increases
    ▼
Auto-scaler adds instances
    │
    ▼
6 application instances
    │
    │ Traffic decreases
    ▼
Instances are removed
```

Auto-scaling is a mechanism for adjusting capacity, not a guarantee of instant or unlimited capacity.

Scaling takes time. The system needs sufficient baseline capacity, sensible scaling thresholds, startup procedures, and safeguards against removing instances that are still handling work.

---

## 10. Practical Considerations

### 10.1 Measure before scaling

Do not assume the application server is the bottleneck.

Measure:

- CPU and memory utilization.
- Request rate and latency percentiles.
- Error rates and timeout rates.
- Database query latency and connection-pool usage.
- Disk and network throughput.
- Queue depth and processing lag.
- Cache hit rate.
- External dependency latency.

For example, if CPU usage is low but requests wait several seconds for a database query, adding application servers may make the situation worse.

### 10.2 Scale the bottleneck

Consider an application with this request path:

```text
Client
  │
  ▼
Load Balancer
  │
  ▼
Application Servers
  │
  ▼
Database
```

If the application servers have spare capacity but the database is saturated, scaling only the application tier will not solve the main problem.

The next step is to investigate the database bottleneck and choose an appropriate fix.

### 10.3 Account for shared dependencies

Horizontally scaling the application increases the potential number of concurrent database connections.

If each of 20 application instances opens 100 database connections, the application fleet may attempt to use up to 2,000 connections.

That can overwhelm a database that was previously comfortable with a much smaller connection count.

Connection pooling, concurrency limits, workload management, and database capacity planning become increasingly important.

### 10.4 Handle graceful shutdown

When removing an application instance, stop sending it new requests and allow in-flight requests or jobs to finish when possible.

Otherwise, an auto-scaler or deployment system may interrupt active work.

For background workers, acknowledge messages and handle retries carefully so that interrupted jobs are not lost or processed incorrectly.

### 10.5 Avoid premature complexity

A small application does not automatically need a fleet of servers, a complex orchestration platform, or distributed database architecture.

Start with the simplest design that meets current requirements, instrument it, and introduce complexity when measurements or reliability requirements justify it.

### 10.6 Design for failure

If availability matters, consider:

- Multiple application instances.
- Redundant routing and load-balancing infrastructure.
- Database availability and recovery.
- Health checks and graceful degradation.
- Backups and tested restoration procedures.
- Timeouts, bounded retries, and circuit breakers where appropriate.
- Capacity remaining after an instance or failure domain becomes unavailable.

The goal is not to eliminate every failure. It is to prevent individual failures from causing unacceptable system-wide impact.

---

## 11. Decision Guide

| Situation | Reasonable starting point |
|---|---|
| Small application with predictable traffic | Vertical scaling, if the existing machine is the bottleneck |
| Application requires additional CPU or RAM | Vertical scaling when the workload can use those resources |
| Traffic exceeds one machine's economical capacity | Horizontal scaling |
| Individual application-instance failures must be tolerated | Multiple instances plus health checks and resilient routing |
| Traffic varies significantly throughout the day | Horizontal scaling with carefully configured auto-scaling |
| Database is CPU- or memory-bound | Optimize first; consider vertical database scaling |
| Database reads dominate the workload | Consider caching or read replicas, subject to consistency requirements |
| Dataset exceeds one node's practical limits | Consider partitioning, sharding, or a distributed storage design |
| Legacy application tightly coupled to local state | Vertical scaling may be simpler initially; redesign state handling before scaling out |
| Mature system with demanding performance and availability requirements | A measured hybrid architecture |

### A practical decision process

Ask these questions in order:

1. **What is the bottleneck?** CPU, RAM, database, network, disk, locks, or something else?
2. **Can optimization solve it?** Query tuning, caching, indexes, or better algorithms may be cheaper than adding capacity.
3. **Can one larger machine solve it economically?** If yes, vertical scaling may be the simplest option.
4. **Do we need more capacity or better failure tolerance?** Multiple instances may address both, with the right architecture.
5. **Can the workload be distributed safely?** Consider state, consistency, shared dependencies, and work distribution.
6. **How will we verify success?** Use load tests, latency targets, error rates, and operational metrics.

---

## 12. Common Misconceptions

### Misconception 1: Horizontal scaling means unlimited capacity

**Reality:** Capacity is still constrained by shared dependencies, coordination overhead, network limits, cost, and the amount of work that can be parallelized.

### Misconception 2: Horizontal scaling automatically removes single points of failure

**Reality:** A system can have ten application instances and still depend on one database or one failed routing component. Redundancy must cover the relevant failure domains.

### Misconception 3: A load balancer is always required

**Reality:** A load balancer is common, but other mechanisms can distribute work. The design needs an appropriate routing or work-assignment mechanism.

### Misconception 4: Vertical scaling guarantees consistency

**Reality:** Consistency depends on data-management and concurrency semantics. A single machine can still have race conditions, stale caches, or incorrect transactions.

### Misconception 5: Vertical scaling is always faster

**Reality:** A larger machine can reduce certain overheads, but performance depends on the workload. A distributed system can sometimes achieve lower overall completion times through parallel execution.

### Misconception 6: Stateless applications do not store data

**Reality:** Stateless application instances can use databases, caches, and external services. Statelessness means requests do not depend on session state being held exclusively by one particular application process.

### Misconception 7: Adding servers always improves performance

**Reality:** Additional instances may overload a shared database or increase contention. Identify the bottleneck before adding capacity.

### Misconception 8: Real systems must choose one approach

**Reality:** Many systems use moderately sized machines and multiple instances, combining vertical and horizontal scaling.

---

## 13. Interview Questions and Answers

### Q1. What is scaling, and why is it necessary?

Scaling increases a system's capacity to handle additional workload while maintaining acceptable performance and reliability. It becomes necessary when request volume, data size, or computational demand approaches the capacity of existing resources.

### Q2. What is the difference between vertical and horizontal scaling?

Vertical scaling increases the resources of an individual machine. Horizontal scaling adds more machines or instances to distribute work. Vertical scaling is often simpler, while horizontal scaling can provide greater aggregate capacity and resilience when designed correctly.

### Q3. Why does vertical scaling have a limit?

Every machine has finite CPU, memory, storage, and network capacity. Beyond a certain point, upgrades may be unavailable, uneconomical, or insufficient to meet the workload's requirements.

### Q4. Why can horizontal scaling improve availability?

If multiple independent application instances serve traffic and routing avoids unhealthy instances, one instance can fail without necessarily making the application unavailable. Shared dependencies and routing infrastructure must also be resilient.

### Q5. Why can horizontal scaling make state management harder?

Each application instance has its own local memory. A request routed to a different instance cannot automatically access state stored only in another instance's memory. External state stores, self-contained tokens, or carefully designed affinity mechanisms can address this issue.

### Q6. Does horizontal scaling require a load balancer?

No. A load balancer is a common mechanism, but client-side routing, DNS-based routing, message queues, partitioned work, and other systems can distribute workload.

### Q7. Why is communication across machines generally more expensive?

Network communication introduces transmission delays, protocol processing, serialization, and possible network failures. However, actual latency depends on the communication mechanism, workload, and infrastructure.

### Q8. Can horizontal scaling solve a database bottleneck?

Not by itself. Adding application instances may increase database traffic. The database workload must be investigated and addressed through appropriate optimization, capacity upgrades, caching, replication, partitioning, or other suitable techniques.

### Q9. What is the difference between scaling and high availability?

Scaling increases capacity. High availability aims to keep a service operational despite failures. Horizontal scaling can contribute to high availability, but only when routing, dependencies, recovery mechanisms, and failure domains are designed appropriately.

### Q10. What is auto-scaling?

Auto-scaling automatically adjusts the number or size of resources in response to workload or operational metrics. It helps match capacity to demand but requires appropriate thresholds, startup time, baseline capacity, and safeguards.

### Q11. A website becomes slow during a flash sale. Would you scale vertically or horizontally?

First, inspect metrics to identify the bottleneck. If the application server is CPU-bound and a larger machine would solve the problem, vertical scaling may help. If requests can be distributed across independent instances, horizontal scaling may provide more capacity. If the database or another shared dependency is saturated, that component also needs attention.

### Q12. How would you design a horizontally scalable web application?

Run multiple application instances behind a suitable routing mechanism, avoid relying exclusively on local session state, use shared or appropriately partitioned data services, monitor bottlenecks, implement health checks, and ensure required dependencies have sufficient capacity and appropriate availability.

### Q13. Does adding four servers guarantee four times the throughput?

No. Four servers may provide close to four times the throughput for highly parallel workloads under ideal conditions, but shared bottlenecks, unequal work distribution, coordination, and overhead reduce the actual gain.

### Q14. Which approach do production systems use?

Many production systems use a hybrid approach: vertical scaling provides suitable resources to each instance, while horizontal scaling distributes work and helps tolerate individual instance failures. The exact design depends on cost, workload, consistency, and availability requirements.

---

## 14. Quick Revision Sheet

### Core definitions

- **Scaling:** Increasing capacity to handle more workload.
- **Vertical scaling / scale up:** Give one machine more resources.
- **Horizontal scaling / scale out:** Add more machines or instances.
- **Load balancing:** Distribute traffic among eligible destinations.
- **Auto-scaling:** Adjust resources automatically as demand changes.
- **Stateless application instance:** Does not rely on retaining request-specific state exclusively in its own memory between requests.

### Remember the trade-offs

| Vertical scaling | Horizontal scaling |
|---|---|
| Simpler architecture | More operational complexity |
| Larger machine | More instances |
| Finite machine capacity | Aggregate capacity can grow across a fleet |
| Local state may be easier to manage | State must be designed for multiple instances |
| A single-machine dependency can be a failure point | Can tolerate individual instance failures when designed correctly |

### Five rules to remember

1. **Measure before scaling.** Identify the actual bottleneck.
2. **Scale the bottleneck.** More application servers will not automatically fix a saturated database.
3. **Design state management deliberately.** Local sessions can make horizontal scaling difficult.
4. **Separate capacity from availability.** More instances do not guarantee high availability.
5. **Use the simplest architecture that meets requirements.** Real systems often combine vertical and horizontal scaling.

### Final mental model

Think of a pizza shop:

- A more capable chef represents vertical scaling.
- Additional chefs represent horizontal scaling.
- Assigning orders represents load balancing.
- A shared oven represents a shared bottleneck.
- Making sure the shop can operate when one chef is absent represents fault tolerance.

The central lesson is that **scaling is not simply about adding CPU cores or servers. It is about increasing useful system capacity without allowing bottlenecks, coordination overhead, or failures to undermine the result.**

---