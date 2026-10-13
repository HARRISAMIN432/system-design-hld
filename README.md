# Chapter 1 — System Design Master Roadmap

> Personal study notes. This chapter is the map for the whole journey: what I will learn, in what order, and why that order matters.

---

## Table of Contents

1. [Why System Design](#1-why-system-design)
2. [Who This Is For](#2-who-this-is-for)
3. [How the Learning Is Structured](#3-how-the-learning-is-structured)
4. [Phase 1 — Foundational Concepts](#4-phase-1--foundational-concepts)
5. [Phase 2 — Networking, Performance and Scale](#5-phase-2--networking-performance-and-scale)
6. [Phase 3 — Distributed Systems and Data](#6-phase-3--distributed-systems-and-data)
7. [Phase 4 — Deployment, Security and Real-World Add-ons](#7-phase-4--deployment-security-and-real-world-add-ons)
8. [Capstone — Four Real-World Systems](#8-capstone--four-real-world-systems)
9. [HLD vs LLD](#9-hld-vs-lld)
10. [Suggested Study Schedule](#10-suggested-study-schedule)
11. [How to Get the Most Out of These Notes](#11-how-to-get-the-most-out-of-these-notes)
12. [Self-Check Questions](#12-self-check-questions)

---

## 1. Why System Design

System design is the discipline of deciding **how the pieces of a software product fit together so that it keeps working as it grows**. Writing code that works for one user is easy. Writing a system that works for ten million users, survives server crashes, stays fast across continents, and remains affordable is a different skill.

System design matters because it is:

| Context | Why it matters |
|---|---|
| **Interviews** | Large product companies (often called FAANG-style companies) test design thinking for mid and senior roles. |
| **Real engineering work** | Architecture decisions made early are expensive to reverse later. |
| **Building a startup** | Knowing when *not* to over-engineer is as valuable as knowing how to scale. |
| **Career growth** | Moving from "implement this ticket" to "decide how this should be built" requires design fluency. |

---

## 2. Who This Is For

- **College students** preparing for placements and early-career interviews.
- **Software engineers** preparing for senior-level interviews.
- **Developers** who want to build a real-world, scalable system, not just a tutorial app.
- **Founders** who need to make sound technical decisions for a new product.

**Prerequisites:** basic backend knowledge (HTTP, APIs, a database), and comfort reading code in JavaScript/Node.js (the capstone examples use it).

---

## 3. How the Learning Is Structured

The material alternates between **theory** and **applied systems**:

```
THEORY PHASES (4 weeks)                 APPLIED PHASE
┌────────────────────────────┐          ┌──────────────────────────────┐
│ Phase 1  Foundations       │          │ Build real systems with      │
│ Phase 2  Networking/Scale  │  ─────►  │ HLD + LLD concepts:          │
│ Phase 3  Distributed Sys   │          │  • Ticket booking            │
│ Phase 4  Deploy/Security   │          │  • E-commerce                │
└────────────────────────────┘          │  • Railway reservation       │
                                        │  • Music streaming           │
                                        └──────────────────────────────┘
```

A recommended companion activity: alongside the capstone coding, also study the **high-level architecture** of each system (how services talk, how data flows) *before* writing code. Knowing the big picture makes the code much easier to write.

---

## 4. Phase 1 — Foundational Concepts

**Goal:** understand the building blocks every large system is made of.

| Topic | Key question it answers |
|---|---|
| Monolith architecture | What does "everything in one codebase" look like and where does it hurt? |
| Microservice architecture | How do we split a system into independent services? |
| Monolith → Microservices migration | How do we migrate safely, step by step? |
| API Gateway | How does a client reach the right service without knowing the backend? |
| Load Balancer | How is traffic spread across many instances of one service? |
| API Gateway vs Load Balancer | Which one sits where, and which comes first? |
| Load balancer types (L4 / L7) | What can each "see" and decide with? |
| Load balancing algorithms | Round robin, weighted round robin, IP hash, least connection, weighted least connection, least response time |
| Proxy vs Reverse Proxy | Who is being hidden: the client or the server? |
| Reverse proxy vs Load balancer | Do we still need a load balancer if we have a reverse proxy? |

**Chapters in this phase:** 2, 3, 4, 5.

---

## 5. Phase 2 — Networking, Performance and Scale

**Goal:** understand how data moves and how to keep a system fast under growing load.

- **Networking protocols:** TCP vs UDP.
- **Web protocols:** HTTP, HTTPS, WebSockets, WebRTC and how they differ.
- **API styles:** REST APIs, gRPC and how to build APIs with gRPC.
- **CDN:** what a Content Delivery Network is and why a global product (like a streaming platform) needs one.
- **Caching:** what to cache, where to cache, and caching static data through a CDN.
- **Rate limiting:** how to protect the backend when a single client floods it with requests (for example, a user hitting refresh 20–25 times in a row).
- **Scaling from zero to a million users:** how a real system evolves.
- **Vertical vs horizontal scaling:** bigger machine vs more machines.

---

## 6. Phase 3 — Distributed Systems and Data

**Goal:** once a system spans many machines and many users, correctness becomes the hard part.

- **Distributed systems basics** and why they are needed at large scale.
- **CAP theorem:** Consistency, Availability, Partition tolerance and the trade-offs between them.
- **Database choice:** SQL vs NoSQL, and selecting a database based on requirements rather than habit.
- **Indexing:** how indexes make queries fast and what they cost.
- **Sharding vs partitioning:** what sharding is and how it is done.
- **Consistent hashing:** why it is needed and how it works.
- **Distributed transactions:** two-phase commit and three-phase commit.
- **Concurrency control:** optimistic and pessimistic locking. The classic scenario: two users try to book the *same* ticket at the *same* moment, and only one may succeed.
- **Event-driven architecture:** the basics.
- **Messaging queues:** why they exist and what problems they solve.

---

## 7. Phase 4 — Deployment, Security and Real-World Add-ons

**Goal:** turn a design into something deployable, secure and operable.

- **Docker and containerization:** why containers, and how to containerize each service.
- **EC2 vs AWS Lambda:** server-based vs serverless architecture, with a coding exercise.
- **Authentication and authorization:** proving identity vs controlling access.
- **Security:** end-to-end encryption using the public-key / private-key concept.
- **Cron jobs:** with millions of users and millions of records in a relational database such as MySQL, we cannot keep piling load on it. A scheduled job can move completed or unused records into a NoSQL store such as Cassandra.
- **Polling, long polling, webhooks:** different ways of getting updates from a server.
- **File uploads:** storing images and videos in object (blob) storage such as S3, and using **pre-signed URLs** so clients upload directly without passing files through the backend.

---

## 8. Capstone — Four Real-World Systems

After the theory, I build four systems end to end, writing code and applying the HLD and LLD concepts.

| System | Main engineering challenges |
|---|---|
| **Movie ticket booking** (BookMyShow-style) | Backend-heavy. Two users requesting the same seat at the same time. Database locks: optimistic and pessimistic. |
| **E-commerce platform** (Amazon-style) | Order placement flow, microservices communicating with each other, delivery and store/seller flows. |
| **Railway reservation** (IRCTC-style) | Concurrent bookings, seat allotment, partial bookings. |
| **Music streaming** (Spotify-style) | Storing audio in object storage, distributing it through a CDN. |

---

## 9. HLD vs LLD

| | High-Level Design (HLD) | Low-Level Design (LLD) |
|---|---|---|
| **Focus** | Components, services, data flow, infrastructure | Classes, modules, schemas, code structure |
| **Question** | "What are the big pieces and how do they talk?" | "How is each piece implemented?" |
| **Examples** | Load balancer, API gateway, queues, databases | Class diagrams, concurrency handling in code |

The capstone systems intentionally use **both**.

---

## 10. Suggested Study Schedule

| Week | Focus | Chapters / Topics |
|---|---|---|
| 1 | Foundations | Monolith vs microservices, gateway, load balancing, proxies |
| 2 | Networking and scaling | Protocols, REST/gRPC, CDN, caching, rate limiting, scaling |
| 3 | Distributed systems | CAP, databases, sharding, consistent hashing, transactions, queues |
| 4 | Deployment and add-ons | Docker, EC2 vs Lambda, auth, security, cron, uploads |
| 5+ | Capstone builds | The four systems above |

Adjust to your own pace. Depth beats speed.

---

## 11. How to Get the Most Out of These Notes

1. **Read, then redraw.** After each chapter, close the notes and redraw the main diagram from memory.
2. **Explain aloud.** If you cannot explain a concept to a friend in two minutes, you have not absorbed it yet.
3. **Always ask "what breaks?"** Every design choice has a failure mode. Learn them.
4. **Always ask "when is this overkill?"** Not every product needs microservices, a CDN, or sharding.
5. **Practice with interview questions.** Each chapter ends with questions in the style interviewers actually ask.
6. **Code along.** Reading about a monolith is not the same as running one and breaking it.

---

## 12. Self-Check Questions

1. Why is system design a different skill from writing correct code?
2. What is the difference between HLD and LLD? Give one example of each.
3. In what order should the four theory phases be learned, and why?
4. Name three problems that appear only when a system becomes distributed.
5. Why does the capstone include a seat-booking scenario?

---

**Next:** [Chapter 2 — Monolith vs Microservices](../Chapter-02-Monolith-vs-Microservices/README.md)
