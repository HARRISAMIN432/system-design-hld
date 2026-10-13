# Chapter 3 — API Gateway vs Load Balancer

> Personal study notes. Goal: know exactly what each component does, where it sits in a real architecture, and answer the classic interview questions around them.

---

## Table of Contents

1. [Recap and Motivation](#1-recap-and-motivation)
2. [What Is an API Gateway](#2-what-is-an-api-gateway)
3. [API Gateway Capabilities](#3-api-gateway-capabilities)
4. [Service Discovery](#4-service-discovery)
5. [What Is a Load Balancer](#5-what-is-a-load-balancer)
6. [API Gateway vs Load Balancer in One Table](#6-api-gateway-vs-load-balancer-in-one-table)
7. [Regions and Availability Zones](#7-regions-and-availability-zones)
8. [End-to-End Request Flow](#8-end-to-end-request-flow)
9. [Failure Scenarios](#9-failure-scenarios)
10. [Classic Interview Questions](#10-classic-interview-questions)
11. [Quick Revision Sheet](#11-quick-revision-sheet)

---

## 1. Recap and Motivation

We now have a microservice application: User, Payment, Order and other services. A real user on a phone or laptop (the **client**) wants to update their profile. Questions:

- Does the client know the API endpoint of each internal service? **No.**
- Does the client know which microservice handles profile updates? **No.**
- Should we expose all that backend structure to clients? **Definitely not.**

The client simply taps "Update profile" and "Save". Something in the middle must translate that action into a call to the right service. That something is the **API Gateway**.

---

## 2. What Is an API Gateway

An API Gateway is an **interface between clients and microservices** and the **single entry point for all API requests**.

```
 Mobile ─┐                         ┌──► User Service
 Laptop ─┼──►  API Gateway  ───────┼──► Payment Service
 Tablet ─┘   (routes by endpoint)  └──► Order Service
```

**Core job:** accept the client's request and **route it to the correct backend microservice**. In one line: *the API Gateway tells us which microservice the user wants to talk to.*

Example: the client calls `/api/v1/update-profile`. The gateway understands this endpoint belongs to the User service and forwards the request there.

Benefits:

- Clients never talk to microservices directly.
- Backend structure stays hidden and can change freely.
- Cross-cutting concerns live in one place.

---

## 3. API Gateway Capabilities

### 3.1 API Composition

Different devices need different data on the same page.

- On a **phone**, the "My Orders" page may show only product details and invoice details.
- On a **laptop**, the same page may also show ratings and reviews.

The gateway can expose fewer or more APIs depending on the device and **compose** responses by calling multiple services:

| Data needed | Source service |
|---|---|
| Product details | Product service |
| Invoice details | Invoice service |
| Ratings and reviews | Rating and Review service |

API composition can also be implemented with **GraphQL** (covered later).

### 3.2 Authentication and Authorization

The gateway handles authentication and authorization centrally. It accepts only authorized clients' requests and allows them to use resources.

**OAuth 2.0 style flow:**

1. Client requests an **access token** from the **Auth server**.
2. Auth server issues the token.
3. Client sends the API request **with the access token** to the gateway.
4. Gateway **validates the token** with the auth server.
5. Valid → route to the appropriate microservice. Invalid → reject.

```
 Client ──(1) request token──► Auth Server
 Client ◄─(2) access token──── Auth Server
 Client ──(3) API call + token─► API Gateway ──(4) validate──► Auth Server
                                      │
                           (5) valid? route : reject
```

**Benefit:** instead of implementing authentication inside every microservice, one centralized auth server and gateway fulfil the requirement.

### 3.3 Service Discovery

Covered in detail next.

### 3.4 Other Commonly Associated Features

(General industry knowledge, not core to this chapter, but worth knowing.)

- Rate limiting and throttling
- Request/response transformation
- Logging, metrics and tracing at one choke point
- Caching of responses
- SSL termination

---

## 4. Service Discovery

Microservices scale horizontally, so one service has **many instances**, each on a different machine with its own IP and port.

| Instance | IP | Port |
|---|---|---|
| user-1 | x.y.z | 4000 |
| user-2 | a.b.c | 4001 |
| order-1 | … | … |

Where is this information stored, and who maintains it? **The service registry (service discovery).**

**How it works:**

1. Whenever a new instance starts (say `user-3`), it **registers** itself with service discovery.
2. Some instances may be down or inactive (say `order-2`, `order-3`).
3. When a request for the Order service arrives, the gateway **asks service discovery**: "Which Order instance is free?"
4. Service discovery replies: "`order-1` is available, IP X.Y.Z, port 402."
5. The gateway routes the request.

A well-known tool for this is **Eureka Server**; Consul and Kubernetes' built-in service discovery are other examples.

---

## 5. What Is a Load Balancer

A load balancer **distributes traffic among multiple instances of the same service**.

- **API Gateway** answers: *which microservice does the user want?*
- **Load Balancer** answers: *which instance of that microservice should handle this request?*

Example: the gateway has decided the request is for Payment. Payment has instances P1, P2, P3. Say P3 is already busy and P2 is inactive. The load balancer picks **P1** and forwards the request.

A load balancer decides using:

- **Health checks** (is the instance alive?)
- **Current traffic / load**
- **Availability**
- A **balancing algorithm** (see Chapter 4)

> Note: some cloud load balancers (e.g. AWS Application Load Balancers) can also route by path to the right service, overlapping with gateway functionality. They tend to be costly, so teams often still use a dedicated gateway.

---

## 6. API Gateway vs Load Balancer in One Table

| Aspect | API Gateway | Load Balancer |
|---|---|---|
| Decides | Which **service** | Which **instance** of the service |
| Understands | API endpoints, auth tokens, request semantics | Instance health and load |
| Typical scope | **Region level** | **Availability-zone level** |
| Extras | Auth, composition, discovery integration | Health checks, algorithms |
| Count | Often one per region | Often one (or more) per service per zone |

---

## 7. Regions and Availability Zones

To understand the interview questions, you need this vocabulary.

**Analogy:** think of a US state as a **region**, and the cities in it as **availability zones (AZs)**.

```
 Region: Massachusetts
   ├── AZ-1: Boston     (dedicated data center)
   └── AZ-2: Cambridge  (dedicated data center)

 Region: New York  (another region)
```

- A **region** is a large geographic area hosting our deployment.
- An **availability zone** is an isolated data center within that region.

**Why they exist:**

- If **one AZ** goes down, shift traffic to the other AZ in the same region.
- If **all AZs** in a region go down, **switch the region**.
- Users get routed to the **nearest** region, which **reduces latency**.

---

## 8. End-to-End Request Flow

Example: a user clicks **Place Order** on an e-commerce site. The request is `POST /api/v1/order`.

Suppose the application is deployed in two regions: **California** (API Gateway A) and **Massachusetts** (API Gateway B).

```
 1. User clicks "Place Order"
 2. Browser has a domain name, needs an IP address
 3. Browser asks DNS
 4. DNS-based load balancer returns the gateway IP of the CLOSEST region
 5. Browser sends HTTP request to that API Gateway
 6. Gateway asks Service Discovery:
        "Closest healthy availability zone for this user?"
        "Address of that zone's load balancer?"
 7. Gateway forwards request to that Load Balancer
 8. Load Balancer picks a healthy, free Order service instance (O1/O2/O3)
 9. Order service handles the request
```

Which component works at which level?

| Component | Level |
|---|---|
| DNS-based load balancer | Global / region selection |
| API Gateway | **Region level** |
| Service Discovery | Tells gateway the closest healthy AZ and its LB |
| Load Balancer | **Availability-zone level**, picks the instance |

---

## 9. Failure Scenarios

| Failure | What happens |
|---|---|
| One instance is down | Load balancer skips it using health checks |
| One availability zone (e.g. Los Angeles) is down | Service discovery points the gateway to the next closest AZ (e.g. San Francisco) |
| All AZs in the region are down | DNS-based load balancer returns the **other region's** gateway IP (Gateway B). A new HTTP request goes there |
| User is far from a region | DNS routes them to the nearest one to **cut latency** |

---

## 10. Classic Interview Questions

### Q1. If the API Gateway is a single entry point, how does it handle millions of requests?

Yes, it is a single *logical* entry point, because thousands of microservices may sit behind it. But it does not have to be a single *machine*:

- Gateways are configured **per region** (Gateway A, Gateway B, …).
- A **DNS-based load balancer** spreads users across regions.
- Within a region, the gateway itself can run as **multiple instances**.
- Traffic from India is routed to the region nearest India, and so on.

So the gateway scales horizontally and traffic is split geographically.

### Q2. Is the API Gateway truly a single entry point?

Logically yes (from the client's perspective there is one front door). Physically no: there are multiple gateways across regions behind DNS-based load balancing.

### Q3. Which comes first: API Gateway or Load Balancer?

It depends which load balancer you mean:

1. **DNS-based (global) load balancer** → comes **first**.
2. **API Gateway** → next, at region level.
3. **Service-level load balancer** → after the gateway, at AZ level, in front of service instances.

So the order is: **DNS load balancer → API Gateway → (service discovery lookup) → Load Balancer → Microservice instance.**

### Q4. Why do we need both?

They solve different problems: the gateway handles *routing by purpose and cross-cutting concerns*; the load balancer handles *distribution across identical instances*.

---

## 11. Quick Revision Sheet

- **API Gateway** = single entry point; routes to the correct **microservice**; handles composition, auth, service-discovery integration.
- **Service Discovery** = registry of live instances with IPs and ports; tells the gateway where to go.
- **Load Balancer** = distributes traffic across **instances** of one service using health and load.
- **Region** = large area; **AZ** = data center inside it.
- Gateway operates at **region** level; LB at **AZ** level.
- **Order of components:** DNS load balancer → API Gateway → Load Balancer → Instance.
- Failover: instance → AZ → region.

---

**Previous:** [Chapter 2](../Chapter-02-Monolith-vs-Microservices/README.md) · **Next:** [Chapter 4 — Load Balancer Types and Algorithms](../Chapter-04-Load-Balancing-Algorithms-and-Types/README.md)
