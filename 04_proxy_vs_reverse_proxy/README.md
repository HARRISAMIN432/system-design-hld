# Chapter 5 — Proxy vs Reverse Proxy

> Personal study notes. Goal: understand forward and reverse proxies from first principles, separate them from VPNs and load balancers, and know when each is justified.

---

## Table of Contents

1. [The Intuition](#1-the-intuition)
2. [Forward Proxy](#2-forward-proxy)
3. [Forward Proxy Benefits](#3-forward-proxy-benefits)
4. [Reverse Proxy](#4-reverse-proxy)
5. [Reverse Proxy Benefits](#5-reverse-proxy-benefits)
6. [Forward vs Reverse: The Only Real Difference](#6-forward-vs-reverse-the-only-real-difference)
7. [Proxy vs VPN](#7-proxy-vs-vpn)
8. [Reverse Proxy vs Load Balancer](#8-reverse-proxy-vs-load-balancer)
9. [Do We Need Proxies Everywhere](#9-do-we-need-proxies-everywhere)
10. [Interview Questions](#10-interview-questions)
11. [Quick Revision Sheet](#11-quick-revision-sheet)

---

## 1. The Intuition

A six-year-old wants chocolate. The child does not walk into the shop. They tell a parent, the parent goes to the shop, buys the chocolate and hands it over.

The shopkeeper never knows who the chocolate was really for. The parent **hid the child** from the outside world and kept them **anonymous**.

Mapping:

| Story | Networking |
|---|---|
| Child | **Client** |
| Parent | **Proxy server** |
| Outside world | **Internet** |
| Shopkeeper's shop | **Server** |

---

## 2. Forward Proxy

A **proxy server** (also called a **forward proxy**) sits **between the client and the internet**.

```
 Client ───► Forward Proxy ───► Internet ───► Server
 (hidden)    (visible IP)
```

**Flow:**

1. The client wants `google.com`.
2. The request goes to the proxy first.
3. The proxy forwards it to the internet **using its own IP address**, not the client's.
4. The server sees a request from the proxy's IP and has no idea who the real client is.
5. The server responds to the proxy, and the proxy relays the response to the client.

**Result:** the client's network is hidden. No server can communicate with the client directly.

---

## 3. Forward Proxy Benefits

### 3.1 IP Anonymity

The client's IP address is masked from the outside world. Servers see only the proxy.

### 3.2 Access Control

Example: at a company, all employees' internet access is forced through the company proxy.

- If an employee tries to open a blocked site (say, an AI chat tool or social media), the proxy checks its rules, **cancels the request** and returns a message such as "this site is restricted".
- If the site is allowed, the proxy forwards the request.

Organizations use this to enforce policy centrally.

### 3.3 Request Grouping (Collapsing)

If four employees all request the same resource at once, the proxy can **combine** these into a single request to the destination and then distribute the one response to all four clients.

### 3.4 Accessing Restricted Content

A proxy located in a different network or region may be able to reach content that is blocked for the client directly (subject to the law and the provider's terms).

### 3.5 Caching

After the first request for a resource passes through, the proxy can **store the response**. When the same client (or another) asks again, the proxy serves it **from cache** without contacting the origin, saving time and bandwidth.

**Position to remember:** a forward proxy always sits **between the client and the internet**.

---

## 4. Reverse Proxy

A **reverse proxy** sits **in front of servers**, between the **internet and the servers**.

```
 Internet ───► Reverse Proxy ───┬──► Server 1
 (clients)    (visible IP)      ├──► Server 2
                                └──► Server 3
                                (IPs hidden)
```

**Flow:**

1. A client wants `facebook.com`.
2. The request goes to the reverse proxy, **never directly** to an individual server.
3. The reverse proxy forwards it to one of the servers.
4. The server's response travels back through the proxy to the client.

**Result:** the real servers' IP addresses are never exposed to the internet. Only the reverse proxy's IP is public.

---

## 5. Reverse Proxy Benefits

| Benefit | Explanation |
|---|---|
| **IP anonymity for servers** | Clients and attackers never learn the servers' real IPs. |
| **Load balancing** | It can distribute traffic across servers using health checks and availability. |
| **Caching** | It can cache responses so repeated requests don't hit the servers. |
| **Protection against attacks (e.g. DDoS)** | Attack traffic hits the reverse proxy's IP first, not the servers. Proxies can also filter and rate limit. |
| **SSL termination** | HTTPS is decrypted at the proxy, so backend servers don't each handle certificates and encryption work. |
| **Logging and monitoring** | One central place for access logs and metrics. |

> Strictly speaking, hiding a server's IP reduces exposure but is not a complete defense on its own. Real DDoS protection also needs rate limiting, filtering, scrubbing services and correct network configuration.

---

## 6. Forward vs Reverse: The Only Real Difference

Both are intermediaries. They differ in **whom they work for** and **what they hide**:

| | Forward Proxy | Reverse Proxy |
|---|---|---|
| Works for | **Client** | **Server** |
| Hides | Client network / IP | Server network / IPs |
| Sits | Between **client and internet** | Between **internet and servers** |
| Typical benefits | Anonymity, access control, grouping, caching | Server anonymity, load balancing, caching, attack protection, SSL termination |

In short, they differ only in the **direction of communication**.

---

## 7. Proxy vs VPN

A VPN also routes your traffic through another server, but its purpose differs.

**VPN flow:**

1. The client connects to a VPN. A **VPN tunnel** is created.
2. Everything sent through the tunnel is **encrypted**.
3. The VPN server holds the mechanism to **decrypt** the data and forward it.

| | Proxy | VPN |
|---|---|---|
| Main function | **IP masking** | **Encrypted tunnel** + IP masking |
| Encrypts traffic? | Not by itself | **Yes** |
| Scope | Often per application or per protocol | Typically the whole device's traffic |
| Typical use | Filtering, caching, anonymity | Privacy and secure transport |

A plain proxy only masks the IP and does not encrypt traffic as a feature. A VPN encrypts everything passing through the tunnel.

---

## 8. Reverse Proxy vs Load Balancer

A reverse proxy can balance load, so are load balancers redundant?

### Case A: a single server

An application with few users and only **one** server (S1).

- **Load balancer needed?** No. There is nothing to distribute traffic among.
- **Reverse proxy useful?** **Yes.** It still provides:
  - IP anonymity / masking
  - Logging
  - Caching
  - SSL termination
  - A protective layer in front of the server

### Case B: a global, large-scale system

Traffic arrives worldwide, from many regions and availability zones.

- A **load balancer** is needed to distribute traffic globally and across zones and instances.
- A **reverse proxy** is also used.
- Typical split of responsibilities:
  - **Load balancer:** traffic distribution
  - **Reverse proxy:** IP masking, logging, caching, SSL termination, filtering

Many popular tools (Nginx, HAProxy, Envoy) blur the line and can do both roles. Conceptually, though, the responsibilities are separable, and at scale they are usually split into dedicated layers.

---

## 9. Do We Need Proxies Everywhere

**No.**

- A **small application** (for example a simple to-do app) with low traffic and low security concern does not need proxy servers.
- A **global platform** with heavy traffic and real security needs **does** need them.

Add infrastructure when you have the problem it solves, not before.

---

## 10. Interview Questions

1. What is a forward proxy? Where does it sit?
2. What is a reverse proxy? Where does it sit?
3. What is the main difference between them?
4. List four benefits of a forward proxy and four of a reverse proxy.
5. How is a proxy different from a VPN?
6. How does a reverse proxy protect against DDoS?
7. If a reverse proxy can load balance, why do we still use load balancers?
8. Does a single-server application need a reverse proxy? A load balancer?
9. What is SSL termination and why do it at the proxy?
10. Do all applications need proxy servers?

---

## 11. Quick Revision Sheet

- **Forward proxy** = client's agent; between **client and internet**; hides **client**.
- **Reverse proxy** = server's agent; between **internet and servers**; hides **servers**.
- Only the **direction** differs.
- **Forward proxy gives:** IP anonymity, access control, request grouping, restricted content access, caching.
- **Reverse proxy gives:** server IP anonymity, load balancing, caching, DDoS shielding, SSL termination, logging.
- **VPN** = encrypted tunnel; **proxy** = IP masking.
- **One server:** no load balancer needed, reverse proxy still useful.
- **Global scale:** use both.
- **Tiny apps:** may need neither.

---

**Previous:** [Chapter 4](../Chapter-04-Load-Balancing-Algorithms-and-Types/README.md) · **Back to start:** [Chapter 1 — Roadmap](../Chapter-01-Course-Roadmap/README.md)
