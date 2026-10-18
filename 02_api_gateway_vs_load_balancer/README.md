# Chapter 3 — API Gateway vs Load Balancer

> **Goal:** Understand API gateways, load balancers, service discovery, and multi-region routing from first principles. Learn how these components work together, how they behave under failure, and how to make sound architectural decisions in production systems.

## Table of Contents

- [1. The Problem: Why Do We Need These Components?](#1-the-problem-why-do-we-need-these-components)
- [2. Understanding an API Gateway](#2-understanding-an-api-gateway)
- [3. API Gateway Responsibilities](#3-api-gateway-responsibilities)
- [4. Service Discovery](#4-service-discovery)
- [5. Understanding Load Balancers](#5-understanding-load-balancers)
- [6. API Gateway vs Load Balancer](#6-api-gateway-vs-load-balancer)
- [7. Regions and Availability Zones](#7-regions-and-availability-zones)
- [8. How Everything Works Together](#8-how-everything-works-together)
- [9. Failure Scenarios and Resilience](#9-failure-scenarios-and-resilience)
- [10. Security and Production Considerations](#10-security-and-production-considerations)
- [11. Common Architectural Patterns](#11-common-architectural-patterns)
- [12. Common Mistakes and Misconceptions](#12-common-mistakes-and-misconceptions)
- [13. Classic Interview Questions](#13-classic-interview-questions)
- [14. Practical Exercises](#14-practical-exercises)
- [15. Quick Revision Sheet](#15-quick-revision-sheet)

---

# 1. The Problem: Why Do We Need These Components?

Before understanding API gateways and load balancers, we need to understand the problems they solve.

## 1.1 From a Monolith to Microservices

Imagine an e-commerce application that initially runs as a monolith.

```text
                 ┌─────────────────────┐
                 │     Web Client      │
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │  Monolithic Backend │
                 │                     │
                 │  User Management    │
                 │  Orders             │
                 │  Payments           │
                 │  Products           │
                 └─────────────────────┘
```

The client sends requests to one backend application. The application handles all the business functionality.

As the system grows, the engineering team separates it into independently deployable services.

```text
                    ┌───────────────┐
                    │  Web Client   │
                    └───────┬───────┘
                            │
             ┌──────────────┼──────────────┐
             ▼              ▼              ▼
       ┌──────────┐   ┌──────────┐   ┌──────────┐
       │  User    │   │  Order   │   │ Payment  │
       │ Service  │   │ Service  │   │ Service  │
       └──────────┘   └──────────┘   └──────────┘
```

This introduces several new problems.

**Problem 1: Service routing**

The client needs a way to reach the correct service for each operation.

**Problem 2: Service instances**

A service may run on multiple machines or containers. The system must decide which instance receives a request.

**Problem 3: Authentication and security**

The system needs a consistent way to authenticate callers, enforce access policies, and protect backend services.

**Problem 4: Scalability and availability**

Traffic must be distributed across healthy instances, and the system must continue operating when individual components fail.

**Problem 5: Infrastructure changes**

Instances are frequently created, destroyed, restarted, or moved. Their IP addresses and ports may change.

Two important components address different parts of these problems:

- **API Gateway:** Provides a managed entry point into backend APIs and applies API-level routing and policies.
- **Load Balancer:** Distributes traffic across eligible destinations, commonly multiple instances of a service.

Service discovery helps the system locate those destinations.

## 1.2 The Two Fundamental Questions

Keep these two questions in mind throughout the chapter:

1. **Where should this request go?** This is the routing problem.
2. **Which healthy destination should handle it?** This is the traffic-distribution problem.

An API gateway often helps answer the first question. A load balancer often helps answer the second.

However, these are conceptual responsibilities, not strict product boundaries. Some gateways perform load balancing, and some load balancers route requests using HTTP paths, hostnames, or headers.

Understanding the responsibility matters more than memorizing the component's name.

---

# 2. Understanding an API Gateway

## 2.1 What Is an API Gateway?

An **API Gateway** is an API-facing entry point that receives client requests and applies policies such as routing, authentication, rate limiting, and request transformation before forwarding requests to backend services.

It hides much of the internal service topology from clients.

Consider these public endpoints:

```text
GET  /api/v1/users/42
GET  /api/v1/orders/9001
POST /api/v1/payments
```

The gateway can route them as follows:

| Public endpoint | Backend destination |
|---|---|
| `/api/v1/users/42` | User Service |
| `/api/v1/orders/9001` | Order Service |
| `/api/v1/payments` | Payment Service |

The client does not need to know the internal hostnames, ports, or number of instances behind each service.

## 2.2 Basic Architecture

```text
                         PUBLIC NETWORK
                               │
                               ▼
                     ┌──────────────────┐
                     │   API Gateway    │
                     │                  │
                     │ Routing          │
                     │ Authentication   │
                     │ Rate Limiting    │
                     └────────┬─────────┘
                              │
               ┌──────────────┼──────────────┐
               ▼              ▼              ▼
         ┌──────────┐   ┌──────────┐   ┌──────────┐
         │  User    │   │  Order   │   │ Payment  │
         │ Service  │   │ Service  │   │ Service  │
         └──────────┘   └──────────┘   └──────────┘
```

This is a simplified logical view. A production deployment may include several gateway replicas, load balancers, service meshes, and additional network components.

## 2.3 What Happens When a Request Arrives?

Suppose a client sends:

```http
GET /api/v1/orders/9001 HTTP/1.1
Host: api.example.com
Authorization: Bearer <access-token>
```

A typical gateway processes the request through these stages:

1. **Receive:** Accept the incoming HTTP request.
2. **Match:** Identify the applicable route using the HTTP method, path, hostname, or other configured properties.
3. **Validate:** Apply relevant authentication, authorization, and request-validation policies.
4. **Resolve destination:** Determine the backend destination through static configuration, service discovery, or an internal routing mechanism.
5. **Forward:** Send the request to the selected backend.
6. **Process response:** Apply configured response transformations or other policies.
7. **Return:** Send the resulting HTTP response to the client.

The precise order of policy execution depends on the gateway implementation and configuration.

## 2.4 Why Not Let Clients Call Every Service Directly?

Direct access is possible, but it introduces several architectural problems.

| Without a gateway | With a gateway |
|---|---|
| Clients may need multiple service URLs. | Clients can use a stable API entry point. |
| Backend topology can leak into clients. | Internal service locations can remain private. |
| Cross-cutting policies may be duplicated. | Shared API policies can be centralized. |
| Backend changes may require client changes. | Backend implementations can often change without changing public routes. |
| Public access must be managed across services. | Public exposure can be restricted to the gateway. |

A gateway is not mandatory for every microservice architecture. Small systems may work perfectly well with direct service access or a simpler reverse proxy.

The gateway becomes particularly useful when a system has many APIs, multiple client types, centralized API policies, or a need to hide backend topology.

---

# 3. API Gateway Responsibilities

An API gateway is more than a URL router. Its value comes from the additional policies and transformations it can enforce at the API boundary.

## 3.1 Request Routing

Routing maps an incoming request to a backend destination.

Example:

```text
POST /api/v1/orders
        │
        ▼
    API Gateway
        │
        ▼
    Order Service
```

A route can be selected using:

- URL path
- HTTP method
- Hostname
- HTTP headers
- Version or other configured request attributes

For example:

```text
/api/v1/users/*     → User Service
/api/v1/orders/*    → Order Service
/api/v1/payments/*  → Payment Service
```

**Important:** Selecting the backend service does not necessarily determine which replica will process the request. That is a separate concern.

## 3.2 Authentication and Authorization

These concepts are related but different.

- **Authentication:** Who is making the request?
- **Authorization:** Is that caller allowed to perform the requested operation?

Consider a user who wants to access an order.

```http
GET /api/v1/orders/9001
Authorization: Bearer eyJ...
```

The gateway might validate the access token and reject an invalid or expired token.

However, a valid token does not automatically mean that the user can access order `9001`.

The system must also establish whether that order belongs to the user or whether the user has an appropriate administrative permission.

A common design is:

1. The client authenticates with an identity provider or authorization server.
2. The authorization server issues an access token.
3. The client includes that token in API requests.
4. The gateway validates the token, either locally or through an appropriate verification mechanism.
5. The gateway forwards the request if the gateway-level policies pass.
6. The backend enforces business-specific permissions and resource ownership where necessary.

```text
Client
  │
  │ 1. Obtain access token
  ▼
Authorization Server
  │
  │ 2. Issue token
  ▼
Client
  │
  │ 3. API request + token
  ▼
API Gateway
  │
  │ 4. Validate token and API policies
  ▼
Order Service
  │
  │ 5. Check order ownership and business permissions
  ▼
Response
```

The gateway does not necessarily contact the authorization server on every request. It may validate signed tokens locally using cached or published verification keys.

**Security principle:** Centralize common authentication checks, but do not assume that passing the gateway means every business operation is authorized.

## 3.3 Rate Limiting and Throttling

A public API may receive excessive traffic from a single client or API consumer.

A gateway can limit requests according to policies such as:

```text
Free API consumer:
    100 requests per minute

Premium API consumer:
    1,000 requests per minute
```

These numbers are illustrative.

Rate limiting determines whether additional requests are allowed under a configured policy. Throttling controls or slows request processing when limits or capacity constraints are reached.

An exceeded limit commonly results in:

```http
HTTP/1.1 429 Too Many Requests
```

A production implementation must also define the scope of the limit:

- Per user
- Per API key
- Per tenant
- Per IP address
- Per endpoint
- Across the entire system

Distributed gateways may need shared counters or another coordination strategy to enforce global limits accurately.

## 3.4 Request and Response Transformation

Different clients or legacy systems may use different API formats.

For example, a client sends:

```json
{
  "firstName": "Harris",
  "lastName": "Amin"
}
```

A gateway might transform the payload into the format expected by a legacy backend.

Similarly, it can rewrite paths, modify headers, or translate certain protocol details when supported.

However, extensive business logic and complicated transformations inside the gateway create tight coupling. Prefer keeping business rules in the appropriate service.

## 3.5 API Composition

Suppose a dashboard needs information from three services:

- User Service: user profile
- Order Service: recent orders
- Payment Service: payment summary

Without composition, the client may need to issue three requests.

```text
Client ──► User Service
Client ──► Order Service
Client ──► Payment Service
```

An API gateway or a dedicated backend-for-frontend (BFF) can aggregate those requests:

```text
                   ┌─────────────────┐
Client ───────────►│ Gateway / BFF   │
                   └────────┬────────┘
                            │
                 ┌──────────┼──────────┐
                 ▼          ▼          ▼
               User       Order      Payment
               Service    Service    Service
                 │          │          │
                 └──────────┼──────────┘
                            ▼
                    Combined response
                            │
                            ▼
                          Client
```

A combined response might look like:

```json
{
  "user": {
    "name": "Harris"
  },
  "recentOrders": [
    {
      "id": "9001",
      "status": "shipped"
    }
  ],
  "paymentSummary": {
    "currency": "USD",
    "totalPaid": 240
  }
}
```

### The trade-off

Composition reduces client-side coordination and can reduce the number of client-to-server round trips.

However, it introduces additional dependencies:

- If one backend is slow, the combined response may be delayed.
- If one backend fails, the gateway needs a defined partial-failure policy.
- Parallel backend calls consume resources.
- Combining data can couple the gateway to backend response structures.

For complex aggregation, a dedicated BFF or application-level composition layer may be more appropriate.

GraphQL can also provide a client-oriented data-query interface, but GraphQL and API gateways are not the same thing. GraphQL is an API query language and execution model; it can be deployed behind or alongside a gateway.

## 3.6 Caching

Some API responses change infrequently and can be cached.

For example:

```text
GET /api/v1/products/123
```

A gateway may cache an eligible response to reduce backend traffic and latency.

Caching must respect freshness requirements and authorization boundaries. Responses containing private user data should not be shared across users accidentally.

## 3.7 Observability

The gateway is a useful place to collect API-level information, including:

- Request counts
- Latency
- HTTP status codes
- Rate-limit rejections
- Trace identifiers
- Backend error rates

However, gateway logs alone are insufficient to explain failures deep inside a distributed system. Distributed tracing and service-level metrics are needed to understand the full request path.

## 3.8 SSL/TLS Termination

A gateway commonly terminates incoming HTTPS connections, decrypts the request, and forwards it to backend services.

The gateway-to-backend connection may also use TLS, depending on the security requirements.

Encryption at the public edge does not automatically guarantee encryption throughout the internal network.

---

# 4. Service Discovery

Service discovery addresses a fundamental problem in dynamic environments:

**How does one component find the current network location of another component?**

## 4.1 Why Hardcoded IP Addresses Are Fragile

Suppose the Order Service initially runs at:

```text
10.0.1.10:8080
```

A deployment replaces the container, and the new instance receives:

```text
10.0.3.25:8080
```

If every caller has the old address hardcoded, requests will fail until those callers are reconfigured.

The problem becomes more difficult when a service has dozens or hundreds of replicas.

A more flexible system uses service discovery.

## 4.2 Service Registry

A service registry maintains information about available service instances.

Conceptually:

| Service | Instance | Address | Health |
|---|---|---|---|
| User | U1 | `10.0.1.10:8080` | Healthy |
| User | U2 | `10.0.1.11:8080` | Healthy |
| Order | O1 | `10.0.2.10:8080` | Healthy |
| Order | O2 | `10.0.2.11:8080` | Unhealthy |

These are illustrative addresses.

The registry may also track metadata such as availability zone, version, or other routing attributes.

## 4.3 How Service Discovery Works

A typical registry-based implementation follows this process:

1. A service instance starts.
2. The platform registers it or makes it discoverable.
3. Health information is updated through probes, heartbeats, or registration mechanisms.
4. A consumer resolves the service name.
5. The consumer obtains or uses the available destination information.
6. Traffic is directed to an eligible instance.

The registration mechanism, health checks, and lookup process vary by platform.

```text
                   ┌──────────────────┐
                   │ Service Registry │
                   └────────┬─────────┘
                            │
                ┌───────────┴───────────┐
                │                       │
                ▼                       ▼
          ┌──────────┐            ┌──────────┐
          │ Order O1 │            │ Order O2 │
          └──────────┘            └──────────┘
                ▲
                │
          Consumer resolves
          "Order Service"
```

The diagram is conceptual: the registry need not sit directly in the data path for every request.

## 4.4 Two Service Discovery Models

### Client-side discovery

The consumer obtains a list of service instances and selects a destination itself.

```text
Consumer
   │
   ├── Resolve service ──► Registry
   │                         │
   │◄── Instance list ───────┘
   │
   └── Select instance ──► Order Instance
```

The consumer is responsible for instance selection and may use a client-side load-balancing algorithm.

### Server-side discovery

The consumer sends the request to a stable endpoint, and an intermediary resolves or selects the destination.

```text
Consumer
   │
   ▼
Stable Service Endpoint
   │
   ▼
Load Balancer / Proxy
   │
   ▼
Selected Instance
```

The intermediary might use service discovery to maintain an up-to-date set of eligible destinations.

This is common in managed cloud environments and container platforms.

## 4.5 Service Discovery Is Not the Same as Load Balancing

| Service discovery | Load balancing |
|---|---|
| Finds or tracks service destinations. | Distributes requests across eligible destinations. |
| Answers where instances can be reached. | Decides which destination should receive traffic. |
| Deals with registration, resolution, and endpoint information. | Deals with selection, health, and traffic distribution. |

These responsibilities often work together.

A load balancer may obtain endpoints from a service registry, or a platform may combine discovery and traffic distribution into one system.

## 4.6 Examples of Service Discovery Systems

- **Kubernetes Services and DNS:** Provide stable service names and routes to eligible pods through the cluster networking implementation.
- **Consul:** Provides service discovery and health-checking capabilities.
- **Eureka:** A service registry commonly associated with Spring-based microservice architectures.
- **Cloud service discovery:** Cloud providers offer platform-specific mechanisms for resolving service names and discovering registered targets.

### Important Kubernetes detail

A Kubernetes Service typically provides a stable virtual endpoint. The cluster networking implementation directs traffic to eligible pods.

Applications often do not need to query the Kubernetes API directly for every pod address.

---

# 5. Understanding Load Balancers

## 5.1 What Is a Load Balancer?

A **load balancer** distributes incoming requests or connections across multiple eligible backend destinations according to a configured policy.

Its purpose is to improve availability, resource utilization, and scalability.

Suppose the Order Service has three replicas:

```text
                     ┌──────────────────┐
                     │  Load Balancer   │
                     └─────────┬────────┘
                               │
                   ┌───────────┼───────────┐
                   ▼           ▼           ▼
              ┌────────┐  ┌────────┐  ┌────────┐
              │ Order  │  │ Order  │  │ Order  │
              │   O1   │  │   O2   │  │   O3   │
              └────────┘  └────────┘  └────────┘
```

Instead of directing every request to O1, the load balancer distributes traffic across the available replicas.

## 5.2 Why Multiple Instances?

Consider a service receiving 3,000 requests per second.

Assume, for illustration, that one instance can safely handle 1,000 requests per second for the workload in question.

One instance may be insufficient.

Running multiple replicas allows the service to spread work across available compute resources.

However, capacity is not always perfectly additive. Database contention, shared dependencies, synchronization, network overhead, and uneven request complexity can prevent linear scaling.

The load balancer distributes traffic; it does not automatically create additional capacity. A separate scaling mechanism must add or remove instances when necessary.

## 5.3 Health Checks

A load balancer needs a way to determine which destinations are eligible to receive traffic.

Common health-check mechanisms include:

- TCP connection checks
- HTTP health endpoints
- Application-specific readiness checks

For example:

```http
GET /health/ready
```

A service might return:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{"status":"ready"}
```

If an instance fails its configured health checks, the load balancer may remove it from the eligible destination pool.

When the instance recovers, it can be added back after passing the configured recovery checks.

### Liveness versus readiness

These checks are not interchangeable.

- **Liveness:** Is the application alive, or should it be restarted?
- **Readiness:** Is the application currently ready to accept traffic?

A process can be alive but not ready. For example, it may still be loading configuration or warming a cache.

Health-check design matters because overly aggressive checks can incorrectly remove healthy instances, while poorly designed checks can keep unhealthy instances in rotation.

## 5.4 Load-Balancing Algorithms

The algorithm determines how the load balancer chooses a destination.

| Algorithm | Basic idea | Typical consideration |
|---|---|---|
| Round Robin | Cycle through eligible instances. | Simple when instances and requests have similar costs. |
| Weighted Round Robin | Give some instances a larger share of requests. | Useful when capacities differ. |
| Least Connections | Prefer an instance with fewer active connections. | Helpful for workloads with variable connection durations. |
| Weighted Least Connections | Consider both connection count and configured weight. | Useful for heterogeneous instances. |
| IP Hash | Derive the destination from a client-related hash. | Can provide affinity but may produce uneven distribution. |
| Random / Power of Two Choices | Select a random destination or compare a small sample. | Can be efficient for large pools. |

No algorithm is universally best.

For example, least connections may be more appropriate than round robin for long-lived connections, but connection count is not necessarily a good measure of CPU load or application queue depth.

Some systems use latency, request cost, queue length, or other signals.

The detailed algorithms are covered in the next chapter.

## 5.5 Does a Load Balancer Always Select an Instance?

No.

A load balancer can distribute traffic across different types of destinations:

- Application instances
- Containers or pods
- Virtual machines
- Backend services
- Multiple regional endpoints

The phrase "instance-level load balancing" describes a common use case, not a universal restriction.

A global traffic-management system may distribute traffic between regions, while a regional load balancer distributes traffic across application instances.

## 5.6 Does a Load Balancer Understand HTTP?

It depends on the load balancer's operating mode.

**Layer 4 load balancing**

Operates primarily at the transport layer, using information such as IP addresses, protocols, and TCP or UDP ports.

It does not normally need to inspect HTTP paths.

**Layer 7 load balancing**

Operates at the application layer and can understand HTTP requests.

It may route based on:

- Hostname
- URL path
- HTTP headers
- Other supported request attributes

For example:

```text
api.example.com/users/*  → User backend
api.example.com/orders/* → Order backend
```

This is why a layer 7 load balancer can overlap with some API gateway responsibilities.

The distinction is about supported functionality and policy depth, not merely whether a component can inspect a URL.

---

# 6. API Gateway vs Load Balancer

## 6.1 The Core Difference

An API gateway is primarily an API-management and policy-enforcement layer.

A load balancer is primarily a traffic-distribution mechanism.

Consider a request:

```http
GET /api/v1/orders/9001
```

A possible sequence is:

1. An API gateway determines that the request belongs to the Order API.
2. A load balancer selects an eligible Order Service instance.
3. The instance processes the request.

The responsibilities are separate even if one product performs both.

## 6.2 Comparison Table

| Dimension | API Gateway | Load Balancer |
|---|---|---|
| Primary responsibility | API routing and policies | Traffic distribution |
| Typical routing decision | Which API or backend service? | Which eligible destination? |
| Authentication | Commonly supports integration or enforcement | May support certain authentication or access policies, depending on product |
| Authorization | Can enforce API-level policies | Usually not its primary purpose |
| Rate limiting | Common API capability | May support traffic limits, depending on product |
| Health checks | May use backend health information | Common capability |
| Load distribution | May provide it | Core capability |
| Request transformation | Commonly supported | Supported by some application-layer products |
| API composition | Possible, but not universal | Not usually the primary purpose |
| Service discovery | May integrate with discovery | May integrate with discovery |
| Typical consumers | Web, mobile, partner, and other API clients | Any supported traffic source |
| Must both exist? | No | No |

## 6.3 Can One Component Replace the Other?

Sometimes.

A layer 7 load balancer can route traffic by host or path and may support TLS termination, access rules, and other policies.

An API gateway may also balance traffic across multiple backend endpoints.

For a small application, one component may provide all the required functionality.

For a large platform, separate components may be useful because they have different policy requirements, ownership boundaries, scaling needs, and operational responsibilities.

The correct question is not "Which component is always required?"

It is:

**Which capabilities does this system need, and which component should own each capability?**

## 6.4 Do Not Memorize a Universal Deployment Level

It is tempting to memorize this table:

```text
API Gateway  = Region level
Load Balancer = Availability-zone level
```

That is not a universal architectural rule.

An API gateway can be deployed per region, per cluster, per environment, or as a managed global service. A load balancer can operate globally, regionally, across zones, or within a cluster.

Their placement depends on the deployment and the product.

Think in terms of **responsibility and scope**, not fixed geographical levels.

---

# 7. Regions and Availability Zones

Distributed applications often run across multiple geographic and infrastructure failure domains.

Understanding these terms helps explain availability and disaster recovery.

## 7.1 Region

A region is a geographic area containing cloud infrastructure.

For example, a cloud provider may offer regions in different parts of Europe, Asia, or North America.

Applications deployed in different regions can serve geographically distributed users or provide disaster-recovery capacity.

## 7.2 Availability Zone

An availability zone (AZ) is an isolated infrastructure location within a region, designed to reduce the risk that one localized infrastructure failure affects every zone.

An AZ is not necessarily equivalent to a city. Depending on the provider, it may represent one or more data centers or another provider-defined failure domain.

```text
                    REGION A
        ┌───────────────────────────────┐
        │                               │
        │  ┌───────────┐ ┌───────────┐  │
        │  │   AZ-1    │ │   AZ-2    │  │
        │  │           │ │           │  │
        │  │ Gateway   │ │ Gateway   │  │
        │  │ Services  │ │ Services  │  │
        │  └───────────┘ └───────────┘  │
        │                               │
        └───────────────────────────────┘
                       │
                       │ Cross-region deployment
                       ▼
                    REGION B
        ┌───────────────────────────────┐
        │                               │
        │  ┌───────────┐ ┌───────────┐  │
        │  │   AZ-1    │ │   AZ-2    │  │
        │  │           │ │           │  │
        │  │ Gateway   │ │ Gateway   │  │
        │  │ Services  │ │ Services  │  │
        │  └───────────┘ └───────────┘  │
        │                               │
        └───────────────────────────────┘
```

This is a conceptual deployment. The actual topology depends on the provider and application.

## 7.3 Why Deploy Across Multiple AZs?

Suppose an application runs in only one AZ.

If that zone becomes unavailable, the application may become unavailable even though the rest of the region remains operational.

Deploying across multiple zones allows traffic to move to healthy capacity in another zone, provided that networking, capacity, dependencies, and data access support the failover.

Multi-AZ deployment improves resilience against certain infrastructure failures.

It does not automatically protect against regional outages, application-wide bugs, bad deployments, or failures in shared dependencies.

## 7.4 Why Deploy Across Multiple Regions?

Multi-region deployment can protect against some regional failures and improve latency for users who are geographically distributed.

However, it introduces additional complexity:

- Cross-region networking and latency
- Data replication and consistency
- Regional failover
- Traffic steering
- Disaster-recovery testing
- Capacity planning
- Regional configuration and secret management

A multi-region application is not automatically highly available. Its data stores and critical dependencies must also support the intended recovery strategy.

## 7.5 Global Traffic Management

A global traffic-management layer directs clients toward appropriate regional endpoints.

Possible mechanisms include:

- DNS-based routing
- Anycast networking
- Global application load balancing
- Provider-specific edge routing

The implementation determines how quickly traffic can move after a failure.

### DNS-based routing caveat

DNS-based routing can return different addresses based on geography, health information, or configured policies.

However, DNS caching and client resolver behavior can delay the effect of a routing change. DNS failover is not necessarily instantaneous.

A global traffic manager is also not always a DNS load balancer. Some systems steer traffic through a global network or proxy instead.

---

# 8. How Everything Works Together

Let's combine the concepts into a realistic request flow.

## 8.1 Example: Placing an Order

A customer clicks **Place Order** in an e-commerce application.

The client sends:

```http
POST /api/v1/orders HTTP/1.1
Host: api.example.com
Authorization: Bearer <access-token>
Content-Type: application/json

{
  "productId": "P123",
  "quantity": 2
}
```

Assume the application has:

- A global traffic-management layer
- An API gateway in each active region
- Multiple Order Service instances
- Service discovery or platform-managed endpoint resolution

A possible architecture is:

```text
                         Client
                           │
                           ▼
                  Global Traffic Manager
                           │
                           ▼
                  Regional API Gateway
                           │
                           ▼
                   Order Service Endpoint
                           │
                           ▼
                    Load Distribution
                           │
                ┌──────────┼──────────┐
                ▼          ▼          ▼
              Order O1   Order O2   Order O3
```

The diagram shows the logical responsibilities. A real deployment might merge the service endpoint and load-distribution functions or place additional load balancers before the gateway.

## 8.2 Step-by-Step Request Lifecycle

### Step 1: The client resolves the public hostname

The browser or mobile client needs to reach `api.example.com`.

DNS resolves the hostname to an address or to infrastructure that can direct the client toward an appropriate endpoint.

DNS resolution does not necessarily choose the final backend instance.

### Step 2: Global traffic management selects a destination

Depending on the architecture, a global traffic manager directs the request toward a suitable regional endpoint.

Selection might consider geographic proximity, endpoint health, configured routing policies, or other signals.

The nearest region is not always the best destination. Capacity, availability, data residency, and application dependencies may also matter.

### Step 3: The regional gateway receives the request

The gateway identifies the route:

```text
POST /api/v1/orders → Order Service
```

It applies configured security policies, such as token validation and request limits.

### Step 4: The backend destination is resolved

The gateway or an underlying proxy uses configuration, service discovery, or a platform-managed endpoint to find the Order Service destination.

This does not necessarily involve querying a registry for every request. Destination information may already be available locally and updated asynchronously.

### Step 5: Traffic distribution selects an eligible destination

A load balancer or proxy chooses an eligible Order Service instance.

It considers the configured balancing policy and health information.

For example:

```text
Order O1 → Healthy
Order O2 → Unhealthy
Order O3 → Healthy
```

O2 should be excluded from normal traffic while it is considered unhealthy.

### Step 6: The service processes the request

The selected Order Service validates the request, applies business rules, and interacts with the required dependencies.

For an order-creation operation, this might include inventory checks, database writes, or communication with other services.

The exact business workflow depends on the application.

### Step 7: The response travels back

The service returns a response through the intermediary components to the client.

For example:

```http
HTTP/1.1 201 Created
Content-Type: application/json

{
  "orderId": "9001",
  "status": "created"
}
```

A `201 Created` response indicates successful resource creation in this example.

## 8.3 Who Does What?

| Component | Responsibility in this flow |
|---|---|
| DNS | Resolves the public hostname. |
| Global traffic manager | Directs traffic toward an appropriate regional endpoint. |
| API gateway | Applies API policies and routes the request. |
| Service discovery | Makes backend destinations discoverable. |
| Load balancer or proxy | Selects an eligible backend destination. |
| Order Service | Executes the business operation. |
| Database and other dependencies | Persist data and support business processing. |

Not every system requires every component as a separate product.

The important distinction is between the **logical responsibilities** and the **physical deployment**.

## 8.4 Request Flow Is Not Always a Strict Chain

A common mistake is to assume that every request must follow:

```text
DNS → API Gateway → Service Discovery → Load Balancer → Instance
```

That sequence is one possible explanation of a system, not a universal implementation.

For example:

- Service discovery may update a proxy's endpoint list before requests arrive.
- DNS may resolve directly to a load balancer.
- A gateway may perform load balancing itself.
- A Kubernetes Service may provide the stable backend endpoint.
- A service mesh sidecar may make the final instance-selection decision.
- A global proxy may route traffic without relying on DNS-based regional selection.

The right architecture depends on the technology stack and deployment requirements.

---

# 9. Failure Scenarios and Resilience

A production architecture must consider not only how requests succeed, but also what happens when components fail.

## 9.1 One Backend Instance Fails

Suppose Order O2 becomes unhealthy.

```text
Before failure:

Load Balancer
   ├── O1 ✓
   ├── O2 ✓
   └── O3 ✓

After failure:

Load Balancer
   ├── O1 ✓
   ├── O2 ✗
   └── O3 ✓
```

The load balancer excludes O2 after its health-check policy detects the failure.

Requests can continue reaching O1 and O3.

However, this only works if the remaining instances have enough capacity and the load balancer's health information is accurate.

## 9.2 One Availability Zone Fails

If AZ-1 becomes unavailable, traffic may be directed toward healthy instances in AZ-2.

For this to work:

- Application instances must exist in AZ-2.
- Network routing must support the failover.
- Dependencies must remain accessible.
- The surviving zone must have sufficient capacity.
- Health and traffic-management mechanisms must detect the failure.

Deploying two zones without sufficient surviving capacity does not guarantee successful failover.

## 9.3 An Entire Region Fails

If Region A becomes unavailable, the global traffic-management layer may direct new requests to Region B.

However, successful recovery depends on more than changing the endpoint.

The application must also consider:

- Whether Region B is active or can be activated.
- Whether required data is available and sufficiently current.
- Whether databases can accept writes in the recovery region.
- Whether authentication and other dependencies remain available.
- Whether Region B can handle the additional traffic.

Cross-region failover is an application and data architecture problem, not just a routing problem.

## 9.4 The API Gateway Fails

If a single gateway instance fails, a load balancer or other traffic-management mechanism can direct traffic to another healthy gateway instance.

If the entire gateway tier fails, backend services may remain healthy but become unreachable through the public API.

This is why gateway deployments often require:

- Multiple replicas
- Health checks
- Capacity planning
- Appropriate availability-zone placement
- Monitoring and alerting

The gateway is a critical component and can become a bottleneck if poorly designed.

## 9.5 The Load Balancer Fails

A managed or redundant load-balancing service may hide much of its own infrastructure complexity.

A self-managed load balancer needs an appropriate high-availability design.

The exact solution depends on whether the load balancer is implemented as a managed service, a cluster, a proxy deployment, or a software component in a container platform.

## 9.6 Service Discovery Becomes Unavailable

The impact depends on the discovery implementation.

If a consumer already has a usable list of destinations, it may continue routing traffic temporarily even if the registry becomes unavailable.

If a new service resolution is required and no usable information is available, requests may fail.

This illustrates an important resilience principle:

**A control-plane failure does not always have to cause an immediate data-plane failure.**

Systems can use cached endpoint information, local proxy configuration, and carefully designed fallback behavior to remain operational during certain control-plane disruptions.

However, stale information can also direct traffic toward removed or unhealthy instances, so caching requires appropriate update and expiration strategies.

## 9.7 A Backend Becomes Slow Instead of Failing

An instance can be alive and return successful health checks while responding very slowly to real requests.

Possible protections include:

- Request timeouts
- Concurrency limits
- Circuit breakers
- Bounded queues
- Load shedding
- Retry policies with backoff
- Autoscaling where appropriate

Retries must be used carefully.

Retrying an expensive request can increase load on an already overloaded service. Retrying a payment or order-creation request can also cause duplicate effects unless the operation is designed to be safely retried.

A circuit breaker can temporarily stop calls to a failing dependency, while timeouts prevent callers from waiting indefinitely.

## 9.8 Failure Summary

| Failure | Possible response |
|---|---|
| One instance fails | Remove it from eligible destinations. |
| One instance becomes slow | Apply timeouts, load shedding, or other overload controls. |
| One AZ fails | Route traffic to surviving zones if capacity and dependencies permit. |
| Gateway replica fails | Direct traffic to another healthy replica. |
| Discovery control plane fails | Continue using usable cached or locally available endpoint information where supported. |
| One region fails | Activate or route traffic to a suitable recovery region. |
| A database fails | Apply database-specific redundancy and recovery mechanisms. |

No single gateway or load balancer can compensate for every failure in the system.

---

# 10. Security and Production Considerations

Understanding the components is only part of designing a production-ready architecture.

## 10.1 Protect Internal Services

If the architecture assumes that clients must access backend services through the gateway, network controls should prevent unintended direct public access to those services.

Otherwise, an attacker might bypass gateway policies by connecting directly to a backend endpoint.

Internal services may still require their own authentication and authorization controls.

## 10.2 Avoid Treating the Gateway as the Entire Security System

A gateway can enforce shared API policies, but services should validate business-critical rules themselves.

For example:

```text
Gateway:
    Is the token valid?
    Is this API accessible to the caller?

Order Service:
    Does this order belong to the caller?
    Can this order transition to the requested state?
```

The gateway is not a replacement for secure application logic.

## 10.3 Avoid an Overloaded Gateway

Putting every business operation into the gateway makes it difficult to maintain and scale independently.

Keep the gateway focused on API-facing concerns. Put domain-specific workflows and business rules in appropriate services or application layers.

## 10.4 Monitor the Whole Request Path

Useful metrics include:

- Request rate
- Error rate
- Latency percentiles
- Backend connection failures
- Gateway CPU and memory usage
- Active connections
- Queue depth
- Health-check failures
- Rate-limit rejections

Distributed tracing can connect a client request to downstream service calls, making it easier to identify where latency or failure originates.

## 10.5 Understand the Bottleneck Risk

An API gateway can centralize useful policies, but it also adds a hop and consumes resources.

A poorly configured gateway may become the bottleneck for an otherwise scalable application.

Production design should consider:

- Horizontal scaling
- Connection pooling
- Timeouts
- Resource limits
- Payload size limits
- Rate limiting
- Backpressure
- Failure isolation

The goal is not to remove every intermediary. It is to ensure that each intermediary provides sufficient value without becoming an unacceptable failure or performance bottleneck.

---

# 11. Common Architectural Patterns

There is no single architecture suitable for every application. The following patterns illustrate how responsibilities can be arranged.

## 11.1 Pattern A: Gateway with Direct Backend Routing

```text
Client
  │
  ▼
API Gateway
  │
  ├──► User Service
  ├──► Order Service
  └──► Payment Service
```

The gateway routes requests to backend service endpoints. Each endpoint may internally resolve to one or multiple instances.

**Suitable for:** Small or medium systems where the gateway or platform can manage backend destinations directly.

## 11.2 Pattern B: Gateway Plus Service-Level Load Balancing

```text
Client
  │
  ▼
API Gateway
  │
  ▼
Order Service Endpoint
  │
  ▼
Load Balancer
  │
  ├──► Order O1
  ├──► Order O2
  └──► Order O3
```

The gateway chooses the backend service, and the service endpoint distributes traffic among its instances.

**Suitable for:** Systems where backend services need independent scaling and traffic distribution.

The load balancer may be a dedicated managed service, an internal proxy, or a platform-managed virtual endpoint.

## 11.3 Pattern C: Global Routing with Multi-Region Gateways

```text
                       Client
                         │
                         ▼
                 Global Traffic Manager
                         │
                ┌────────┴────────┐
                ▼                 ▼
          ┌───────────┐     ┌───────────┐
          │ Region A  │     │ Region B  │
          │ Gateway   │     │ Gateway   │
          └─────┬─────┘     └─────┬─────┘
                │                 │
                ▼                 ▼
           Backend A          Backend B
```

The global layer directs traffic to a suitable regional endpoint. Each region has its own gateway and backend infrastructure.

**Suitable for:** Geographically distributed applications or systems requiring cross-region recovery.

The data layer must be designed to support the intended consistency, availability, and recovery objectives.

## 11.4 Pattern D: Kubernetes-Based Routing

A simplified Kubernetes deployment might look like this:

```text
External Client
      │
      ▼
Ingress / Gateway
      │
      ▼
Kubernetes Service
      │
      ▼
Eligible Application Pods
```

The exact setup depends on the cluster configuration.

A Kubernetes Service can provide a stable endpoint for a set of pods. An Ingress controller or Gateway API implementation can provide HTTP routing at the cluster edge. Cloud load balancers may also be involved.

Do not assume that every Kubernetes deployment requires a separate load balancer for every microservice.

## 11.5 Pattern E: Gateway with a Service Mesh

A service mesh provides infrastructure for service-to-service communication, often through proxies.

A simplified flow is:

```text
External Client
      │
      ▼
API Gateway
      │
      ▼
Source Service + Proxy
      │
      ▼
Destination Proxy
      │
      ▼
Destination Service
```

A mesh can provide capabilities such as traffic management, service identity, encryption, and telemetry.

The gateway and service mesh have different primary scopes, although their capabilities can overlap.

A mesh does not eliminate the need to define clear API boundaries or authorization policies.

## 11.6 Choosing a Pattern

| Situation | Reasonable starting point |
|---|---|
| Small application with a few services | A simple reverse proxy or gateway may be sufficient. |
| Many public APIs and shared policies | A dedicated API gateway may be valuable. |
| Many replicas of a backend | Load balancing or platform-managed endpoint distribution is important. |
| Containerized application on Kubernetes | Consider Services, Ingress or Gateway API, and cluster networking. |
| Multi-region user traffic | Add suitable global traffic management and regional recovery design. |
| Complex service-to-service traffic requirements | Evaluate whether a service mesh provides enough value to justify its operational cost. |

---

# 12. Common Mistakes and Misconceptions

### Mistake 1: "An API gateway only routes requests."

Routing is fundamental, but gateways often also provide authentication integration, rate limiting, request transformation, and observability.

### Mistake 2: "A load balancer only works at Layer 4."

Layer 7 load balancers can understand HTTP and route by hostnames, paths, and headers.

### Mistake 3: "Every microservice needs a separate load balancer."

Not necessarily. One load balancer can serve multiple backend pools, and platform networking may provide traffic distribution without a dedicated load-balancer resource per service.

### Mistake 4: "Service discovery selects the instance for every request."

Not always. Discovery can supply endpoint information, while a separate proxy or client chooses the instance.

### Mistake 5: "The API gateway always operates at the regional level."

No. Its placement depends on the architecture and implementation.

### Mistake 6: "DNS always routes users to the nearest healthy region."

DNS routing can use geography and health information, but resolver caching, routing policies, and provider behavior affect the result. The geographically closest endpoint may not be the best endpoint.

### Mistake 7: "A valid access token means the user can access every resource."

No. Authentication establishes identity or verifies credentials. Authorization determines what that identity may do.

### Mistake 8: "Adding a load balancer automatically scales the application."

A load balancer distributes traffic among available destinations. Additional capacity must be provided by existing headroom, horizontal scaling, vertical scaling, or another capacity-management strategy.

### Mistake 9: "A load balancer always knows which instance has the least CPU usage."

Not necessarily. A basic load balancer may use round robin, connection counts, or other limited signals. Application-level resource utilization requires appropriate telemetry and a supported balancing mechanism.

### Mistake 10: "Multi-AZ deployment guarantees high availability."

Not by itself. The application must have healthy surviving capacity, resilient dependencies, appropriate network paths, and tested failover behavior.

### Mistake 11: "A single entry point means one physical gateway server."

A single logical entry point can be backed by many gateway replicas across availability zones or regions.

### Mistake 12: "Every request follows DNS → gateway → service discovery → load balancer."

That is one possible conceptual flow, not a mandatory chain. Discovery and balancing may be integrated, and some components may not exist separately.

---

# 13. Classic Interview Questions

## Q1. What is the difference between an API gateway and a load balancer?

An API gateway manages API-facing traffic and policies, such as routing, authentication integration, and rate limiting.

A load balancer distributes traffic across eligible destinations.

The gateway often determines which backend API should handle a request, while the load balancer often determines which eligible instance receives it. Their functionality can overlap.

## Q2. Why do microservices need an API gateway?

A gateway provides clients with a stable API entry point and can centralize shared policies.

It also reduces the need for clients to understand internal service topology. However, not every microservice architecture requires a dedicated gateway.

## Q3. If the API gateway is a single entry point, isn't it a single point of failure?

It can become one if deployed as a single instance without redundancy.

In production, the gateway tier can run multiple replicas behind appropriate traffic management. Health checks, capacity planning, and failure isolation are also important.

The gateway is a single logical entry point, not necessarily a single physical machine.

## Q4. Which comes first: the API gateway or the load balancer?

There is no universal ordering because "load balancer" can refer to different layers.

One possible flow is:

```text
Client
  → Global Traffic Manager
  → Regional API Gateway
  → Service-Level Load Balancer
  → Backend Instance
```

Another architecture may place a regional load balancer in front of multiple gateway replicas:

```text
Client
  → Regional Load Balancer
  → API Gateway Replica
  → Backend Service
```

A third architecture may use one application-layer component for both gateway and load-balancing functions.

Always identify which traffic is being balanced and what each component is responsible for.

## Q5. Can a load balancer replace an API gateway?

Sometimes.

A layer 7 load balancer may support host-based and path-based routing, TLS termination, and certain access policies.

However, a dedicated API gateway may provide more specialized capabilities for API authentication, quotas, consumer management, versioning, request policies, or composition.

Choose according to the actual requirements rather than the component name.

## Q6. Can an API gateway replace a load balancer?

Sometimes.

Some gateways can distribute traffic across multiple backend instances.

A separate load balancer may still be useful for specialized traffic distribution, internal network endpoints, or independently managed backend pools.

Avoid deploying duplicate layers unless their responsibilities justify the additional complexity.

## Q7. What is service discovery, and why is it necessary?

Service discovery allows consumers to find current service destinations without relying on permanently hardcoded instance addresses.

It is particularly useful when instances are created, removed, restarted, or moved dynamically.

## Q8. What is the difference between client-side and server-side discovery?

In client-side discovery, the consumer obtains a list of instances and selects one.

In server-side discovery, the consumer sends traffic to a stable endpoint, and an intermediary selects the destination.

Both approaches can be valid, depending on the platform and architecture.

## Q9. What happens when a backend instance fails?

The system should detect that the instance is unhealthy and stop directing new traffic to it.

Existing connections and in-flight requests require additional consideration. A failed request is not automatically safe to retry, especially if the operation has side effects.

## Q10. What is the difference between a region and an availability zone?

A region is a geographic area containing cloud infrastructure.

An availability zone is an isolated infrastructure location within a region.

Multi-AZ deployments help protect against certain localized failures. Multi-region deployments can provide broader geographic resilience but require more complex data and recovery strategies.

## Q11. How can an API gateway handle millions of requests?

A gateway can scale horizontally using multiple replicas and suitable traffic distribution.

It also requires appropriate resource sizing, connection management, rate limits, caching where safe, and downstream capacity.

A distributed gateway does not remove the need to scale backend services and dependencies.

## Q12. Does service discovery have to run on every request?

No.

Service discovery can update destination information that consumers or proxies use for many requests. The registry does not have to be contacted synchronously for every API call.

## Q13. Does a load balancer know whether an application operation succeeded?

A basic load balancer usually knows about connection outcomes, health checks, and configured response-level signals. It does not inherently understand every business operation.

For example, an HTTP request might return `200 OK` even when the business result indicates that an order could not be fulfilled. Business success must be represented and monitored at the application level.

## Q14. How does a system handle an entire regional outage?

A multi-region design can direct traffic toward another region, but only if the destination region is capable of serving requests.

The recovery plan must account for application capacity, databases, replication, authentication, dependencies, and the required recovery objectives.

## Q15. What happens if the load balancer detects a failure too slowly?

Requests may continue reaching an unhealthy destination until the failure is detected and traffic is removed.

Reducing detection time can improve recovery, but overly aggressive health checks can cause false positives and unnecessary traffic shifts.

Health-check intervals, thresholds, timeouts, and recovery behavior must be configured according to the application.

## Q16. What is the difference between a reverse proxy and an API gateway?

A reverse proxy accepts traffic on behalf of backend servers and forwards requests.

An API gateway is a role or architectural component that commonly builds on reverse-proxy functionality with API-specific routing and policy management.

The categories overlap: many API gateways are reverse proxies, but not every reverse proxy is a full API gateway.

## Q17. Is a service mesh the same as an API gateway?

No.

An API gateway commonly manages client-to-service API traffic at the system boundary.

A service mesh commonly manages service-to-service communication and related policies inside a distributed application.

Their capabilities can overlap, and a system may use both.

## Q18. When would you avoid a dedicated API gateway?

You might avoid one when the application has very few endpoints, shared API policies are minimal, and a simpler reverse proxy or platform-native routing mechanism meets the requirements.

A dedicated gateway adds another operational component and can increase latency and complexity.

## Q19. What is the difference between global load balancing and regional load balancing?

Global traffic management directs traffic among regions or global endpoints.

Regional load balancing distributes traffic among destinations within a region or another configured regional scope.

The exact scope depends on the implementation; these are architectural responsibilities rather than universal product labels.

## Q20. How would you design a resilient e-commerce API?

A reasonable starting point would include:

1. An HTTPS public endpoint.
2. Redundant API gateway replicas.
3. Authentication and API-level authorization policies.
4. Service discovery or stable backend endpoints.
5. Load balancing across healthy service instances.
6. Multi-zone application deployment.
7. Timeouts, bounded retries, and overload protection.
8. Database redundancy and a tested recovery strategy.
9. Metrics, logs, distributed tracing, and alerts.
10. Multi-region traffic management if the availability requirements justify it.

The final design depends on the required latency, availability, cost, consistency, and operational complexity.

---

# 14. Practical Exercises

Use these exercises to test whether you understand the responsibilities rather than just the terminology.

## Exercise 1: Identify the Responsible Component

For each situation, identify the primary responsibility involved.

- [ ] A request to `/api/v1/orders` must reach the Order Service.
- [ ] One Order Service replica has become unhealthy.
- [ ] The IP address of a newly deployed service instance is unknown to consumers.
- [ ] A user exceeds their API quota.
- [ ] An entire availability zone becomes unavailable.
- [ ] An API needs to combine profile and order information into one response.
- [ ] A public hostname must direct traffic toward a suitable regional endpoint.

<details>
<summary>Show suggested answers</summary>

1. API gateway or another HTTP routing component.
2. Load balancer or proxy health management.
3. Service discovery or platform-managed endpoint resolution.
4. API gateway or another component enforcing the applicable rate limit.
5. Multi-AZ deployment and traffic failover mechanisms.
6. API composition, a BFF, or another suitable aggregation layer.
7. Global traffic management, potentially using DNS-based routing.

Some responsibilities can be implemented by the same component.

</details>

## Exercise 2: Design a Request Flow

You are building a microservice application with:

- User Service
- Order Service
- Payment Service
- Three instances of Order Service
- Two availability zones
- Two geographic regions

Draw an architecture that shows:

1. The public entry point.
2. Regional routing.
3. API gateway replicas.
4. Backend service discovery or endpoint resolution.
5. Load balancing across Order Service instances.
6. The behavior when one Order Service instance fails.
7. The behavior when an entire region becomes unavailable.

Then identify which parts of the architecture require separate products and which responsibilities could be combined.

## Exercise 3: Analyze a Failure

An application has the following components:

```text
Client
  → API Gateway
  → Load Balancer
  → Order Service
  → Database
```

The Order Service instances are healthy, but the database is unavailable.

Answer these questions:

1. Will the load balancer necessarily detect the database failure?
2. Can the gateway guarantee that an order will be created?
3. What should the Order Service return if it cannot complete the operation?
4. When is retrying the request safe?
5. What monitoring would help identify the actual failure?

Think carefully: a healthy service process does not imply that every dependency is healthy.

## Exercise 4: Challenge the Architecture

Evaluate the following claim:

> "Every microservice architecture requires a DNS load balancer, one API gateway per region, one load balancer per availability zone, and a service registry queried on every request."

Identify at least five problems with this statement.

<details>
<summary>Show suggested answers</summary>

Possible issues:

1. DNS-based global routing is not mandatory.
2. A dedicated API gateway is not required for every microservice architecture.
3. Gateways are not universally deployed one per region.
4. Load balancer placement depends on the architecture and platform.
5. One load balancer per AZ is not a universal requirement.
6. Service discovery may be integrated into a platform or proxy.
7. Discovery information does not need to be queried for every request.
8. Some components combine multiple responsibilities.

A good architecture begins with requirements and failure assumptions, not a fixed list of products.

</details>

## Exercise 5: Explain It Without Jargon

Explain the difference between an API gateway, a load balancer, and service discovery in no more than five sentences.

A strong answer should explain:

- Which service a request targets.
- How an instance is selected.
- How the system knows where instances are.
- Why these responsibilities can overlap in real deployments.

---

# 15. Quick Revision Sheet

## Core Concepts

| Concept | Remember |
|---|---|
| API Gateway | API-facing entry point for routing and shared policies. |
| Load Balancer | Distributes traffic across eligible destinations. |
| Service Discovery | Makes service destinations discoverable. |
| Reverse Proxy | Receives requests on behalf of backend servers and forwards them. |
| Global Traffic Management | Directs traffic among global or regional endpoints. |
| Region | Geographic area containing infrastructure. |
| Availability Zone | Isolated infrastructure location within a region. |
| Authentication | Establishes or verifies identity. |
| Authorization | Determines what a caller is permitted to do. |
| Health Check | Provides a signal about an endpoint's eligibility or condition. |

## The Three Most Important Questions

**1. Which service should handle the request?**

Usually answered by API routing, often implemented by an API gateway.

**2. Which instance should receive the request?**

Usually answered by a load balancer, proxy, or client-side balancing mechanism.

**3. Where can the service instances be found?**

Answered by service discovery or a platform that provides stable service endpoints.

## Example Architecture

```text
Client
  │
  ▼
Global Traffic Management
  │
  ▼
Regional API Gateway
  │
  ▼
Service Endpoint / Load Balancer
  │
  ▼
Healthy Backend Instance
```

This is one common pattern, not a mandatory sequence.

## Key Interview Takeaways

- A single logical entry point does not mean a single physical gateway.
- API gateways and load balancers have different primary responsibilities, but their capabilities can overlap.
- Service discovery and load balancing are related but distinct.
- Layer 4 and Layer 7 load balancing differ in the information they use to make routing decisions.
- Multi-AZ and multi-region deployments require resilient dependencies, adequate capacity, and tested failover.
- Authentication at the gateway does not eliminate service-level authorization.
- Retries, health checks, and failover must be designed around application behavior and failure modes.

## Final Mental Model

Think of an e-commerce request as a journey:

1. **Global traffic management** directs the client toward an appropriate endpoint.
2. **The API gateway** applies API-level rules and routes the request.
3. **Service discovery or endpoint resolution** makes the backend destinations available.
4. **A load balancer or proxy** selects an eligible destination.
5. **The backend service** performs the business operation.

The same physical component may perform more than one of these responsibilities.

**The most important lesson:** Understand what decision must be made, which component makes it, and how the system behaves when that decision or its destination fails.

---
