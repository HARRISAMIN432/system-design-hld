# Chapter 7 — Scaling: Horizontal vs Vertical

> Personal study notes. Goal: understand why systems must scale, the two ways to do it, the trade-offs of each, and why real-world systems use a hybrid approach.

---

## Table of Contents

1. [What Is Scaling](#1-what-is-scaling)
2. [The Pizza Shop Analogy](#2-the-pizza-shop-analogy)
3. [Why Scaling Is Needed](#3-why-scaling-is-needed)
4. [Vertical Scaling (Scale Up)](#4-vertical-scaling-scale-up)
5. [Horizontal Scaling (Scale Out)](#5-horizontal-scaling-scale-out)
6. [Detailed Comparison](#6-detailed-comparison)
7. [Which One Do Real Systems Use](#7-which-one-do-real-systems-use)
8. [Horizontal Scaling Needs a Load Balancer](#8-horizontal-scaling-needs-a-load-balancer)
9. [Practical Considerations](#9-practical-considerations)
10. [Decision Guide](#10-decision-guide)
11. [Interview Questions](#11-interview-questions)
12. [Quick Revision Sheet](#12-quick-revision-sheet)

---

## 1. What Is Scaling

**Scaling** means increasing a system's capacity so it can handle more load (more users, more requests, more data) without degrading performance or failing.

There are two fundamental ways to scale:

| Name | Also called | Idea |
|---|---|---|
| **Vertical scaling** | Scale up | Make **one machine** more powerful |
| **Horizontal scaling** | Scale out | Add **more machines** |

---

## 2. The Pizza Shop Analogy

You own a pizza shop with **one chef**. The chef is slow and can prepare **only 2 pizza orders per hour**.

On weekends you receive **10 orders per hour**. The chef's maximum is 2, so you cannot serve everyone, and you cannot make customers wait forever. You have two options:

**Option 1: Hire one faster, stronger chef** who can make 10 pizzas per hour.
→ This is **vertical scaling**: you upgraded the capability of the single worker.

**Option 2: Hire 3–4 chefs.**
→ This is **horizontal scaling**: you increased the *number* of workers. Orders are distributed among them; for example, 15 incoming orders are split across chefs 1, 2, 3 and 4, and everything gets prepared on time.

---

## 3. Why Scaling Is Needed

A client sends requests to a server where our application is hosted. That server has finite resources:

- **RAM** (e.g. 16 GB)
- **CPU** (a certain number of cores, strong or weak)

| Users | Situation |
|---|---|
| 5 – 100 | One server easily handles it |
| 10,000 | Server may start struggling |
| 1,000,000 | One server almost certainly cannot cope |

As users grow, requests grow. If the server cannot cope, it slows down or crashes. Because everything depends on it, the server becomes a **single point of failure**.

We have two options: **increase the server's capacity**, or **increase the number of servers**.

---

## 4. Vertical Scaling (Scale Up)

**Increase the capacity of the existing server.**

You can do it two ways:

1. **Replace** the current server with a bigger, more powerful machine.
2. **Upgrade** the same server: more RAM, more or faster CPUs.

Example:

```
 Before:  16 GB RAM, 4 CPUs      ─►  After: 32 GB RAM, more powerful CPUs
 One machine, just bigger.
```

```
        ┌────────────┐              ┌──────────────────┐
 Client │  Server    │   scale up   │  Bigger Server   │
 ─────► │ 16GB, 4CPU │  ─────────►  │  32GB, 8 CPU     │
        └────────────┘              └──────────────────┘
```

**Pros**

- Simple: no change to application architecture.
- No load balancer required.
- Fast communication: everything happens inside one machine (**inter-process communication**).
- Data consistency is easy: there is one copy of the data/state.

**Cons**

- **Hardware has a limit.** You can only add so much RAM and CPU.
- **Single point of failure.** If that one machine crashes, everything is down.
- Often requires **downtime** to upgrade.
- Very large machines get disproportionately expensive.

---

## 5. Horizontal Scaling (Scale Out)

**Increase the number of servers** and host the application on each.

```
                      ┌────────────┐
                 ┌──► │  Server 1  │
 Clients ──► LB ─┼──► │  Server 2  │
                 └──► │  Server 3  │
                      └────────────┘
```

Different users' requests are distributed across the servers, so no single server carries all the load.

**Pros**

- **Resilient / fault tolerant.** If one or two servers crash or slow down, others keep serving traffic. No single point of failure.
- **Easy, practically unlimited scaling.** Need more capacity? Add more machines (4, 5, 50…).
- Can use many inexpensive commodity machines.
- Can scale without taking the system down.

**Cons**

- **Needs a load balancer** to distribute requests.
- **Data consistency becomes a challenge.** Multiple servers must see consistent data.
- **Slower communication.** Servers talk over the network (network calls / RPC), which is slower than in-process communication.
- More operational complexity (deployment, monitoring, state management).

---

## 6. Detailed Comparison

| Aspect | Horizontal scaling | Vertical scaling |
|---|---|---|
| **Number of machines** | Many (e.g. 3+) | One big machine |
| **Load balancer** | **Required** to route requests to instances | **Not needed** (single application server) |
| **Single point of failure** | **No.** Other instances take over | **Yes.** One crash takes everything down |
| **Resilience** | High | Low |
| **Data consistency** | Can become a bottleneck / challenge | Naturally consistent |
| **Scalability limit** | Easy to scale; add more machines | Limited by hardware (RAM/CPU ceiling) |
| **Communication** | Between servers via **network calls (RPC)**, slower | **Inter-process communication**, faster |
| **Speed (inter-component)** | Slower due to network | Faster |
| **Upgrade downtime** | Usually none | Often needed |
| **Cost curve** | Roughly linear (many small machines) | Rises steeply for large machines |

---

## 7. Which One Do Real Systems Use

**A hybrid approach.** Real-world systems take the best of both:

| Take from horizontal scaling | Take from vertical scaling |
|---|---|
| **Resilience** (no single point of failure) | **Data consistency** |
| **Easy scalability** | **Speed** (fast in-process communication) |

How this looks in practice:

- Run **multiple instances** of each service (horizontal) so failure of one doesn't matter and capacity can grow.
- Choose **reasonably powerful machines** for each instance (vertical) so each instance is efficient, and so that related work stays inside one process where possible, keeping latency low.
- Databases and stateful components are often scaled **vertically first** (simpler consistency), then **horizontally** (replication, sharding) when they outgrow one machine. Those techniques are covered in later chapters.

---

## 8. Horizontal Scaling Needs a Load Balancer

With several server instances (Server 1, 2, 3), thousands of users arrive. Something must decide *which instance gets which request*.

```
 Users ──► Load Balancer ──┬──► Server 1
                           ├──► Server 2
                           └──► Server 3
```

The load balancer:

- distributes requests across instances so **no single instance is overloaded**,
- keeps traffic away from unhealthy instances (health checks),
- uses an algorithm such as round robin or least connection (see Chapter 4).

In vertical scaling there is only one application server, so there is nothing to balance.

---

## 9. Practical Considerations

(Points that go slightly beyond the core comparison but matter in real designs.)

- **Make services stateless when you scale horizontally.** If a user's session is stored in the memory of Server 1, a request routed to Server 2 will not find it. Keep session state in a shared store (e.g. a cache or database) or use sticky routing deliberately.
- **Auto-scaling.** Cloud platforms can add and remove instances automatically based on CPU, memory or request count, so you pay for capacity only when you need it.
- **Scale the bottleneck, not everything.** Measure first. Scale only the component that is actually saturated.
- **Don't scale prematurely.** A to-do app with 100 users does not need ten servers.
- **Plan for the ceiling of vertical scaling.** Know the point at which a bigger machine stops being an option.

---

## 10. Decision Guide

| Situation | Lean toward |
|---|---|
| Early-stage product, low traffic | Vertical (keep it simple) |
| Need high availability / no downtime | Horizontal |
| Traffic is unpredictable and spiky | Horizontal with auto-scaling |
| Strong consistency needs on a single dataset | Vertical first, then careful horizontal strategies |
| Approaching hardware limits | Horizontal |
| Large, global, mature system | Hybrid |

---

## 11. Interview Questions

1. What is scaling? Why do we need it?
2. Explain vertical and horizontal scaling with an example.
3. What are the limits of vertical scaling?
4. Why does vertical scaling create a single point of failure?
5. Why does horizontal scaling need a load balancer?
6. Why is data consistency harder in horizontal scaling?
7. Why is communication faster in vertical scaling? (In-process vs network calls.)
8. Which approach do real systems use and why?
9. What must you do to your application before scaling it horizontally? (Make it stateless or externalize state.)
10. If a website is slow during a flash sale, how would you reason about which kind of scaling to apply?

---

## 12. Quick Revision Sheet

- **Vertical (scale up):** bigger machine. Simple, consistent, fast; but hardware-limited and a single point of failure.
- **Horizontal (scale out):** more machines. Resilient, easy to scale; but needs a **load balancer**, has **consistency** challenges and slower **network** communication.
- **Pizza analogy:** one faster chef vs several chefs.
- **Vertical communication:** inter-process (fast). **Horizontal communication:** network calls / RPC (slower).
- **Real world = hybrid:** take **resilience + scalability** from horizontal, **consistency + speed** from vertical.
- **Stateless services** make horizontal scaling much easier.

---

**Previous:** [Chapter 6 — Network Protocols](../Chapter-06-Network-Protocols/README.md) · **Back to start:** [Chapter 1 — Roadmap](../Chapter-01-Course-Roadmap/README.md)
