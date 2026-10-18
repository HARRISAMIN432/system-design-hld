# Chapter 5 — Proxy vs Reverse Proxy

> **Goal:** Understand forward and reverse proxies from first principles, how they process requests, how they differ from VPNs and load balancers, and when each is justified in a real-world architecture.

---

## Table of Contents

1. [The Intuition: Why Do Proxies Exist?](#1-the-intuition-why-do-proxies-exist)
2. [What Is a Proxy Server?](#2-what-is-a-proxy-server)
3. [Forward Proxy](#3-forward-proxy)
4. [Forward Proxy: Capabilities and Trade-offs](#4-forward-proxy-capabilities-and-trade-offs)
5. [Reverse Proxy](#5-reverse-proxy)
6. [Reverse Proxy: Capabilities and Trade-offs](#6-reverse-proxy-capabilities-and-trade-offs)
7. [Forward Proxy vs Reverse Proxy](#7-forward-proxy-vs-reverse-proxy)
8. [Proxy vs VPN](#8-proxy-vs-vpn)
9. [Reverse Proxy vs Load Balancer](#9-reverse-proxy-vs-load-balancer)
10. [Proxy vs Firewall vs NAT](#10-proxy-vs-firewall-vs-nat)
11. [Where Proxies Fit in Real Architectures](#11-where-proxies-fit-in-real-architectures)
12. [Performance, Security, and Failure Scenarios](#12-performance-security-and-failure-scenarios)
13. [Common Misconceptions](#13-common-misconceptions)
14. [Classic Interview Questions](#14-classic-interview-questions)
15. [Practical Exercises](#15-practical-exercises)
16. [Quick Revision Sheet](#16-quick-revision-sheet)

---

# 1. The Intuition: Why Do Proxies Exist?

Imagine a university with thousands of students who need to access the internet.

The university wants to control which websites students can access, reduce unnecessary bandwidth consumption, and monitor network traffic according to its policies.

Instead of allowing every student device to connect directly to external websites, the university routes eligible traffic through an intermediary.

Now imagine the opposite situation: the university hosts a website used by students around the world.

Students should not need to know the internal IP addresses of the university's application servers. The university can expose one public-facing endpoint that receives requests and forwards them to its internal infrastructure.

Both situations use intermediaries, but the purpose differs.

- In the first situation, the intermediary represents the **clients**.
- In the second situation, the intermediary represents the **servers**.

These are the two fundamental proxy patterns.

## 1.1 Forward Proxy: The Client's Representative

<pre>
Student Device ──► University Proxy ──► Internet ──► Website
                         │
                   Applies policies
                   on client traffic
</pre>

The student device uses a proxy to access an external website.

The website sees the proxy as its immediate network peer rather than receiving a direct connection from the student's device.

## 1.2 Reverse Proxy: The Server's Representative

<pre>
Internet ──► Public Reverse Proxy ──► Internal Application Server
                     │
               Applies policies
               on incoming traffic
</pre>

An external client accesses a public endpoint. The reverse proxy forwards the request to an internal application server.

The client does not need to connect directly to the application server.

## 1.3 The Key Insight

Both proxies act as intermediaries, but they serve different sides of the communication.

A forward proxy typically controls or facilitates **outbound client traffic**.

A reverse proxy typically controls or facilitates **inbound traffic to servers**.

This distinction explains why forward proxies are common in corporate networks, while reverse proxies are common in web hosting and application infrastructure.

---

# 2. What Is a Proxy Server?

A **proxy server** is an intermediary that receives a request from one party and makes or forwards a corresponding request toward another party.

The proxy participates in the communication instead of simply allowing the original client to communicate directly with the destination.

A basic proxy interaction looks like this:

<pre>
Client              Proxy               Destination
  │                   │                       │
  │── Request ───────►│                       │
  │                   │── Forward request ───►│
  │                   │◄── Response ──────────│
  │◄── Response ──────│                       │
</pre>

The proxy can apply policies before forwarding requests and can process responses before returning them.

Depending on its implementation, it may:

- Inspect application requests.
- Enforce access-control policies.
- Authenticate users.
- Modify request or response headers.
- Cache eligible responses.
- Record logs and metrics.
- Terminate and establish network connections.
- Forward requests to selected destinations.

Not every proxy supports every capability.

## 2.1 A Proxy Is Not Just a Router

A router primarily forwards network packets between networks.

A proxy participates in a communication on behalf of another party. An application-layer proxy may terminate one connection and establish another.

For example, an HTTP reverse proxy might receive an HTTPS request from a client, decrypt it, inspect the HTTP request, and then establish a separate connection to a backend server.

This allows the proxy to enforce policies that a basic network router cannot apply to encrypted HTTP content without additional inspection capabilities.

## 2.2 Why Is a Proxy Called an Intermediary?

Consider these two paths.

**Without a proxy:**

<pre>
Client ─────────────────────────► Server
</pre>

The client connects directly to the destination.

**With a proxy:**

<pre>
Client ─────► Proxy ─────────────► Server
</pre>

The client connects to the proxy, and the proxy communicates with the destination.

The proxy can therefore become a central control point between the two parties.

This architecture can improve control and simplify infrastructure, but it also introduces another component that must be secured, monitored, and kept available.

---

# 3. Forward Proxy

## 3.1 Definition

A **forward proxy** is an intermediary that acts on behalf of clients when they access external destinations.

It is typically positioned on the client side of the communication.

<pre>
CLIENT NETWORK                 EXTERNAL NETWORK

┌───────────────┐
│ Client A      │──┐
└───────────────┘  │
                   ▼
              ┌───────────────┐
              │ Forward Proxy │
              │               │
              │ Policy checks │
              │ Optional cache│
              └───────┬───────┘
                      │
                      ▼
                   Internet
                      │
                      ▼
                 ┌──────────┐
                 │ Website  │
                 └──────────┘
</pre>

The forward proxy is configured or required by the client-side environment.

Examples include:

- A company's outbound web proxy.
- A university's internet-access proxy.
- A proxy used by a development environment to reach external APIs.
- A controlled proxy used to access resources through another network.

## 3.2 Step-by-Step Request Flow

Suppose an employee opens:

`https://example.com`

The organization requires browser traffic to pass through its forward proxy.

1. The browser is configured to use the proxy, or the network enforces proxy usage.
2. The browser sends the request through the proxy mechanism.
3. The proxy evaluates applicable rules.
4. If the request is permitted, the proxy connects to the destination.
5. The destination sends its response to the proxy.
6. The proxy returns the response to the browser.

The proxy can reject requests before they reach the destination.

For HTTPS, the details depend on the proxy mode. A typical HTTP proxy can use the `CONNECT` method to establish a tunnel to the destination. The TLS connection may then run through that tunnel without the proxy decrypting the HTTPS content.

An organization that intentionally inspects HTTPS content requires a separate, explicitly configured TLS inspection mechanism and appropriate trust configuration.

## 3.3 Example: Corporate Website Filtering

Suppose a company blocks certain websites on its managed devices.

Its policy might be:

```text
Allowed:
    company.com
    docs.example.com
    github.com

Restricted:
    blocked.example
```

When a client requests a restricted destination, the proxy evaluates the request against the policy and denies it.

The client might receive an access-denied response or a policy notification.

This is an example of centralized outbound access control.

The proxy can enforce rules for traffic that actually passes through it. It cannot automatically control every connection from every device unless the network and device configurations prevent bypassing it.

## 3.4 Forward Proxy and IP Visibility

A destination server typically sees the forward proxy's address as the immediate source of the connection.

For example:

```text
Client IP:          192.168.1.25
Forward Proxy IP:   203.0.113.10
Destination Server: 198.51.100.20
```

These are illustrative documentation addresses.

If the proxy establishes the external connection, the destination's network connection generally originates from the proxy's public-facing address.

However, this does not guarantee anonymity.

The destination may still identify the user through:

- Authentication credentials.
- Cookies and session identifiers.
- Browser fingerprinting.
- Application-level identifiers.
- Headers deliberately added by the proxy.
- Other identifying information in the request.

**Remember:** A forward proxy can mask the client's direct network address from a destination, but IP masking and anonymity are not equivalent.

---

# 4. Forward Proxy: Capabilities and Trade-offs

## 4.1 Centralized Access Control

Organizations can enforce outbound traffic policies in one place.

For example, an organization may permit access to approved package registries while restricting unapproved external destinations.

Benefits include:

- Consistent policy enforcement.
- Centralized audit logs.
- Reduced dependence on individual browser settings.
- A common place to block known malicious destinations.

However, enforcement depends on the network architecture. Clients must not be able to bypass the proxy through unrestricted direct connections if the proxy is intended to be mandatory.

## 4.2 IP Masking

External servers generally observe the proxy's address as the source of the proxy-originated connection.

This can be useful when clients should not expose their direct public IP addresses to destinations.

It does not prevent identity disclosure through other channels, nor does it encrypt traffic by itself.

## 4.3 Caching

A forward proxy can cache eligible responses.

Suppose 100 employees download the same large public software package.

Without a shared cache, each download may consume bandwidth from the external source.

With a cache:

```text
First request:
Client ──► Proxy ──► Origin Server
                  │
                  └── Store eligible response

Later request:
Client ──► Proxy Cache
              │
              └── Return cached response
```

This can reduce repeated downloads and external bandwidth consumption.

Caching is only safe when the response is cacheable and the cache correctly respects freshness, authorization, and relevant request headers. Private or personalized responses must not be shared accidentally.

## 4.4 Request Collapsing

Suppose several clients simultaneously request the same cacheable resource, and the resource is not yet in the cache.

A capable proxy can combine concurrent cache misses into a single upstream request.

```text
Client A ──┐
Client B ──┼──► Proxy ───► One upstream request
Client C ──┤        │
Client D ──┘        └────► Shared response
```

This behavior is sometimes called request collapsing or request coalescing.

It is not a universal proxy feature, and it must be implemented carefully. Requests that differ in authorization, cache keys, or other relevant properties may not be safe to combine.

## 4.5 Accessing Another Network's Resources

A forward proxy may provide access through a network that has different routing or access policies.

For example, a development environment may require a corporate proxy to access approved external package repositories.

A proxy does not inherently defeat access restrictions, and it should not be treated as a guarantee that geographically or legally restricted content can be accessed.

## 4.6 Limitations of a Forward Proxy

A forward proxy introduces trade-offs:

- It can become a bottleneck for outbound traffic.
- It adds latency and operational complexity.
- It can become a single point of failure if deployed without redundancy.
- It requires appropriate security controls and monitoring.
- It can see sensitive metadata and, in some configurations, application content.
- Misconfigured access rules can expose the proxy to abuse.

A forward proxy should be deployed because its policy, security, or performance benefits justify these costs.

---

# 5. Reverse Proxy

## 5.1 Definition

A **reverse proxy** is an intermediary that accepts requests on behalf of backend servers and forwards them to the appropriate destination.

It is positioned on the server side of the communication.

External clients communicate with the reverse proxy rather than connecting directly to individual backend servers.

```text
                  PUBLIC NETWORK
                        │
                        ▼
                ┌────────────────┐
                │ Reverse Proxy  │
                │                │
                │ TLS handling   │
                │ Routing        │
                │ Filtering      │
                └───────┬────────┘
                        │
               ┌────────┼────────┐
               ▼        ▼        ▼
            Server A  Server B  Server C
```

The backend servers may run on private addresses that are not directly reachable from the public internet.

## 5.2 Example: Hosting a Web Application

Suppose an application uses these internal services:

```text
Frontend:  10.0.1.10:3000
Backend:   10.0.1.20:8080
Admin API: 10.0.1.30:9000
```

The organization wants to expose a single public domain:

`https://app.example.com`

A reverse proxy can route incoming requests:

```text
https://app.example.com/
    └──► Frontend

https://app.example.com/api/*
    └──► Backend

https://app.example.com/admin/*
    └──► Admin API
```

The proxy receives the public request, selects the configured backend, and returns the backend's response.

This simplifies public routing and lets the organization change internal server addresses without necessarily changing the public API.

## 5.3 Step-by-Step Request Flow

Consider this request:

```http
GET /api/v1/users/42 HTTP/1.1
Host: app.example.com
```

The flow is:

1. The client resolves the public hostname.
2. The client connects to the reverse proxy or the infrastructure in front of it.
3. The reverse proxy receives the request.
4. The proxy selects a backend according to its routing configuration.
5. The proxy forwards the request to the backend.
6. The backend processes the request and returns a response.
7. The proxy relays the response to the client.

The client uses the public endpoint throughout the interaction. It does not need to know the backend's private address.

## 5.4 Does a Reverse Proxy Hide the Server's IP?

It can help keep backend addresses private, but this is not automatic.

For example, if the backend server has a public IP address and accepts connections directly from the internet, clients may still reach it without using the reverse proxy.

A stronger design uses network controls to ensure that the backend accepts traffic only from authorized infrastructure.

The reverse proxy can then act as the public entry point while backend servers remain on private networks.

Even in this arrangement, internal addresses might be disclosed through configuration leaks or application responses. The important security objective is to prevent unauthorized direct access, not merely to keep addresses secret.

## 5.5 Reverse Proxy vs API Gateway

A reverse proxy forwards requests on behalf of servers.

An API gateway is a more specific architectural role focused on API entry points and API-level policies.

An API gateway is commonly implemented using reverse-proxy functionality, but not every reverse proxy provides the full capabilities expected from an API gateway.

For example:

- A simple reverse proxy may route `/api/*` to one backend.
- An API gateway may additionally manage API consumers, authentication policies, quotas, versions, and request validation.

The categories overlap, and a single product can serve both roles.

---

# 6. Reverse Proxy: Capabilities and Trade-offs

## 6.1 Backend Isolation

A reverse proxy allows clients to use a public endpoint while backend servers remain on a private network.

This reduces direct exposure and creates a controlled path into the application.

However, a reverse proxy does not replace firewalls, network segmentation, or backend authentication.

## 6.2 Load Distribution

A reverse proxy can forward requests across several backend instances.

```text
                   Reverse Proxy
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
          App Server 1 App Server 2 App Server 3
```

A capable proxy can use balancing policies and health information to select eligible destinations.

This is one reason reverse proxies and load balancers overlap.

## 6.3 TLS Termination

HTTPS uses TLS to protect communication between a client and a TLS endpoint.

A reverse proxy can terminate the client's TLS connection and forward the request to the backend.

```text
Client
  │
  │ HTTPS
  ▼
Reverse Proxy
  │
  │ HTTP or HTTPS, depending on configuration
  ▼
Backend Server
```

When the proxy terminates TLS, it decrypts the incoming application traffic and can inspect HTTP requests.

The backend connection may use HTTP within a suitably protected network, or HTTPS when encryption between the proxy and backend is required.

For sensitive or zero-trust environments, encrypting both network segments is often appropriate.

### Why terminate TLS at a proxy?

- Centralized certificate management.
- Consistent HTTPS configuration.
- Reduced certificate-management work for individual backends.
- A place to enforce certain connection-level policies.

TLS termination does not mean the entire request path is automatically encrypted.

## 6.4 Caching

A reverse proxy can cache eligible responses from the backend.

For example, a public product catalog may be requested thousands of times.

If the responses are cacheable, the proxy can serve repeated requests without contacting the application every time.

This reduces backend load and may reduce response latency.

Dynamic and personalized responses require more careful cache policies.

## 6.5 Filtering and Rate Limiting

A reverse proxy may reject requests based on configured rules, including:

- Disallowed methods.
- Invalid request sizes.
- Rate limits.
- IP-based access restrictions.
- Hostname or path policies.
- Known malicious request patterns.

Specialized web application firewalls and edge-security systems may provide additional protection.

The amount of protection depends on the capabilities and configuration of the deployed components.

## 6.6 Logging and Observability

The reverse proxy can record information about incoming requests and upstream responses, such as:

- Request path.
- Response status.
- Request duration.
- Upstream latency.
- Connection failures.
- Request identifiers.

These records help identify whether a request failed at the edge or deeper in the application.

Sensitive values such as passwords, access tokens, and session cookies should not be logged indiscriminately.

## 6.7 DDoS Protection: What a Reverse Proxy Can and Cannot Do

A reverse proxy can reduce exposure of backend infrastructure and filter certain unwanted requests.

However, simply placing a reverse proxy in front of a server does not guarantee protection against distributed denial-of-service attacks.

If attackers can overwhelm the proxy's public network capacity, the application may still become unavailable.

Effective protection may require:

- Upstream DDoS mitigation.
- Sufficient network capacity.
- Traffic filtering.
- Rate limiting.
- Web application firewall rules.
- Connection and request limits.
- Redundant infrastructure.
- Appropriate network access controls.

**Key distinction:** Hiding backend addresses reduces direct exposure. DDoS resilience requires a broader design.

## 6.8 Limitations of a Reverse Proxy

A reverse proxy introduces its own risks:

- It may become a bottleneck.
- It can become a single point of failure without redundancy.
- Incorrect routing can send requests to the wrong backend.
- TLS or header misconfiguration can create security vulnerabilities.
- Excessive buffering or caching can consume resources.
- A compromised proxy may expose a large part of the application.

Its deployment should include appropriate security hardening, capacity planning, and monitoring.

---

# 7. Forward Proxy vs Reverse Proxy

## 7.1 The Fundamental Difference

The key difference is **which side the proxy represents**.

A forward proxy acts on behalf of clients accessing destinations.

A reverse proxy acts on behalf of servers receiving requests.

Both can hide network addresses, enforce policies, cache content, and process requests. Their deployment context and responsibilities determine which capabilities are useful.

## 7.2 Comparison Table

| Dimension | Forward Proxy | Reverse Proxy |
|---|---|---|
| Represents | Client | Server |
| Typical traffic direction | Outbound client traffic | Inbound server traffic |
| Typical location | Client or enterprise network | Server infrastructure or edge |
| Primary purpose | Control or facilitate client access | Control or facilitate access to backend servers |
| Common visibility benefit | Masks the client's direct network address from destinations | Can keep backend addresses private |
| Access control | Outbound destination policies | Inbound application and backend policies |
| Caching | Can cache eligible external resources | Can cache eligible backend responses |
| Load balancing | Not its typical primary purpose | Frequently supported |
| TLS handling | Depends on proxy mode and configuration | Often terminates public HTTPS |
| Common example | Corporate web proxy | NGINX in front of a web application |

## 7.3 One Request, Two Possible Proxy Roles

Imagine a company employee accesses `app.example.com`.

The employee's browser uses a corporate forward proxy to access the public internet.

The company's application uses a reverse proxy to receive public requests.

```text
Employee Browser
       │
       ▼
Corporate Forward Proxy
       │
       ▼
Internet
       │
       ▼
Application Reverse Proxy
       │
       ▼
Application Backend
```

Both proxies may be involved in the same overall communication path.

The corporate proxy represents the employee's client-side environment. The reverse proxy represents the application's server-side environment.

This example is a useful way to remember that the terms describe roles, not mutually exclusive technologies.

---

# 8. Proxy vs VPN

A proxy and a VPN can both change the apparent source address of traffic reaching a destination, but they operate differently.

## 8.1 How a Proxy Works

A proxy typically accepts or tunnels particular kinds of traffic and forwards it toward a destination.

For an HTTP proxy:

```text
Application
    │
    ▼
HTTP Proxy
    │
    ▼
Destination Website
```

The proxy may operate at the application layer and can apply policies to requests it understands.

A plain proxy does not inherently encrypt the entire path between the client device and the proxy.

However, the application protocol itself may be encrypted. For example, HTTPS can protect application traffic inside a proxy tunnel.

## 8.2 How a VPN Works

A virtual private network establishes a logical network connection through a VPN tunnel.

```text
Device
   │
   │ Encrypted VPN tunnel
   ▼
VPN Gateway
   │
   ▼
Destination Network or Internet
```

The tunnel protects traffic between the VPN endpoint and the VPN gateway according to the VPN protocol and configuration.

A full-tunnel VPN can route most or all IP traffic through the tunnel. A split-tunnel VPN routes only selected traffic through it.

The VPN gateway then forwards traffic according to its routing configuration.

A VPN does not automatically encrypt traffic beyond its endpoint. HTTPS or another application-layer security mechanism may still be needed to protect traffic between the VPN gateway and the destination.

## 8.3 Comparison Table

| Dimension | Proxy | VPN |
|---|---|---|
| Main abstraction | Intermediary for requests or connections | Virtual network connection through a tunnel |
| Scope | Often application-specific, but can support other traffic types | Can cover selected routes or most device traffic |
| Tunnel encryption | Not inherent to a plain proxy | Typically provides encrypted tunnel transport |
| Destination IP visibility | Can expose the proxy's address | Can expose the VPN gateway's address |
| Application inspection | Possible when the proxy can inspect the traffic | Usually does not inspect application content merely by tunneling it |
| Typical use | Web filtering, caching, outbound policies, reverse-proxy routing | Remote network access and protected network transport |

## 8.4 Important Privacy Distinction

Neither technology guarantees complete anonymity.

A proxy operator may observe connection metadata and, depending on configuration, application content.

A VPN operator may observe connection metadata and traffic that is not protected by end-to-end encryption.

Websites can also identify users through accounts, cookies, and other signals regardless of the apparent IP address.

A VPN can protect traffic over an untrusted network, but it does not replace HTTPS or make an untrusted VPN operator trustworthy.

---

# 9. Reverse Proxy vs Load Balancer

This comparison is particularly important in system design interviews.

## 9.1 Why Do They Seem Similar?

A reverse proxy receives requests and forwards them to backend servers.

A load balancer distributes requests or connections across eligible destinations.

If a reverse proxy forwards traffic to several backend instances, it is performing a load-balancing role as well.

The concepts overlap because they describe different responsibilities.

## 9.2 Example: One Backend Server

Suppose an application has only one backend server.

```text
Client
   │
   ▼
Reverse Proxy
   │
   ▼
Application Server
```

A dedicated load-balancing function may not be necessary because there is only one destination.

The reverse proxy can still provide:

- TLS termination.
- Routing.
- Request filtering.
- Logging.
- Caching where appropriate.
- A controlled public entry point.

A reverse proxy does not need multiple backend servers to be useful.

## 9.3 Example: Multiple Backend Servers

Now suppose the application has three replicas.

```text
Client
   │
   ▼
Reverse Proxy / Load Balancer
   │
   ├──► Application Server 1
   ├──► Application Server 2
   └──► Application Server 3
```

The same component can receive public traffic, enforce request policies, and distribute requests among the backend replicas.

A separate load-balancer product is not inherently required.

## 9.4 When Might Both Appear as Separate Layers?

A larger architecture might use:

```text
Client
   │
   ▼
External Load Balancer
   │
   ▼
Reverse Proxy / API Gateway
   │
   ▼
Internal Load Balancer
   │
   ▼
Backend Instances
```

Each layer must have a justified responsibility.

For example:

- The external load balancer distributes connections across gateway replicas.
- The gateway applies API-level routing and security policies.
- The internal load balancer distributes requests across service instances.

This is only one possible design. Another architecture might combine the external load balancer and reverse-proxy responsibilities in a single application-layer service.

## 9.5 Comparison Table

| Dimension | Reverse Proxy | Load Balancer |
|---|---|---|
| Primary role | Intermediate between clients and backend servers | Distribute traffic across eligible destinations |
| Must have multiple backends? | No | Multiple destinations are needed for distribution to provide value |
| TLS termination | Common capability | Supported by many products |
| HTTP path routing | Common for application-layer proxies | Supported by layer 7 load balancers |
| Health-based routing | Often supported | Common capability |
| Caching | Often supported | Depends on the product |
| Request transformation | Often supported | Depends on the product |
| Relationship | May perform load balancing | May be implemented using reverse-proxy functionality |

## 9.6 The Interview Answer

If asked, "Why do we need a load balancer if a reverse proxy can already balance traffic?", answer:

> We do not necessarily need separate components. A reverse proxy can perform load balancing if it supports the required algorithms and health management. We separate the responsibilities when the architecture benefits from independent scaling, ownership, network placement, or specialized functionality. The correct design depends on requirements, not on the names of the products.

---

# 10. Proxy vs Firewall vs NAT

These concepts are related to network security but solve different problems.

## 10.1 Firewall

A firewall applies rules that permit, reject, or otherwise control network traffic.

It may operate at different layers, depending on the type of firewall.

Examples include:

- Blocking inbound connections to a private database.
- Allowing HTTPS traffic to a public endpoint.
- Restricting outbound connections to approved destinations.

A firewall does not necessarily terminate application connections or forward requests as a proxy does.

## 10.2 NAT

Network Address Translation (NAT) modifies IP address information, and sometimes transport-layer port information, as traffic passes through a network device.

For example, several private devices can share a public IPv4 address for outbound internet access.

NAT does not inherently provide application-level authentication, request caching, or HTTP routing.

## 10.3 Proxy

A proxy participates in the communication and forwards requests or connections on behalf of a client or server.

An application-layer proxy can understand HTTP methods, paths, headers, and other application details.

## 10.4 Comparison

| Component | Primary purpose | Example |
|---|---|---|
| Firewall | Enforce network traffic rules | Block public access to a database |
| NAT | Translate network addresses or ports | Allow private devices to share a public IPv4 address |
| Forward proxy | Represent clients accessing external destinations | Corporate outbound web proxy |
| Reverse proxy | Represent servers receiving requests | NGINX in front of an application |
| Load balancer | Distribute traffic among eligible destinations | Spread requests across application replicas |

A single network appliance or cloud service may combine some of these functions. The conceptual distinction remains useful even when products overlap.

---

# 11. Where Proxies Fit in Real Architectures

## 11.1 Small Web Application

A small application might use a single reverse proxy in front of an application server.

```text
Internet
   │
   ▼
Reverse Proxy
   │
   ▼
Application Server
   │
   ▼
Database
```

This can provide HTTPS termination, routing, and a controlled entry point.

A dedicated load balancer may not be needed if there is only one backend instance and the deployment does not require additional distribution capabilities.

However, a single proxy and server are not highly available merely because a proxy is present.

## 11.2 Scalable Web Application

A larger application may run several replicas.

```text
                    Internet
                        │
                        ▼
                 Public Endpoint
                        │
                        ▼
                Reverse Proxy Tier
                        │
                ┌───────┼───────┐
                ▼       ▼       ▼
              App 1   App 2   App 3
                │       │       │
                └───────┼───────┘
                        ▼
                    Database
```

The reverse-proxy tier can distribute traffic across backend instances.

In a real deployment, the public endpoint and proxy tier may themselves be backed by redundant load-balancing infrastructure.

## 11.3 Corporate Network

A company may use a forward proxy for outbound access and a reverse proxy for applications it hosts.

```text
Company Employees
        │
        ▼
  Forward Proxy
        │
        ▼
     Internet
        │
        ▼
 Application Reverse Proxy
        │
        ▼
  Company Application
```

The forward proxy protects and controls outbound access from company devices.

The reverse proxy provides a controlled entry point for inbound access to company-hosted services.

## 11.4 Kubernetes

A Kubernetes-based application might use:

```text
External Client
      │
      ▼
External Load Balancer
      │
      ▼
Ingress Controller / Gateway
      │
      ▼
Kubernetes Service
      │
      ▼
Application Pods
```

The exact arrangement depends on the cluster and its networking implementation.

An Ingress controller or Gateway API implementation can perform reverse-proxy routing. A Kubernetes Service can provide a stable endpoint for eligible pods. An external cloud load balancer may distribute traffic to the cluster entry point.

Not every deployment needs each layer as a separate component.

## 11.5 Choosing Whether to Add a Proxy

Before introducing a proxy, ask:

1. Do we need centralized access policies?
2. Do we need to hide backend endpoints from direct public access?
3. Do we need TLS termination or HTTP routing?
4. Do we need caching or request filtering?
5. Do we need a stable entry point while backends change?
6. Can an existing platform component already provide these capabilities?
7. What happens if the proxy becomes unavailable?
8. How will we secure, scale, and monitor it?

**Engineering principle:** Add a proxy when its capabilities justify the latency, operational complexity, and failure modes it introduces.

---

# 12. Performance, Security, and Failure Scenarios

## 12.1 The Proxy Becomes a Bottleneck

If all traffic passes through one under-provisioned proxy, that proxy may become the system's limiting factor.

Possible remedies include:

- Horizontal scaling.
- Appropriate connection and worker limits.
- Efficient connection pooling.
- Caching eligible responses.
- Load distribution across proxy replicas.
- Monitoring request latency and resource usage.

The correct solution depends on whether the bottleneck is CPU, memory, network capacity, connection count, or another resource.

## 12.2 The Proxy Fails

If a single proxy is the only path to an application and it fails, the application may become unreachable even when its backend servers are healthy.

A resilient design may use multiple proxy replicas behind a suitable traffic-management mechanism.

Redundancy alone is not enough: the system must also detect failures and direct traffic toward healthy replicas.

## 12.3 The Proxy Misconfigures Client IP Headers

A reverse proxy may forward client information through headers such as:

```http
X-Forwarded-For: 198.51.100.25
X-Forwarded-Proto: https
```

These headers can be useful, but they are not inherently trustworthy.

An attacker may send a forged `X-Forwarded-For` header directly to an application if the application accepts requests from untrusted sources.

A secure deployment should:

- Restrict direct backend access.
- Trust forwarding headers only from known proxy infrastructure.
- Configure trusted proxy hops appropriately.
- Avoid using arbitrary client-supplied forwarding headers as proof of identity.

The exact behavior depends on the proxy and application configuration.

## 12.4 The Proxy Caches Private Data

Suppose a website returns a user-specific account page.

If the proxy incorrectly treats the response as public and shares it across users, one user may receive another user's information.

Caching policy must account for:

- Authentication.
- Authorization.
- Cache-control directives.
- Request headers that affect the response.
- Freshness and invalidation requirements.

Caching is a performance feature, but incorrect caching can become a security vulnerability.

## 12.5 HTTPS Is Decrypted at the Proxy

When a reverse proxy terminates TLS, it can inspect the decrypted HTTP request.

This makes request routing and filtering possible, but it also makes the proxy a sensitive security boundary.

Protect it with appropriate:

- Access controls.
- Certificate management.
- Patch management.
- Secret handling.
- Logging policies.
- Network segmentation.

If the proxy forwards requests using plain HTTP, the security of the internal network segment must be considered. For sensitive workloads, use TLS between the proxy and backend as required.

## 12.6 The Backend Is Exposed Directly

A reverse proxy can hide internal endpoints only if the network architecture prevents unauthorized direct access.

A robust design may use private addresses, security groups, firewall rules, and backend authentication.

Do not rely solely on the fact that users normally access the application through the proxy.

## 12.7 Retry and Timeout Problems

A proxy may time out while waiting for a backend response.

The client may then retry, creating another request.

For read-only operations, retries may be relatively straightforward. For operations such as payments, order creation, or account changes, retrying can cause duplicate effects unless the operation is designed to be idempotent or protected by an idempotency mechanism.

Timeouts and retries should be designed together with application semantics.

---

# 13. Common Misconceptions

### Misconception 1: "A proxy guarantees anonymity."

A proxy can hide the client's direct network address from a destination, but cookies, accounts, authentication, headers, and other signals can still identify the client.

### Misconception 2: "A proxy encrypts all traffic."

A plain proxy does not inherently encrypt the complete client-to-destination path. Encryption depends on the protocols and connections being used.

### Misconception 3: "A VPN and a proxy are the same thing."

A proxy forwards requests or connections on behalf of another party. A VPN creates a virtual network connection through a tunnel, typically with encrypted tunnel transport.

### Misconception 4: "A reverse proxy automatically prevents DDoS attacks."

It can filter traffic and reduce backend exposure, but upstream capacity and dedicated mitigation may be required to handle large distributed attacks.

### Misconception 5: "A reverse proxy always requires multiple servers."

No. It can provide TLS termination, routing, filtering, and logging even when only one backend exists.

### Misconception 6: "A load balancer and reverse proxy must be separate."

No. A reverse proxy can perform load balancing, and a layer 7 load balancer can perform many reverse-proxy functions.

### Misconception 7: "Only the proxy's IP address is ever visible."

A destination normally sees the proxy as the immediate network peer, but additional identifying information may be present in application headers or other request data.

### Misconception 8: "All traffic passes through a forward proxy once it is configured."

Not necessarily. Traffic can bypass the proxy unless the relevant network or device configuration enforces its use.

### Misconception 9: "A reverse proxy is a firewall."

A reverse proxy can enforce certain traffic policies, but it does not automatically replace network firewalls or other security controls.

### Misconception 10: "Adding more proxy layers always improves security."

Every additional layer introduces configuration, operational, and failure complexity. Layers should be added when their responsibilities are justified.

---

# 14. Classic Interview Questions

## Q1. What is a proxy server?

A proxy server acts as an intermediary between a client and a destination or between an external client and backend servers. It receives and forwards requests and may enforce policies, cache responses, or transform traffic.

## Q2. What is the difference between a forward proxy and a reverse proxy?

A forward proxy represents clients accessing external destinations. A reverse proxy represents backend servers receiving incoming requests.

The main distinction is which side of the communication the proxy serves.

## Q3. Why would an organization deploy a forward proxy?

To enforce outbound access policies, provide centralized logging, control external destinations, cache eligible content, and manage client access to external resources.

## Q4. Why would a website deploy a reverse proxy?

To provide a stable public endpoint, route requests to backend services, terminate TLS, filter traffic, cache eligible responses, and reduce direct exposure of internal servers.

## Q5. Does a forward proxy hide the client's IP address?

It typically hides the client's direct network address from the destination connection by originating a separate connection through the proxy. However, the client may still be identifiable through other information.

## Q6. Does a reverse proxy hide backend IP addresses?

It can keep backend addresses private when combined with appropriate network controls. Merely placing a reverse proxy in front of publicly reachable servers does not prevent direct access.

## Q7. What is the difference between a proxy and a VPN?

A proxy is an intermediary for requests or connections. A VPN provides a virtual network connection through a tunnel, typically with encrypted transport between the device and VPN endpoint.

Both can change the apparent source IP seen by a destination, but neither guarantees anonymity.

## Q8. Can a reverse proxy perform load balancing?

Yes. Many reverse proxies distribute traffic across multiple eligible backend destinations. Whether a separate load balancer is needed depends on the architecture and required capabilities.

## Q9. Why use a reverse proxy when there is only one backend server?

A reverse proxy can still provide TLS termination, routing, logging, request filtering, caching, and a stable public endpoint. Load balancing across multiple backends is not required for these benefits.

## Q10. What is TLS termination?

TLS termination is the point where a TLS connection ends and the protected traffic is decrypted by the endpoint.

A reverse proxy may terminate the client's HTTPS connection and establish another connection to the backend. That backend connection can use HTTP or HTTPS depending on the architecture and security requirements.

## Q11. Does TLS termination at a reverse proxy make backend communication insecure?

Not automatically. The security depends on how the proxy-to-backend connection is protected, the network controls, and the application's requirements. HTTPS can be used for both connection segments.

## Q12. How does a reverse proxy protect backend servers?

It provides a controlled entry point, can filter incoming requests, and can keep backends on private networks. Network controls must also prevent unauthorized direct access.

## Q13. Can a reverse proxy replace an API gateway?

It can replace some gateway functionality. A dedicated API gateway may offer additional API-specific policies and management capabilities. The decision depends on what the application requires.

## Q14. What is the difference between a reverse proxy and a firewall?

A firewall controls traffic according to network or application rules. A reverse proxy receives and forwards requests on behalf of backend servers. Some products combine capabilities, but the responsibilities are distinct.

## Q15. What is the difference between a proxy and NAT?

A proxy participates in forwarding a request or connection on behalf of another party. NAT translates network address or port information as packets traverse a network.

## Q16. What happens if the reverse proxy fails?

If it is the only path to the backend, incoming requests may fail even if the application servers remain healthy. Multiple replicas, suitable health checks, and resilient traffic management can reduce this risk.

## Q17. How does caching at a reverse proxy improve performance?

It allows eligible responses to be served without repeatedly contacting the backend. This can reduce backend load and latency, provided cache freshness and access-control requirements are respected.

## Q18. Can a proxy inspect HTTPS traffic?

It depends on the proxy mode. An HTTP proxy may tunnel HTTPS without decrypting it. A proxy that terminates TLS or performs explicitly configured TLS inspection can inspect the decrypted HTTP traffic.

## Q19. What is the difference between a reverse proxy and a service mesh?

A reverse proxy is an intermediary that receives and forwards requests. A service mesh is an infrastructure architecture for managing service-to-service communication, often using proxies and centralized control mechanisms. They can overlap, but they describe different architectural scopes.

## Q20. When should you avoid adding a proxy?

When the required capabilities are unnecessary, an existing platform component already provides them, or the operational and performance costs exceed the benefits.

The decision should consider security, availability, routing requirements, observability, and future scale.

---

# 15. Practical Exercises

## Exercise 1: Identify the Proxy Type

For each scenario, decide whether it describes a forward proxy, reverse proxy, or another component.

- [ ] A university restricts which websites students can access.
- [ ] A public web endpoint routes `/api/*` to a private backend.
- [ ] A company caches approved external package downloads.
- [ ] A proxy distributes incoming requests across three application replicas.
- [ ] A device creates an encrypted tunnel to a corporate network.
- [ ] A network firewall blocks public connections to a database.
- [ ] An organization translates private IP addresses to a public IPv4 address.

<details>
<summary>Show suggested answers</summary>

1. Forward proxy.
2. Reverse proxy.
3. Forward proxy.
4. Reverse proxy performing load balancing.
5. VPN.
6. Firewall.
7. NAT.

</details>

## Exercise 2: Design a Secure Web Architecture

You are deploying an application with:

- A public frontend.
- Three backend application instances.
- A private database.
- HTTPS for external clients.

Draw a design that includes:

1. A public entry point.
2. A reverse proxy or gateway.
3. Traffic distribution across backend instances.
4. A network boundary protecting the database.
5. TLS handling.
6. Monitoring and health checks.

Explain what happens if the proxy fails and how you would reduce that risk.

## Exercise 3: Debug a Header-Spoofing Problem

An application trusts the following header to determine the client's IP:

```http
X-Forwarded-For: 203.0.113.25
```

A client can connect directly to the backend and supply an arbitrary value for this header.

Answer:

1. Why is this dangerous?
2. Why is the header not proof of the client's identity?
3. What network restrictions should be applied?
4. How should the backend decide which proxy headers to trust?

## Exercise 4: Diagnose a Caching Bug

A reverse proxy caches a personalized account response. One user receives another user's account data.

Identify the likely class of mistake and explain how you would prevent it.

Consider authorization, cache keys, cache-control directives, and whether the response should be cached at all.

## Exercise 5: Architecture Interview Challenge

Evaluate this statement:

> "Every production web application needs a forward proxy, a reverse proxy, a separate load balancer, and a VPN."

Explain why this statement is incorrect.

A strong answer should distinguish mandatory requirements from optional infrastructure, describe overlapping capabilities, and explain how the architecture should be selected based on actual requirements.

---

# 16. Quick Revision Sheet

## The Core Definitions

| Term | One-line definition |
|---|---|
| Proxy | An intermediary that forwards requests or connections on behalf of another party. |
| Forward proxy | Represents clients accessing external destinations. |
| Reverse proxy | Represents backend servers receiving incoming requests. |
| VPN | Provides a virtual network connection through a tunnel, typically with encrypted transport. |
| Load balancer | Distributes traffic across eligible destinations. |
| Firewall | Enforces network or application traffic rules. |
| NAT | Translates network addresses or ports. |
| TLS termination | The point where a TLS connection ends and protected traffic is decrypted. |

## The Mental Model

```text
OUTBOUND CLIENT ACCESS

Client
   │
   ▼
Forward Proxy
   │
   ▼
External Destination


INBOUND SERVER ACCESS

External Client
   │
   ▼
Reverse Proxy
   │
   ▼
Backend Server
```

## The Most Important Distinctions

- **Forward proxy:** Controls or facilitates client-side outbound access.
- **Reverse proxy:** Controls or facilitates server-side inbound access.
- **VPN:** Provides a virtual network tunnel; its scope and encryption depend on configuration.
- **Load balancer:** Distributes traffic among eligible destinations.
- **Firewall:** Controls which traffic is permitted.
- **NAT:** Translates network addresses or ports.

## Final Takeaway

A proxy is not defined by a single feature such as IP masking, caching, or load balancing. It is defined by its role as an intermediary.

A forward proxy represents the client. A reverse proxy represents the server. Their capabilities can overlap with gateways, load balancers, and other network components.

The best architectural decision comes from understanding the problem first, assigning clear responsibilities, and avoiding unnecessary layers.

---