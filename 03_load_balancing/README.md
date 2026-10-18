# Chapter 4 — Load Balancing: Types, Algorithms, and Production Design

> **Personal study notes** — A ground-up guide to understanding load balancing, Layer 4 vs. Layer 7 routing, major load-balancing algorithms, their mathematical intuition, and their behavior in real-world distributed systems.
>
> **Learning objective:** Understand what a load balancer observes, how it chooses a backend, how to simulate its algorithms by hand, and how to select, implement, and troubleshoot the right strategy in production.

---

## Table of Contents

1. [Why Load Balancing Exists](#1-why-load-balancing-exists)
2. [How a Load Balancer Works](#2-how-a-load-balancer-works)
3. [Load Balancer Types: L4 vs. L7](#3-load-balancer-types-l4-vs-l7)
4. [Layer 4 Load Balancing in Depth](#4-layer-4-load-balancing-in-depth)
5. [Layer 7 Load Balancing in Depth](#5-layer-7-load-balancing-in-depth)
6. [L4 vs. L7: Complete Comparison](#6-l4-vs-l7-complete-comparison)
7. [Load-Balancing Algorithms: Overview](#7-load-balancing-algorithms-overview)
8. [Static Algorithms](#8-static-algorithms)
9. [Dynamic Algorithms](#9-dynamic-algorithms)
10. [Session Affinity and Sticky Sessions](#10-session-affinity-and-sticky-sessions)
11. [Health Checks and Failure Handling](#11-health-checks-and-failure-handling)
12. [Load Balancing in Production](#12-load-balancing-in-production)
13. [Common Misconceptions](#13-common-misconceptions)
14. [Choosing the Right Algorithm](#14-choosing-the-right-algorithm)
15. [Implementation: Simulating Algorithms](#15-implementation-simulating-algorithms)
16. [Interview Questions and Answers](#16-interview-questions-and-answers)
17. [Quick Revision Sheet](#17-quick-revision-sheet)

---

# 1. Why Load Balancing Exists

## 1.1 The problem: one server is not enough

Imagine building a chat application.

Initially, the application runs on one server:

```text
Users
  |
  v
+----------------+
|  Chat Server   |
|                |
| Send messages  |
| Manage rooms   |
| Handle users   |
+----------------+
```

This works while traffic is low. However, as the application grows, the server may run out of CPU, memory, network capacity, or available connections.

For example:

- 100 concurrent users: the server handles the traffic comfortably.
- 10,000 concurrent users: response times begin increasing.
- 100,000 concurrent users: the server may reach its capacity.
- 1,000,000 concurrent users: one server is unlikely to be sufficient for the workload.

These numbers are illustrative, not universal capacity limits. Actual capacity depends on hardware, workload, application design, and connection behavior.

We need a way to distribute traffic across multiple servers.

## 1.2 Horizontal scaling

Instead of continually upgrading one machine, we run multiple instances of the application.

```text
                   Users
                     |
          +----------+----------+
          |                     |
          v                     v
      User A                 User B
          |                     |
          +----------+----------+
                     |
                     v
              +-------------+
              | Load        |
              | Balancer    |
              +------+------+
                     |
           +---------+---------+
           |         |         |
           v         v         v
       +--------+ +--------+ +--------+
       | Chat-1 | | Chat-2 | | Chat-3 |
       +--------+ +--------+ +--------+
```

Each backend runs the application and can process requests independently, subject to its configuration and access to shared dependencies.

This is called **horizontal scaling**: increasing capacity by adding more instances rather than relying exclusively on a more powerful machine.

The load balancer becomes the entry point that distributes incoming traffic among these instances.

## 1.3 What does a load balancer actually do?

A load balancer typically performs several responsibilities:

1. **Traffic distribution:** Selects a backend for an incoming connection or request.
2. **Health awareness:** Avoids routing new traffic to backends considered unhealthy.
3. **Capacity-aware routing:** May consider connections, weights, latency, or other load indicators.
4. **Failure handling:** Stops sending new traffic to failed instances and may retry eligible operations.
5. **Connection management:** Depending on the implementation, maintains connections, forwards traffic, or terminates client connections.
6. **Observability:** Exposes metrics such as request rate, connection count, latency, and error rate.

Not every load balancer performs every function. For example, a basic L4 balancer may not inspect HTTP requests at all.

## 1.4 Load balancing does not create capacity by itself

Suppose three servers can each process 1,000 requests per second under a particular workload.

Their theoretical combined capacity is approximately:

\[
C_{\text{total}} = C_1 + C_2 + C_3
\]

\[
C_{\text{total}} = 1000 + 1000 + 1000 = 3000
\]

This does not guarantee that the system can sustain 3,000 requests per second. Network overhead, uneven request costs, shared database limits, coordination, and other bottlenecks may reduce the effective capacity.

If all three application servers depend on one overloaded database, adding more application servers may actually make the database problem worse.

**Key principle:** A load balancer distributes work. It cannot eliminate bottlenecks elsewhere in the system.

---

# 2. How a Load Balancer Works

Before learning individual algorithms, understand the common decision-making process.

## 2.1 The request-routing lifecycle

Consider a client requesting:

```http
GET /api/v1/orders HTTP/1.1
Host: shop.example.com
```

A typical HTTP load balancer follows a process similar to this:

```text
Client sends request
        |
        v
Load balancer receives traffic
        |
        v
Determine routing context
        |
        v
Identify eligible healthy backends
        |
        v
Apply the selected algorithm
        |
        v
Forward request to chosen backend
        |
        v
Receive backend response
        |
        v
Return response to client
```

The exact sequence depends on the protocol and implementation.

An L4 load balancer does not need to understand the HTTP path. An L7 load balancer can inspect it and use it as a routing condition.

## 2.2 What information is available?

Different load balancers have access to different information.

| Information | L4 balancer | L7 HTTP balancer |
|---|---|---|
| Client IP address | Yes | Usually |
| Client source port | Yes | Usually available through the connection |
| Destination IP and port | Yes | Usually available |
| TCP or UDP protocol | Yes | Underlying transport is known |
| HTTP method | No | Yes, for HTTP traffic |
| URL path | No | Yes |
| HTTP headers | No | Yes |
| HTTP cookies | No | Yes |
| HTTP request body | No | Potentially, depending on configuration |

The distinction is not that L4 has no information. It is that L4 normally operates without parsing application-level message content.

## 2.3 Backend selection versus request processing

These are separate responsibilities.

- **Backend selection:** Which instance should receive the traffic?
- **Request processing:** What does the request mean, and how should the application respond?

For example, an L7 balancer may route `/api/v1/orders` to the Order service. The Order service still validates the request, checks permissions, applies business rules, and retrieves order data.

A load balancer is not automatically an API implementation or business-logic engine.

## 2.4 Health and eligibility come before selection

Suppose a backend pool contains:

| Backend | Health | Weight |
|---|---|---:|
| Server A | Healthy | 1 |
| Server B | Unhealthy | 1 |
| Server C | Healthy | 1 |

Even if the algorithm would otherwise select Server B, a correctly configured balancer generally excludes it from eligible backends.

The decision is therefore better understood as:

1. Determine which backends are eligible.
2. Apply the selection policy to those backends.
3. Forward traffic.
4. Observe the outcome and update relevant state.

The exact failure behavior, retries, and connection handling vary by product.

---

# 3. Load Balancer Types: L4 vs. L7

## 3.1 The OSI model

The OSI model divides network communication into seven conceptual layers.

| Layer | Name | Examples or responsibilities |
|---:|---|---|
| 7 | Application | HTTP, DNS, application protocols |
| 6 | Presentation | Data representation, encoding, encryption concepts |
| 5 | Session | Session establishment and management concepts |
| 4 | Transport | TCP, UDP |
| 3 | Network | IP addressing and routing |
| 2 | Data Link | Ethernet frames, MAC addressing |
| 1 | Physical | Physical transmission of bits |

For load balancing, the most important distinction is between Layer 4 and Layer 7.

- **L4 load balancing:** Primarily distributes traffic using transport-level connection information.
- **L7 load balancing:** Distributes application requests using application-level information, such as HTTP paths and headers.

Real systems do not always fit perfectly into the OSI model, but the distinction remains useful.

## 3.2 Layer 4: transport-level decisions

An L4 balancer can use information such as:

```text
Source IP:       192.0.2.15
Source port:     53124
Destination IP:  203.0.113.10
Destination port: 443
Protocol:        TCP
```

It can select a backend without interpreting the application message.

It does not need to know whether the connection will carry a login request, an order, a chat message, or some other application operation.

## 3.3 Layer 7: application-level decisions

An L7 balancer can inspect HTTP information:

```http
GET /api/v1/orders HTTP/1.1
Host: shop.example.com
Authorization: Bearer <token>
```

It can apply rules such as:

```text
/api/v1/orders/*   -> Order service
/api/v1/users/*    -> User service
/api/v1/payments/* -> Payment service
```

It can also route based on hostnames, selected headers, or cookies.

This makes L7 particularly useful for web applications, API routing, and reverse-proxy architectures.

## 3.4 An important HTTPS distinction

When HTTPS is encrypted, the HTTP path, headers, and body are not ordinarily visible to a passive intermediary that only forwards encrypted TLS traffic.

An L7 load balancer can inspect HTTP information when it terminates TLS or otherwise receives decrypted application traffic.

Two common arrangements are:

**TLS termination at the load balancer**

```text
Client -- HTTPS --> L7 Load Balancer -- HTTP --> Backend
                       |
                  Decrypts TLS
                  Inspects HTTP
```

**TLS passthrough**

```text
Client -- encrypted TLS traffic --> L4-capable balancer --> Backend
```

In the second arrangement, the balancer does not see the decrypted HTTP path. It cannot route using that path unless another component terminates TLS and performs application-level inspection.

TLS passthrough may be appropriate when encryption must remain end-to-end between the client and backend, although network-level routing and TLS metadata handling depend on the implementation.

---

# 4. Layer 4 Load Balancing in Depth

## 4.1 What an L4 balancer sees

Consider this TCP connection:

```text
Client:
  IP   = 192.0.2.15
  Port = 53124

Load balancer:
  IP   = 203.0.113.10
  Port = 443

Backend pool:
  Server A = 10.0.0.11:5000
  Server B = 10.0.0.12:5000
  Server C = 10.0.0.13:5000
```

The balancer knows the transport-level information and can select a backend according to its algorithm and health state.

It does not need to parse the application message to do so.

## 4.2 Worked example

Assume three healthy backends:

```text
Client
  |
  | TCP connection to public endpoint
  v
+----------------+
| L4 Balancer    |
+-------+--------+
        |
        | Select Server B
        v
+----------------+
| Server B       |
| Port 5000      |
+----------------+
```

The client establishes a connection to the public endpoint. The load balancer directs the connection to Server B.

The connection then carries the application traffic that Server B handles.

The application might exchange messages to join a room, send a chat message, or retrieve data. The L4 balancer generally does not distinguish these operations.

## 4.3 Connection affinity

For a TCP connection, backend selection normally determines where that connection's traffic is forwarded for its lifetime.

Suppose a chat client establishes a TCP connection that is assigned to Server B:

```text
TCP connection C1 -> Server B
  Message 1        -> Server B
  Message 2        -> Server B
  Message 3        -> Server B
```

The balancer does not ordinarily select a different backend for every individual message within the same TCP connection.

This matters for WebSockets, which typically operate over a long-lived TCP connection after the HTTP upgrade handshake.

**Important distinction:** A TCP connection staying on one server does not guarantee that a user will reach the same server after reconnecting. A new connection may be assigned to a different backend.

## 4.4 What happens when a backend fails?

Suppose Server B crashes while the client is connected.

The load balancer may mark Server B unhealthy and stop assigning it new connections. However, an existing connection to Server B generally cannot be transferred seamlessly to Server C with all its in-memory state intact.

The client may need to reconnect.

For chat applications, this means the system needs additional mechanisms for reconnecting users, recovering subscriptions, and retrieving messages that may have been missed.

Load balancing alone does not provide durable messaging.

## 4.5 Common L4 implementation approaches

| Approach | Basic idea | Important trade-off |
|---|---|---|
| NAT-based forwarding | Translates addresses or ports to direct traffic to a backend | Return-path and connection-tracking requirements |
| Direct routing | Forwards traffic using a routing technique that may preserve the original destination or bypass the balancer on the return path | More complex network configuration |
| TCP proxy | Terminates or proxies TCP connections and establishes backend-side connections | Additional connection-management overhead |
| UDP forwarding | Distributes UDP datagrams according to supported flow-selection rules | No inherent TCP-style connection semantics |

The exact behavior depends on the load-balancing product and operating mode.

## 4.6 Advantages and disadvantages

| Advantages | Disadvantages |
|---|---|
| Can handle high-throughput transport traffic efficiently | Cannot route using HTTP paths or headers without application inspection |
| Supports non-HTTP TCP and UDP services | Limited application-level observability |
| Avoids parsing application messages when unnecessary | Application-aware retries and routing are limited |
| Suitable for TCP passthrough scenarios | Does not understand business operations |

**Use L4 when:** You need efficient transport-level distribution, support for non-HTTP protocols, or TLS passthrough without HTTP-level routing.

---

# 5. Layer 7 Load Balancing in Depth

## 5.1 What an L7 balancer sees

An L7 HTTP balancer can understand an HTTP request after the relevant traffic has been decrypted.

For example:

```http
POST /api/v1/orders HTTP/1.1
Host: shop.example.com
Content-Type: application/json
Authorization: Bearer <token>

{
  "productId": "P101",
  "quantity": 2
}
```

Depending on its configuration, the balancer can inspect:

- HTTP method: `POST`
- Path: `/api/v1/orders`
- Host: `shop.example.com`
- Headers
- Cookies
- Selected request-body content, if configured to inspect it

Inspecting a request body can require buffering and additional processing, so it is not necessarily enabled or desirable for every request.

## 5.2 Content-based routing

Imagine an application composed of three microservices:

```text
                       Clients
                          |
                          v
                  +---------------+
                  | L7 Load       |
                  | Balancer      |
                  +-------+-------+
                          |
             +------------+------------+
             |            |            |
             v            v            v
        /users/*      /orders/*    /payments/*
             |            |            |
             v            v            v
        User Service  Order Service Payment Service
```

Routing rules can direct different requests to different services:

```text
GET  /api/v1/users/123    -> User service
POST /api/v1/orders       -> Order service
POST /api/v1/payments     -> Payment service
```

After choosing the service, the balancer may choose an individual instance within that service's backend pool.

This involves two logically distinct decisions:

1. **Service routing:** Which service should handle the request?
2. **Instance selection:** Which healthy instance of that service should receive it?

A single proxy can perform both decisions, or they can be handled by separate infrastructure components.

## 5.3 Host-based routing

The hostname can determine the destination:

```text
api.example.com     -> API service
admin.example.com   -> Admin service
images.example.com  -> Image service
```

This is useful when multiple applications share the same public entry point.

## 5.4 Header-based routing

Headers can direct traffic to different backend pools:

```text
X-API-Version: v1 -> Version 1
X-API-Version: v2 -> Version 2
```

A deployment might also use a header to send test requests to a new version.

Header-based routing must be designed carefully: clients should not be able to bypass authorization or other security controls merely by supplying a particular header.

## 5.5 L7 health checks and observability

An L7 balancer may perform an HTTP health check:

```http
GET /health HTTP/1.1
Host: service.internal
```

It can verify that a backend returns an expected status code or response.

However, a successful health endpoint does not necessarily prove that every business operation works. A service might return `200 OK` from `/health` while its payment provider is unavailable.

Production systems often distinguish between:

- **Liveness:** Is the process running, and should it continue running?
- **Readiness:** Is the instance ready to receive traffic?
- **Dependency health:** Are important downstream systems available?

The correct policy depends on the application. Marking every instance unhealthy whenever an optional dependency fails can cause an otherwise recoverable service to disappear from the pool.

## 5.6 Advantages and disadvantages

| Advantages | Disadvantages |
|---|---|
| Path-, host-, header-, and cookie-based routing | Requires application-protocol processing |
| Can terminate TLS and inspect HTTP | More CPU and memory overhead in many configurations |
| Can expose request-level metrics and logs | Configuration is more complex |
| Can support HTTP-aware retries and timeouts | Retries can duplicate side effects if designed incorrectly |
| Useful for API and microservice routing | Request buffering and inspection can increase latency |

**Use L7 when:** You need HTTP-aware routing, TLS termination, API routing, application-level observability, or HTTP-specific traffic policies.

---

# 6. L4 vs. L7: Complete Comparison

| Feature | L4 Load Balancer | L7 Load Balancer |
|---|---|---|
| Primary layer | Transport | Application |
| Typical protocols | TCP, UDP | HTTP, HTTPS, WebSockets |
| Understands HTTP paths | No | Yes, when HTTP is available for inspection |
| Understands cookies | No | Yes |
| Routes by hostname | Not using ordinary HTTP host headers | Yes |
| Routes by HTTP headers | No | Yes |
| Typical decision unit | Connection or flow | HTTP request, connection, or stream, depending on protocol |
| TLS termination | Not required; depends on product | Commonly supported |
| HTTP-aware health checks | Not inherent | Commonly supported |
| Processing overhead | Often lower | Often higher |
| Common uses | TCP services, UDP traffic, TLS passthrough | Web applications, APIs, reverse proxies |

### Interview takeaway

**L4 decides primarily from transport-level information; L7 can make decisions using application-level information.**

L4 is not automatically faster in every deployment, and L7 is not automatically better. Actual performance depends on hardware, software, TLS processing, traffic patterns, and configuration.

Also, a load balancer can combine capabilities. The terms L4 and L7 describe its behavior and capabilities, not necessarily two completely separate physical devices.

---

# 7. Load-Balancing Algorithms: Overview

A load-balancing algorithm determines which eligible backend receives new traffic.

Different algorithms optimize different objectives. There is no universally best algorithm.

## 7.1 Classification

```text
Load-balancing algorithms
|
+-- Static / state-independent selection
|   |
|   +-- Round Robin
|   +-- Weighted Round Robin
|   +-- IP Hash
|
+-- Dynamic / state-aware selection
    |
    +-- Least Connections
    +-- Weighted Least Connections
    +-- Least Response Time
```

This classification is useful for learning, but not absolute. For example, an IP-hash implementation may also account for backend health, and weighted round robin can react to configuration changes.

The important distinction is whether the selection decision incorporates changing backend load measurements.

## 7.2 Two different kinds of load

Before studying the algorithms, distinguish:

**Incoming request rate**

```text
How many requests per second is a server receiving?
```

**Current workload**

```text
How much work is the server currently processing?
```

These are not equivalent.

A server may receive only 10 requests per second but spend 20 seconds processing each request. Another server may receive 100 requests per second but finish each request in 2 milliseconds.

A good algorithm should match the workload and the resource bottleneck.

## 7.3 Why request counting can be misleading

Consider two requests:

```text
Request A: Read a cached profile      -> 2 ms
Request B: Generate a large report    -> 8 seconds
```

Counting each request equally ignores the difference in processing cost.

Similarly, one active TCP connection may be idle, while another continuously transfers data.

This is why connection count, request rate, latency, CPU utilization, and actual resource consumption must not be treated as interchangeable measurements.

---

# 8. Static Algorithms

## 8.1 Round Robin

### Concept

Round Robin selects backends in a repeating cycle.

For three servers:

```text
Request 1 -> A
Request 2 -> B
Request 3 -> C
Request 4 -> A
Request 5 -> B
Request 6 -> C
```

Assuming all servers remain eligible, each receives approximately one-third of the assignments.

### Worked example

Suppose five requests arrive:

| Request | Selected backend | Next backend |
|---:|---|---|
| 1 | A | B |
| 2 | B | C |
| 3 | C | A |
| 4 | A | B |
| 5 | B | C |

Final distribution:

| Backend | Requests |
|---|---:|
| A | 2 |
| B | 2 |
| C | 1 |

The distribution is as even as possible for this small batch, but not every server must receive exactly the same number of requests at every moment.

### Advantages

- Simple to understand and implement.
- Low selection overhead.
- Works well when servers have similar capacity and requests have similar cost.

### Limitations

Round Robin does not directly consider:

- CPU utilization
- Memory pressure
- Active connection count
- Request execution time
- Different backend capacities

If one server is already overloaded, Round Robin may still assign it another request.

### When to use it

Use Round Robin when backends are relatively homogeneous and workloads are reasonably uniform.

---

## 8.2 Weighted Round Robin

### Concept

Weighted Round Robin assigns each server a weight representing its relative share of traffic.

Suppose:

| Server | Weight |
|---|---:|
| A | 3 |
| B | 1 |

The intended long-run distribution is approximately:

- Server A: 75%
- Server B: 25%

The proportion for server \(i\) is:

\[
P_i = \frac{w_i}{\sum_j w_j}
\]

For Server A:

\[
P_A = \frac{3}{3+1} = 75\%
\]

For Server B:

\[
P_B = \frac{1}{3+1} = 25\%
\]

### Worked example

A simple repeating schedule is:

```text
A -> A -> A -> B -> repeat
```

For eight requests:

```text
1 -> A
2 -> A
3 -> A
4 -> B
5 -> A
6 -> A
7 -> A
8 -> B
```

Result:

| Server | Requests | Share |
|---|---:|---:|
| A | 6 | 75% |
| B | 2 | 25% |

Real implementations may use smooth weighted round robin or other scheduling strategies to distribute assignments more evenly over shorter windows.

### What does the weight mean?

A weight is a configured relative preference. It may reflect CPU capacity, observed performance, or an operational decision.

A weight of 3 does not guarantee that a server can process exactly three times as much work as another server.

### Limitation

Weighted Round Robin controls the allocation of new traffic, but it does not react directly to a long-running request or an overloaded server unless its weights or eligibility are updated.

A powerful server can receive its expected traffic share and still become overloaded if its requests are disproportionately expensive.

### When to use it

Use Weighted Round Robin when backend capacities differ and request costs are reasonably predictable.

---

## 8.3 IP Hash

### Concept

IP Hash uses a hash of the client IP address to select a backend.

Conceptually:

\[
h = \operatorname{hash}(\text{client IP})
\]

\[
\text{backend} = f(h,\text{backend pool})
\]

A simplified modulo-based implementation might calculate:

\[
\text{index} = h \bmod N
\]

where \(N\) is the number of eligible servers.

This is a teaching model. Real implementations may use different hashing algorithms and consistent-hashing strategies.

### Worked example

Suppose there are three backends:

```text
A, B, C
```

For a fixed backend pool:

```text
Client 192.0.2.10 -> A
Client 192.0.2.11 -> C
Client 192.0.2.12 -> B
Client 192.0.2.10 -> A again
```

The same IP tends to map to the same backend while the mapping configuration remains stable.

### Why use IP Hash?

It can provide a form of session affinity when an application stores session state locally on a backend.

It can also improve cache locality when a client repeatedly accesses the same server.

However, IP Hash does not itself store a session or guarantee that a session will survive backend failure.

### Problem: NAT and reverse proxies

Suppose thousands of users connect through a shared corporate proxy.

The load balancer may observe the same source IP for all of them:

```text
User A ----\
User B -----+----> Corporate Proxy ----> Load Balancer
User C -----+                             |
User D ----/                              v
                                       Server A
```

If every request is hashed using only the proxy's IP, many users may be assigned to the same backend.

This can create severe imbalance.

The same issue can occur with carrier-grade NAT, where many users share a public IP address.

### Backend membership changes

With simple modulo hashing, changing the number of backends can remap many clients.

For example:

\[
h \bmod 3 \ne h \bmod 4
\]

for many values of \(h\).

Consistent hashing can reduce unnecessary remapping when the backend pool changes, though it does not eliminate all movement or guarantee even load.

### Security and correctness

If a system uses forwarded IP headers, those headers must be accepted only from trusted proxies and sanitized appropriately. A client-supplied header must not automatically be treated as the verified client address.

### When to use it

Use IP-based affinity only when its limitations are acceptable. Prefer shared session storage or other stateless designs when possible.

---

# 9. Dynamic Algorithms

Dynamic algorithms use changing measurements or state to influence backend selection.

Unlike simple Round Robin, they can adapt to differences in active work. Their effectiveness depends on measurement quality, update frequency, and the relationship between the metric and actual capacity.

## 9.1 Least Connections

### Concept

Least Connections chooses the eligible server with the fewest active connections.

\[
\text{selected} = \arg\min_i C_i
\]

where \(C_i\) is the number of active connections on server \(i\).

### Worked example

Suppose:

| Server | Active connections |
|---|---:|
| A | 12 |
| B | 5 |
| C | 8 |

The next connection goes to Server B because it has the lowest connection count.

After assignment:

| Server | Active connections |
|---|---:|
| A | 12 |
| B | 6 |
| C | 8 |

The algorithm repeats the selection for subsequent new connections.

### Why it can outperform Round Robin

Suppose connections have different lifetimes:

```text
Server A: 50 long-lived connections
Server B:  5 active connections
Server C: 12 active connections
```

Least Connections can prefer Server B rather than blindly following a fixed cycle.

This is especially useful for workloads where connection duration varies substantially.

### Limitation: connections are not work

An open connection may be idle.

Another connection may be using substantial CPU or transferring a large amount of data.

For example:

| Server | Connections | Actual workload |
|---|---:|---|
| A | 10 | Ten mostly idle connections |
| B | 5 | Five CPU-intensive requests |

Least Connections may prefer B, even though B is processing more work.

The algorithm assumes that active connection count is a useful proxy for load. That assumption is not always true.

### When to use it

Use Least Connections when connection duration varies and connection count is meaningfully correlated with workload.

It is particularly useful for long-lived TCP connections, provided connection count is a suitable load indicator.

---

## 9.2 Weighted Least Connections

### Concept

Weighted Least Connections accounts for differences in backend capacity.

A common simplified comparison is:

\[
S_i = \frac{C_i}{w_i}
\]

where:

- \(C_i\) = active connections
- \(w_i\) = configured backend weight
- \(S_i\) = normalized connection score

Select the eligible backend with the lowest score.

This formula describes a useful normalized model. Actual implementations may use different internal comparisons.

### Worked example

Suppose:

| Server | Active connections | Weight |
|---|---:|---:|
| A | 20 | 10 |
| B | 6 | 2 |
| C | 12 | 8 |

Calculate each score.

Server A:

\[
S_A = \frac{20}{10} = 2
\]

Server B:

\[
S_B = \frac{6}{2} = 3
\]

Server C:

\[
S_C = \frac{12}{8} = 1.5
\]

Comparison:

| Server | Score |
|---|---:|
| A | 2.0 |
| B | 3.0 |
| C | 1.5 |

**Selected backend: Server C**, because its normalized connection score is lowest.

### Why normalization matters

Imagine Server A can handle approximately five times as much work as Server B.

Sending the same number of connections to both may be inappropriate.

A weighted algorithm attempts to allocate connections in proportion to relative capacity rather than treating every server as identical.

### Important limitation

Weights do not magically measure current workload. A poorly chosen weight can still direct too much traffic to a weaker server.

The algorithm also inherits Least Connections' weakness: an active connection is not necessarily an active unit of work.

### When to use it

Use Weighted Least Connections when backend capacities differ and active connections are a meaningful indicator of load.

---

## 9.3 Least Response Time

### Concept

Least Response Time uses latency measurements to favor backends that are responding quickly.

However, **Least Response Time is a family of strategies, not one universal formula**.

Implementations may use:

- Recent response-time measurements
- Smoothed latency averages
- Active connection counts
- Load weights
- Combinations of latency and connection information

A simplified teaching formula is:

\[
S_i = C_i \times T_i
\]

where:

- \(C_i\) = active connections
- \(T_i\) = measured or averaged response time
- \(S_i\) = simplified score

Under this model, select the eligible server with the lowest score.

The formula is useful for understanding the intuition, but should not be assumed to match every production implementation.

### Worked example

Suppose three servers have these measurements:

| Server | Active connections | Response time | Score |
|---|---:|---:|---:|
| A | 2 | 3 ms | 6 |
| B | 0 | 2 ms | 0 |
| C | 4 | 1 ms | 4 |

Calculations:

\[
S_A = 2 \times 3 = 6
\]

\[
S_B = 0 \times 2 = 0
\]

\[
S_C = 4 \times 1 = 4
\]

The lowest score is Server B, so the next assignment goes to B.

After the assignment, assume its active connection count becomes 1 and its measured response time remains 2 ms:

\[
S_B = 1 \times 2 = 2
\]

Under the simplified model, B remains the lowest-scoring server.

This example assumes that the measurements are immediately available and that the response-time estimate does not change. Real systems have measurement delays and changing workloads.

### What does response time mean?

For an HTTP request, response time might mean the interval from when the request is sent to the backend until the first response byte arrives, or until the complete response is received.

These measurements are different.

**Time to First Byte (TTFB)** generally measures the time until the first response byte is received at the measurement point. It is not necessarily the same as total request duration.

For a long-lived WebSocket connection, response time is not a straightforward substitute for measuring ongoing connection workload.

### Why latency alone can be misleading

A backend can have low latency because it has recently received very little traffic, yet lack the capacity to handle a large new workload.

Conversely, a server can have temporarily high latency because of one unusually expensive request.

Useful implementations therefore consider how measurements are collected and smoothed, how stale data is handled, and whether latency is correlated with actual saturation.

### Tie-breaking

If two servers have the same score, a system needs a tie-breaking policy.

Possible choices include:

- Round Robin among tied servers
- Random selection
- Stable ordering
- A secondary metric

Round Robin is a reasonable choice for a teaching implementation, but it is not a universal production rule.

### When to use it

Use latency-aware selection when response-time differences are meaningful, measurements are reliable, and latency is an important service objective.

---

## 9.4 Algorithm comparison by example

Suppose three backends have the following state:

| Server | Weight | Connections | Response time |
|---|---:|---:|---:|
| A | 4 | 12 | 20 ms |
| B | 2 | 4 | 10 ms |
| C | 1 | 1 | 5 ms |

Different algorithms can select different servers:

| Algorithm | Selection basis | Likely selection in this example |
|---|---|---|
| Round Robin | Current position in the cycle | Depends on the cycle |
| Weighted Round Robin | Configured weight | Depends on the weighted schedule |
| IP Hash | Client IP hash and backend mapping | Depends on the hash |
| Least Connections | Smallest connection count | C |
| Weighted Least Connections | Connections divided by weight | B or C, depending on scores and exact policy |
| Least Response Time | Latency-aware score | Depends on the chosen latency model |

For Weighted Least Connections using \(C_i/w_i\):

\[
S_A = 12/4 = 3
\]

\[
S_B = 4/2 = 2
\]

\[
S_C = 1/1 = 1
\]

The selected server is C.

For the simplified Least Response Time formula:

\[
S_A = 12 \times 20 = 240
\]

\[
S_B = 4 \times 10 = 40
\]

\[
S_C = 1 \times 5 = 5
\]

The selected server is also C.

Different algorithms can produce the same decision for one snapshot while behaving differently over time.

---

# 10. Session Affinity and Sticky Sessions

## 10.1 The problem

Suppose an application stores a user's session in the memory of one backend.

```text
Login request -> Server A
Session stored in Server A's memory
```

If a later request goes to Server B, Server B may not have that session.

```text
Login  -> Server A
Profile -> Server B
```

The user may appear logged out or the application may fail to find the session.

## 10.2 What is session affinity?

Session affinity attempts to route related requests to the same backend.

Common approaches include:

- Source-IP affinity
- Cookie-based affinity
- Application-level session routing
- Consistent hashing on a suitable key

Cookie-based affinity can work well for HTTP applications because it can identify a client session more precisely than a shared source IP.

However, it must be configured carefully and must not replace authentication or authorization.

## 10.3 Sticky sessions versus shared session storage

**Sticky sessions:**

```text
User A -> Server A
User B -> Server B
```

The load balancer tries to preserve each user's backend assignment.

**Shared session storage:**

```text
                 +-----------------+
Server A ------->| Shared Session  |
Server B ------->| Store           |
Server C ------->|                 |
                 +-----------------+
```

Any backend can retrieve the session from the shared store.

Shared session storage improves the ability to move requests between backends, but introduces another dependency that needs capacity planning and failure handling.

### Trade-offs

| Sticky sessions | Shared session storage |
|---|---|
| Can reduce repeated session lookups | Supports requests across multiple backends |
| May simplify legacy applications | Helps keep application instances stateless |
| Can create uneven load | Adds network calls and storage dependency |
| Backend failure can disrupt session continuity | Shared-store failure can affect many instances |

**Best practice:** Prefer stateless application instances and shared session state when appropriate. Use affinity when it provides a justified benefit rather than as a substitute for sound application design.

---

# 11. Health Checks and Failure Handling

Load-balancing algorithms assume that a backend is eligible to receive traffic. Health checks help establish that eligibility.

## 11.1 Active health checks

The load balancer periodically probes a backend.

```text
Load balancer
    |
    +---- GET /health ----> Server A
    |
    +---- GET /health ----> Server B
    |
    +---- GET /health ----> Server C
```

If a backend repeatedly fails its configured checks, the balancer may remove it from the eligible pool.

Recovery typically requires successful checks or another configured recovery condition.

## 11.2 Passive health checks

A balancer may also observe real traffic.

Examples of concerning signals include:

- Connection failures
- Timeouts
- Repeated backend errors
- Unexpected connection resets

Passive health detection can identify problems that occur between active probes, but it depends on the traffic and failure-detection rules.

## 11.3 Health is not the same as capacity

A server can be healthy but overloaded.

```text
Server A: Healthy, CPU 20%
Server B: Healthy, CPU 98%
Server C: Unhealthy
```

A simple health check might consider both A and B eligible, even though B is approaching saturation.

Health checks answer whether a backend meets a configured eligibility condition. They do not necessarily measure whether it has enough spare capacity for another request.

## 11.4 Timeouts, retries, and circuit breakers

These mechanisms solve related but different problems.

- **Timeout:** Limits how long the caller waits for an operation.
- **Retry:** Attempts an operation again under defined conditions.
- **Circuit breaker:** Temporarily stops calls to a dependency that is repeatedly failing.
- **Health check:** Evaluates whether a backend meets the configured health criteria.

Retries must be designed carefully.

For example, if a payment request times out after the payment provider has already processed it, retrying blindly may charge the customer twice. Idempotency keys and operation-specific retry rules help prevent duplicate effects.

For streaming connections and non-idempotent operations, retry behavior requires particular care.

---

# 12. Load Balancing in Production

## 12.1 A load balancer can become a bottleneck

A single load-balancing instance may itself fail or reach capacity.

A common design uses multiple balancer instances:

```text
                  Clients
                     |
             +-------+-------+
             |               |
             v               v
        +----------+    +----------+
        | LB Node A|    | LB Node B|
        +----+-----+    +----+-----+
             |               |
             +-------+-------+
                     |
          +----------+----------+
          |          |          |
          v          v          v
       Server A   Server B   Server C
```

Traffic can be directed through DNS, a virtual IP, a cloud-managed frontend, or another highly available mechanism.

The exact failover strategy depends on the environment. Multiple load balancers alone do not guarantee high availability if they share a failed network, control plane, or upstream dependency.

## 12.2 Reverse proxy versus load balancer

A reverse proxy receives client traffic and forwards it to upstream servers.

A load balancer distributes traffic among multiple eligible backends.

These roles overlap substantially. An L7 reverse proxy can also function as a load balancer.

For example, an HTTP proxy may:

- Terminate TLS
- Route requests by path
- Distribute requests among instances
- Apply timeouts
- Collect request metrics

A separate API gateway may additionally manage API authentication, rate limits, quotas, and API-specific policies.

The names describe overlapping but distinct responsibilities.

## 12.3 Load balancing does not fix a shared database bottleneck

Consider:

```text
                    Load Balancer
                          |
              +-----------+-----------+
              |           |           |
              v           v           v
          App A         App B        App C
              |           |           |
              +-----------+-----------+
                          |
                          v
                    One Database
```

If the database is already saturated, increasing the number of application instances can increase the number of concurrent database requests without improving throughput.

Before scaling the application tier, investigate:

- Database connection-pool limits
- Query execution time
- Lock contention
- Indexing
- Cache hit rates
- Queue depth
- Database CPU, memory, and I/O
- Whether the workload needs read replicas or a different data-access design

Scaling the layer that is not the bottleneck can make the system more expensive without making it faster.

## 12.4 Observability: what should you measure?

Useful metrics include:

| Metric | Why it matters |
|---|---|
| Requests per second | Measures traffic volume |
| Active connections | Helps understand connection occupancy |
| Backend response latency | Shows how quickly requests are handled |
| Error rate | Detects failed requests |
| Timeout rate | Reveals slow or unresponsive dependencies |
| Backend health | Shows which instances are eligible |
| CPU and memory utilization | Indicates resource pressure |
| Queue depth | Reveals work waiting to be processed |
| Retries | Helps identify unstable dependencies or problematic retry policies |

No single metric tells the whole story. For example, low average latency can hide severe tail latency, so p95 and p99 latency are often useful.

## 12.5 Queueing theory and overload

As a service approaches its practical capacity, small increases in incoming load can cause disproportionately large increases in waiting time.

A simple model from queueing theory illustrates the effect.

For an idealized M/M/1 queue:

\[
\rho = \frac{\lambda}{\mu}
\]

where:

- \(\lambda\) is the average arrival rate.
- \(\mu\) is the average service rate.
- \(\rho\) is utilization.

The model's expected time in the system is:

\[
W = \frac{1}{\mu-\lambda}
\]

This formula assumes a single server, Poisson arrivals, exponential service times, and a stable system with \(\lambda < \mu\). Real production systems often violate some of these assumptions.

The intuition remains valuable: as arrival rate approaches service capacity, waiting time can increase sharply.

Load balancing can distribute work across servers, but it cannot compensate indefinitely for incoming traffic that exceeds the total sustainable capacity.

---

# 13. Common Misconceptions

## Misconception 1: A load balancer sends every request to a different server.

**Incorrect.** The result depends on the algorithm, the protocol, the connection model, and affinity rules.

An L4 balancer commonly maintains backend assignment for a TCP connection. An L7 balancer may select a backend for each HTTP request, but session affinity and connection reuse can influence behavior.

## Misconception 2: L4 cannot work with HTTPS.

**Incorrect.** L4 balancers can distribute encrypted TCP traffic without decrypting HTTP.

What they cannot ordinarily do in that configuration is inspect encrypted HTTP paths and headers.

## Misconception 3: Round Robin guarantees equal workload.

**Incorrect.** It distributes assignments evenly, not necessarily computational work.

A few expensive requests can produce an uneven workload even when request counts are equal.

## Misconception 4: Least Connections always chooses the least busy server.

**Incorrect.** It chooses the server with the fewest active connections according to its state.

Connection count is only a proxy for workload. CPU, memory, request duration, and connection activity may differ substantially.

## Misconception 5: Weighted Round Robin dynamically reacts to load.

**Not by itself.** It distributes traffic according to configured weights. If the weights remain fixed, the algorithm does not automatically know that a server has become overloaded.

## Misconception 6: IP Hash guarantees session persistence.

**Incorrect.** A stable mapping can change when the backend pool or hashing configuration changes. Backend failure also cannot recover session state that existed only in the failed server's memory.

## Misconception 7: A load balancer automatically provides high availability.

**Incorrect.** The load balancer, backend pool, network path, DNS or frontend mechanism, and shared dependencies all require appropriate availability design.

## Misconception 8: More backend instances always mean more throughput.

**Incorrect.** A shared database, external API, queue, or other bottleneck may cap throughput. Adding instances can increase contention or resource consumption.

## Misconception 9: L7 is always better than L4.

**Incorrect.** L7 is more application-aware, but that capability comes with additional processing and configuration. Use the level of inspection that the system actually needs.

---

# 14. Choosing the Right Algorithm

## 14.1 Decision table

| Situation | Good starting point | Reason |
|---|---|---|
| Similar servers, uniform short requests | Round Robin | Simple and predictable |
| Servers have different capacities | Weighted Round Robin | Allocates different traffic shares |
| Long-lived connections with variable durations | Least Connections | Accounts for existing connections |
| Different server capacities and long-lived connections | Weighted Least Connections | Combines connection counts and relative capacity |
| Latency-sensitive workload with reliable measurements | Least Response Time or another latency-aware policy | Uses observed performance |
| Application requires backend affinity | Cookie-based affinity or an appropriate hash strategy | Helps preserve backend assignment |
| Raw TCP or UDP service | An appropriate L4 policy | Avoids unnecessary application inspection |
| HTTP APIs with different routing destinations | L7 routing plus a suitable backend algorithm | Supports application-aware routing |

These are starting points, not universal prescriptions. The best algorithm depends on what the workload consumes, the quality of available measurements, and the behavior required by the application.

## 14.2 A practical selection process

Ask these questions in order:

**1. What are you balancing?**

- HTTP requests?
- TCP connections?
- UDP datagrams?
- Long-lived WebSocket connections?

**2. Are the backends equivalent?**

If not, consider weights or capacity-aware policies.

**3. Are requests approximately uniform in cost?**

If yes, Round Robin may be sufficient.

If not, consider connection-aware or latency-aware routing, and investigate whether the chosen metric reflects actual workload.

**4. Does the application need session affinity?**

If possible, design the application so that any eligible instance can process the request. If affinity is necessary, select a mechanism that matches the session model.

**5. What happens when a backend fails?**

Define health checks, timeouts, retry rules, and client recovery behavior.

**6. How will you know the algorithm is working?**

Monitor latency percentiles, errors, backend utilization, and traffic distribution. Test with representative workloads rather than relying solely on theoretical calculations.

---

# 15. Implementation: Simulating Algorithms

A small simulation is an effective way to verify that you understand the algorithms.

The following Python example implements:

- Round Robin
- Weighted Round Robin using a simple repeating schedule
- Least Connections
- Weighted Least Connections
- Simplified Least Response Time

It is a teaching simulator, not a production load balancer. It models backend selection and connection counts; it does not implement sockets, health probes, TLS, retries, or real latency measurement.

## 15.1 Python implementation

```python
from dataclasses import dataclass


@dataclass
class Server:
    name: str
    weight: int = 1
    connections: int = 0
    response_time: float = 1.0


class LoadBalancer:
    def __init__(self, servers):
        if not servers:
            raise ValueError("At least one server is required")

        if any(s.weight <= 0 for s in servers):
            raise ValueError("Server weights must be positive")

        self.servers = servers
        self.rr_index = 0

        # Teaching implementation of weighted round robin.
        self.weighted_schedule = [
            s
            for s in servers
            for _ in range(s.weight)
        ]
        self.wrr_index = 0

    def round_robin(self):
        server = self.servers[self.rr_index]
        self.rr_index = (
            self.rr_index + 1
        ) % len(self.servers)
        return server

    def weighted_round_robin(self):
        server = self.weighted_schedule[self.wrr_index]
        self.wrr_index = (
            self.wrr_index + 1
        ) % len(self.weighted_schedule)
        return server

    def least_connections(self):
        return min(
            self.servers,
            key=lambda s: s.connections
        )

    def weighted_least_connections(self):
        return min(
            self.servers,
            key=lambda s: s.connections / s.weight
        )

    def least_response_time(self):
        return min(
            self.servers,
            key=lambda s: (
                s.connections * s.response_time
            )
        )


servers = [
    Server("A", weight=4, connections=12,
           response_time=20),
    Server("B", weight=2, connections=4,
           response_time=10),
    Server("C", weight=1, connections=1,
           response_time=5),
]

lb = LoadBalancer(servers)

print("Round Robin:",
      lb.round_robin().name)

print("Weighted Round Robin:",
      lb.weighted_round_robin().name)

print("Least Connections:",
      lb.least_connections().name)

print("Weighted Least Connections:",
      lb.weighted_least_connections().name)

print("Least Response Time:",
      lb.least_response_time().name)
```

### Important implementation details

1. The weighted schedule is deliberately simple. Some servers may receive consecutive assignments, and real implementations may use smoother scheduling.
2. Least Connections selects the server with the smallest connection count.
3. Weighted Least Connections uses `connections / weight`.
4. Least Response Time uses the simplified teaching score `connections * response_time`.
5. Python's `min()` returns the first minimum encountered when scores tie. This implementation does not implement the round-robin tie-breaking policy discussed earlier.
6. The example uses static measurements. A production implementation must update connection counts and latency estimates as traffic changes.

## 15.2 Complexity analysis

Let \(N\) be the number of eligible backend servers.

| Algorithm | Simplified selection complexity | Notes |
|---|---|---|
| Round Robin | \(O(1)\) | Maintains a rotating index |
| Weighted Round Robin | \(O(1)\) per selection in this implementation | Schedule construction requires \(O(\sum w_i)\) space |
| Least Connections | \(O(N)\) | Examines every backend |
| Weighted Least Connections | \(O(N)\) | Calculates normalized scores |
| Least Response Time | \(O(N)\) | Calculates scores for every backend |

These complexities describe the teaching implementation. Production systems may use heaps, trees, optimized scheduling structures, or other techniques.

A precomputed weighted schedule can also consume excessive memory when weights are large, so it is not always the right implementation.

---

# 16. Interview Questions and Answers

## Q1. What is a load balancer?

A load balancer distributes incoming traffic among eligible backend instances. It can improve scalability and availability when combined with suitable health checks, redundancy, and application design.

## Q2. What is the difference between L4 and L7 load balancing?

L4 primarily uses transport-level information, such as IP addresses, ports, and TCP/UDP state. L7 can inspect application-level information, such as HTTP paths, headers, and cookies, when that information is available.

## Q3. Can an L4 load balancer route `/api/v1/orders` to the Order service?

Not based on the HTTP path alone. An ordinary L4 balancer does not parse the HTTP request path. An L7 component can use that path to make the routing decision.

## Q4. Which is faster, L4 or L7?

L4 often has lower processing overhead because it avoids application-level parsing. However, actual performance depends on the implementation, traffic, hardware, TLS configuration, and other factors.

## Q5. What is the weakness of Round Robin?

It distributes assignments in a cycle without considering current workload or request cost. A server may receive expensive requests even when it is already overloaded.

## Q6. How does Weighted Round Robin improve on Round Robin?

It assigns different traffic shares according to configured backend weights. It is useful when servers have different capacities, but it does not inherently adapt to current load.

## Q7. Why can IP Hash cause uneven traffic distribution?

Many users may share one public IP through a proxy or NAT. Hashing only that IP can map many users to the same backend. Backend changes can also remap clients.

## Q8. Compare Least Connections and Weighted Least Connections.

Least Connections selects the backend with the fewest active connections. Weighted Least Connections compares connection counts relative to configured capacity weights, using a ratio such as `connections / weight`.

## Q9. What is TTFB?

Time to First Byte is the time from a request being initiated at a measurement point until the first response byte is received. It is different from the time required to receive the complete response.

## Q10. Is Least Response Time a standardized formula?

No. It describes a family of latency-aware selection strategies. The exact formula, averaging method, tie-breaking rules, and metrics depend on the implementation.

## Q11. What happens if a backend fails while a client has an active TCP connection?

The connection may fail or reset. The balancer can stop sending new connections to the unhealthy backend, but it cannot generally transfer the existing connection and its in-memory state seamlessly to another server.

## Q12. How do you prevent a load balancer from becoming a single point of failure?

Deploy redundant load-balancing instances or use a highly available managed frontend, with an appropriate failover mechanism. Also consider shared network dependencies and the availability of backend services.

## Q13. Does session affinity guarantee that a user's session will survive a server crash?

No. Affinity can preserve routing under normal conditions, but it cannot restore session state that existed only in the failed backend's memory. Shared or durable session storage may be needed.

## Q14. Why can adding more application servers make a system slower?

If a shared dependency such as a database is already saturated, more application instances can generate additional concurrent requests and contention. The bottleneck must be identified before scaling.

## Q15. How would you choose a load-balancing algorithm for a WebSocket chat application?

Start by considering long-lived connection duration, connection counts, server capacities, and the need to recover sessions after disconnection. Least Connections or Weighted Least Connections may be useful, but the application must also handle reconnects and maintain any required message or session state independently of the balancer.

---

# 17. Quick Revision Sheet

## Core concepts

- **Load balancer:** Distributes traffic among eligible backend instances.
- **Horizontal scaling:** Adds instances to increase capacity.
- **L4:** Uses transport-level information; suitable for TCP/UDP and TLS passthrough.
- **L7:** Can route using HTTP paths, hosts, headers, and cookies when application traffic is available for inspection.
- **Health checks:** Determine whether backends meet configured eligibility criteria.
- **Session affinity:** Attempts to keep related traffic associated with a backend.
- **High availability:** Requires redundancy and appropriate failure-handling design.

## Algorithm summary

| Algorithm | Decision rule | Main weakness |
|---|---|---|
| Round Robin | Select the next backend in a cycle | Ignores capacity and request cost |
| Weighted Round Robin | Distribute assignments according to weights | Does not inherently adapt to live load |
| IP Hash | Map a client IP through a hash | NAT, proxies, and backend changes can disrupt distribution |
| Least Connections | Select the fewest active connections | Connections do not always represent work |
| Weighted Least Connections | Minimize connections relative to weight | Depends on meaningful weights and connection counts |
| Least Response Time | Favor backends with a lower latency-based score | Measurements can be stale or misleading |

## Formulas worth remembering

**Weighted Round Robin — approximate traffic share**

\[
P_i = \frac{w_i}{\sum_j w_j}
\]

**Weighted Least Connections — simplified score**

\[
S_i = \frac{C_i}{w_i}
\]

Select the lowest score.

**Least Response Time — simplified teaching model**

\[
S_i = C_i \times T_i
\]

Select the lowest score, while remembering that actual implementations may use different latency-aware policies.

## The final mental model

Think of load balancing as three separate questions:

1. **Eligibility:** Which backends are healthy and allowed to receive traffic?
2. **Selection:** Which eligible backend should receive the next connection or request?
3. **Resilience:** What happens when the selected backend becomes slow, overloaded, or unavailable?

L4 and L7 describe the information available for routing. The algorithms describe how a backend is selected. Health checks, timeouts, retries, redundancy, and application design determine how well the overall system handles failure.

Understanding all three dimensions is what turns knowledge of load balancing from a list of definitions into practical distributed-systems engineering.

---
