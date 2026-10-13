# Chapter 4 — Load Balancing: Types and Algorithms

> Personal study notes. Goal: understand what a load balancer actually "sees", why L4 and L7 behave differently, and be able to simulate every major algorithm by hand.

---

## Table of Contents

1. [Why Load Balancing](#1-why-load-balancing)
2. [Load Balancer Types: L4 vs L7](#2-load-balancer-types-l4-vs-l7)
3. [L4 Load Balancer in Depth](#3-l4-load-balancer-in-depth)
4. [L7 Load Balancer in Depth](#4-l7-load-balancer-in-depth)
5. [L4 vs L7 Comparison](#5-l4-vs-l7-comparison)
6. [Load Balancing Algorithms Overview](#6-load-balancing-algorithms-overview)
7. [Static Algorithms](#7-static-algorithms)
8. [Dynamic Algorithms](#8-dynamic-algorithms)
9. [Choosing an Algorithm](#9-choosing-an-algorithm)
10. [Interview Questions](#10-interview-questions)
11. [Quick Revision Sheet](#11-quick-revision-sheet)

---

## 1. Why Load Balancing

Take a chat application. Users A and B cannot message each other directly; they need a **chat server** in between. When traffic grows, one chat server is not enough, so we **scale horizontally** and run multiple instances (Chat-1, Chat-2, Chat-3), each on its own server with its own IP.

The question: **when User A sends message M1, which chat instance should receive it?**

That decision is the load balancer's job. It:

- Distributes traffic across instances.
- Runs **health checks** (which instances are alive?).
- Observes **load** (which instances are busy, which have capacity?).
- Routes each request using these inputs and an algorithm.

(Message delivery between users in the chat example uses WebSockets, which is a separate topic.)

---

## 2. Load Balancer Types: L4 vs L7

The OSI model has seven layers:

| Layer | Name |
|---|---|
| 7 | Application |
| 6 | Presentation |
| 5 | Session |
| **4** | **Transport** (TCP / UDP) |
| 3 | Network |
| 2 | Data link |
| 1 | Physical |

The naming is literal:

- **L4 load balancer** works at the **Transport layer** (TCP/UDP).
- **L7 load balancer** works at the **Application layer** (HTTP, HTTPS, WebSockets).

What a component can do depends on what it can *see* at its layer.

---

## 3. L4 Load Balancer in Depth

An L4 load balancer works with TCP and UDP. It **cannot understand**:

- HTTP requests
- HTTP headers
- Cookies
- URL paths (e.g. `/api/v1/order`)

### What it uses to decide

- Source IP address and port (the client)
- Destination IP address and port (the server)
- Protocol (TCP or UDP)

### Worked example

Setup: a chat service scaled to three instances on three servers (Server 1, 2, 3), each running the service on port 5000. A client has an IP and an OS-assigned port.

1. The client opens the app and tries to establish a connection with the chat service.
2. The L4 load balancer looks at the client IP/port, checks which instances are healthy and available, and picks one (say Server 2).
3. It establishes a **TCP connection** between the client and Server 2.
4. The request may be "create room", "send message" or "join room". The L4 balancer **has no idea** which one. It just forwards packets.
5. The chat service itself interprets the request.

### Connection stickiness

After the TCP connection is established, **every message from that client goes to the same server** until the connection is closed. For a long-lived connection this means all that user's traffic stays on Server 2.

### Analogy

An L4 load balancer is like a **post-office clerk who only reads the "From" and "To" addresses on the envelope** and never opens it.

### Strengths and weaknesses

| Strengths | Weaknesses |
|---|---|
| Very fast, low overhead | Cannot route by URL, header or cookie |
| Works for any TCP/UDP protocol | No content-based decisions |
| Simple | No request-level smarts |

---

## 4. L7 Load Balancer in Depth

An L7 load balancer works at the application layer, where we have HTTP, HTTPS and WebSockets. It **understands**:

- HTTP requests and methods
- URL path
- Headers
- Cookies
- Request body

### Worked example

Several microservices, each with multiple instances:

- Chat service: Chat-1, Chat-2
- User service: User-1, User-2

A user updates their profile, triggering `/api/v1/update-profile`. The L7 load balancer reads the path, understands that it belongs to the **User** service, checks which User instances are available (often via service discovery), and routes to a free one.

This means an L7 balancer can do **content-based routing** and, effectively, some of what an API gateway does.

### Strengths and weaknesses

| Strengths | Weaknesses |
|---|---|
| Path, header, cookie, host-based routing | More CPU and processing per request |
| Can terminate SSL, rewrite headers | Slightly higher latency than L4 |
| Smarter health checks and observability | More complex configuration |

---

## 5. L4 vs L7 Comparison

| Feature | L4 | L7 |
|---|---|---|
| OSI layer | Transport (4) | Application (7) |
| Protocols | TCP, UDP | HTTP, HTTPS, WebSockets |
| Sees | IPs, ports, protocol | URL, headers, cookies, body |
| Routing basis | Connection info | Request content |
| Speed | Faster | Slower (more inspection) |
| Intelligence | Low | High |
| Example use | Raw TCP services, extreme throughput | Microservice routing, web APIs |

---

## 6. Load Balancing Algorithms Overview

```
Load balancing algorithms
├── Static  (ignore live server state)
│     ├── Round Robin
│     ├── Weighted Round Robin
│     └── IP Hash
└── Dynamic (use live server state)
      ├── Least Connection
      ├── Weighted Least Connection
      └── Least Response Time
```

---

## 7. Static Algorithms

### 7.1 Round Robin

Requests are handed to servers **in a repeating cycle**.

With users U1–U4 and two instances (Chat-1, Chat-2):

```
Request 1 → Chat-1
Request 2 → Chat-2
Request 3 → Chat-1
Request 4 → Chat-2
Request 5 → Chat-1
Request 6 → Chat-2
```

**Result:** traffic is split **equally**.

**Problem:** servers are not always equal. If Server A has many CPUs and lots of memory, and Server B is small, equal distribution wastes A's capacity (A can handle, say, 100 requests but gets only 5).

### 7.2 Weighted Round Robin

Assign each server a **weight** reflecting its capacity.

Example: Chat-1 (powerful) has weight **3**, Chat-2 (weaker) has weight **1**.

```
Requests 1,2,3 → Chat-1
Request  4     → Chat-2
Requests 5,6,7 → Chat-1
Request  8     → Chat-2
```

This fixes the unused-capacity issue.

**Remaining problem:** it still ignores how long each request takes. Suppose requests 1–3 on Chat-1 finish in 10 ms each, but request 4 sent to weak Chat-2 takes **10 seconds**. If request 8 also lands on Chat-2, the weak server gets overloaded again, because the algorithm only counts requests, not their cost.

### 7.3 IP Hash

Pass the **client IP address** into a hash function. The hash value maps to a server.

```
hash(client IP) → value → server
value 1 → Chat-1
value 2 → Chat-2
```

The same client IP always hashes to the same value and therefore the same server. This gives a form of **session affinity**.

**Problem: proxies.** Suppose a proxy server sits between users and the load balancer. All four users' requests now appear to come from **one source IP** (the proxy's). The hash is therefore **always the same**, and **every request goes to one instance**, defeating load balancing.

---

## 8. Dynamic Algorithms

Dynamic algorithms use **live state** (connections, response times).

### 8.1 Least Connection

Route the new request to the server with the **fewest active connections**.

Example: U1 and U2 are connected to Chat-1; U3 is connected to Chat-2. A new request from U4 arrives:

| Server | Active connections |
|---|---|
| Chat-1 | 2 |
| Chat-2 | 1 |

→ U4 goes to **Chat-2** (now 2 connections each).

**Problem:** it counts connections, not capacity or traffic. Suppose Server A is very powerful and Server B is weak, and the weak one has fewer connections. New traffic keeps going to the weak server while the strong one sits idle. Also, an open connection does not necessarily mean it is sending traffic: a connection may be idle while another is very busy.

### 8.2 Weighted Least Connection

Fixes the above by assigning **weights** and comparing a **ratio**:

```
ratio = active connections / weight
```

Send the request to the server with the **lowest ratio**.

Example: Server 1 has weight 10 and 2 active connections → ratio 2/10 = **0.2**. Server 2 has weight 10 and 1 active connection → ratio 1/10 = **0.1**. Server 2 (lower ratio) gets the new request.

With unequal weights, the stronger server naturally absorbs more connections before its ratio rises, so its capacity is actually used.

### 8.3 Least Response Time

This algorithm uses **TTFB**: **Time To First Byte**, the interval between sending a request and receiving the response from the server (for a chat message: send message → server acknowledges → message delivered → server acknowledges again).

For each server compute:

```
score = active connections × TTFB
```

Send the request to the server with the **lowest score**.

**Worked example** (three servers):

| Server | Active connections | TTFB (ms) | Score |
|---|---|---|---|
| Server 1 | 2 | 3 | **6** |
| Server 2 | 0 | 2 | **0** |
| Server 3 | 4 | 1 | **4** |

Lowest = Server 2 → route the new request there.

After that, Server 2 has 1 connection → score 1 × 2 = **2**. Still the lowest (Server 1 = 6, Server 3 = 4), so the next request also goes to Server 2.

**Tie-break rule:** if two servers tie on score (e.g. both 4), fall back to **round robin** between them.

---

## 9. Choosing an Algorithm

| Situation | Good fit |
|---|---|
| Identical servers, short uniform requests | Round robin |
| Servers of different capacity, uniform requests | Weighted round robin |
| Need the same client → same server (sessions, caching) | IP hash (beware of proxies) |
| Long-lived connections (WebSockets, chat) | Least connection |
| Mixed capacity plus long connections | Weighted least connection |
| Latency-sensitive services with variable request cost | Least response time |

**Static** = simple, predictable, cheap, but blind to current conditions.
**Dynamic** = adaptive and efficient, but needs monitoring data and slightly more overhead.

---

## 10. Interview Questions

1. What is the difference between L4 and L7 load balancers? Which is faster? Which is smarter?
2. Can an L4 load balancer route `/api/v1/orders` to the Order service? Why or why not?
3. What information does an L4 balancer use to choose a server?
4. What is the weakness of round robin? How does weighted round robin fix it? What does it still miss?
5. Why can IP hash break when a proxy is present?
6. Compare least connection and weighted least connection.
7. What is TTFB and how is it used in least response time?
8. What happens when two servers tie in least response time?
9. In a chat app using persistent connections, which algorithm would you pick and why?
10. Is a load balancer itself a single point of failure? How do you mitigate it? (Run several, with DNS or floating-IP failover.)

---

## 11. Quick Revision Sheet

- **L4** = transport layer, TCP/UDP, sees IP/port/protocol only. Fast, content-blind.
- **L7** = application layer, HTTP/HTTPS/WebSocket, sees paths/headers/cookies. Smarter, heavier.
- **Static:** Round Robin, Weighted Round Robin, IP Hash.
- **Dynamic:** Least Connection, Weighted Least Connection, Least Response Time.
- **Weighted LC formula:** `active connections ÷ weight` → pick the lowest.
- **Least response time formula:** `active connections × TTFB` → pick the lowest; tie → round robin.
- **IP hash + proxy** = everything lands on one server.
- **Post-office clerk analogy:** L4 reads only the envelope.

---

**Previous:** [Chapter 3](../Chapter-03-API-Gateway-vs-Load-Balancer/README.md) · **Next:** [Chapter 5 — Proxy vs Reverse Proxy](../Chapter-05-Proxy-vs-Reverse-Proxy/README.md)
