# Chapter 6 — Network Protocols (Network, Transport and Application Layers)

> Personal study notes. Goal: understand what a networking protocol is, how protocols from different layers cooperate to deliver a single message, and how to choose between TCP, UDP, HTTP, HTTPS, WebSockets and WebRTC.

---

## Table of Contents

1. [What Is a Networking Protocol](#1-what-is-a-networking-protocol)
2. [The Three Layers We Care About](#2-the-three-layers-we-care-about)
3. [A Message's Journey: A Chat Example](#3-a-messages-journey-a-chat-example)
4. [Transport Layer: TCP](#4-transport-layer-tcp)
5. [Transport Layer: UDP](#5-transport-layer-udp)
6. [TCP vs UDP](#6-tcp-vs-udp)
7. [Application Layer: Client-Server Protocols](#7-application-layer-client-server-protocols)
8. [HTTP and HTTPS](#8-http-and-https)
9. [WebSockets](#9-websockets)
10. [Other Client-Server Protocols: FTP and SMTP](#10-other-client-server-protocols-ftp-and-smtp)
11. [Peer-to-Peer Protocols and WebRTC](#11-peer-to-peer-protocols-and-webrtc)
12. [What "HTTP Is TCP-Based" Really Means](#12-what-http-is-tcp-based-really-means)
13. [Choosing the Right Protocol](#13-choosing-the-right-protocol)
14. [Interview Questions](#14-interview-questions)
15. [Quick Revision Sheet](#15-quick-revision-sheet)

---

## 1. What Is a Networking Protocol

> **Networking protocols are the set of rules and conventions that govern how data is transmitted and received across a network.**

They tell two devices communicating over the internet:

- how data should be **sent**,
- how data should be **received**,
- and how both sides should **behave** during the conversation.

Without shared rules, two machines would have no common language and could not understand each other's bytes.

---

## 2. The Three Layers We Care About

The OSI model has seven layers. For system design we mostly work with three:

| Layer | Protocols | Responsibility |
|---|---|---|
| **Application** (7) | HTTP, HTTPS, WebSockets, FTP, SMTP, WebRTC | *What* is said: the format and rules of the conversation between apps |
| **Transport** (4) | TCP, UDP | *How* it travels: the reliability of the path between two endpoints |
| **Network** (3) | IPv4, IPv6 | *Where* it goes: addressing and routing between devices |

```
 ┌─────────────────────────────┐
 │ Application: HTTP, WebSocket│  ← defines the message and conversation rules
 ├─────────────────────────────┤
 │ Transport:   TCP / UDP      │  ← reliable (or fast) path between client & server
 ├─────────────────────────────┤
 │ Network:     IPv4 / IPv6    │  ← finds the device, routes the packets
 └─────────────────────────────┘
```

---

## 3. A Message's Journey: A Chat Example

Suppose I use a messaging app to send "Hi, how are you?" to a friend. Three problems must be solved, one per layer.

### Problem 1: Find my friend's device (Network layer)

The internet has an enormous number of devices, each with its own IP address. To deliver the message to the right place, we first need the **IP address** of the destination. **IPv4 and IPv6** help with **IP addressing and routing**.

### Problem 2: Get the data there reliably and securely (Transport layer)

Messages do not go straight from my device to my friend's device. They go **through the app's servers**:

```
 Device A ──encrypted──► App Server ──encrypted──► Device B
```

So I first need a **reliable path** and a **secure connection** between Device A and the server. **TCP and UDP** provide this layer's guarantees: they define how data should travel between client and server.

### Problem 3: Define the actual conversation (Application layer)

Even with an address and a reliable path, in what **format** is "Hi, how are you?" sent so that both apps understand it? That is the job of **application-layer protocols**. Chat apps typically use **WebSockets** for client-server communication.

### Summary

| Layer | Role in one line |
|---|---|
| Network | **Finds** the device |
| Transport | **Creates the path** between client and server |
| Application | **Delivers the actual message** in an agreed format |

---

## 4. Transport Layer: TCP

**TCP (Transmission Control Protocol)** provides a **reliable connection**.

Key properties:

- **Reliable delivery:** every packet is expected to arrive; nothing is silently dropped.
- **Ordered delivery:** packets are delivered in a **sequence**; each packet has a sequence number.
- **Acknowledgements:** the receiver acknowledges every data packet.
- **Retransmission:** if a packet is lost or damaged, the sender **sends it again**.
- **Connection-oriented:** a connection is set up before data flows.

### The Three-Way Handshake

Before sending data, client and server establish the connection in three steps:

1. **Introduce:** the client says hello to the server.
2. **Agree on how to talk:** they exchange sequence numbers (and ports).
3. **Confirm readiness:** both sides confirm they are ready.

```
 Client                                   Server
   │ ───── 1. SYN (my sequence no.) ───────► │
   │ ◄──── 2. SYN-ACK (ack + my seq no.) ─── │
   │ ───── 3. ACK ─────────────────────────► │
   │          connection established         │
   │ ════════ data packets ════════════════► │
   │ ◄═══════ ACK per packet ═══════════════ │
```

Analogy: "Hey, how are you?" → "Yes, I'm here, are you ready?" → "Yes, ready." Conversation begins.

### Data transfer

- The client sends data in **packets**.
- For every packet, the server **sends an acknowledgement**.
- If a packet fails to arrive, the client **resends** it, and the server acknowledges it.
- Packets carry sequence numbers so order is preserved.

**Focus of TCP: delivery and order.**

---

## 5. Transport Layer: UDP

**UDP (User Datagram Protocol)** is essentially the opposite of TCP:

- **Connectionless:** no three-way handshake. The sender simply starts sending packets.
- **Unreliable:** if a packet is lost, nobody retries it.
- **No ordering:** packets are not guaranteed to arrive in sequence.
- **No acknowledgements** and no retransmission.

Because it skips all of that bookkeeping, **UDP is fast**.

### Where UDP is used

- **Video calls**
- **Video streaming**
- **Online games**
- (Also live audio, DNS lookups and similar latency-sensitive traffic)

**Why it fits:** in a video call, if a few frames are lost due to a network glitch, you do not want the call to pause and replay them. You want the call to keep going in real time. If frame 2 arrives before frame 1, it barely matters. Fresh data matters more than perfect data.

---

## 6. TCP vs UDP

| Feature | TCP | UDP |
|---|---|---|
| Connection | Connection-oriented (3-way handshake) | Connectionless |
| Reliability | Reliable; lost packets are resent | Unreliable; lost packets are ignored |
| Ordering | Guaranteed order via sequence numbers | No ordering |
| Acknowledgements | Yes, per packet | None |
| Speed | Slower (overhead) | Faster (minimal overhead) |
| Typical uses | Web pages, APIs, email, file transfer, chat | Video calls, streaming, gaming |

**Rule of thumb:** when *correctness* matters, use TCP. When *speed and freshness* matter more than perfection, use UDP.

---

## 7. Application Layer: Client-Server Protocols

Application-layer protocols come in two families:

1. **Client-server protocols**
2. **Peer-to-peer (P2P) protocols**

In a **client-server protocol**:

- The **client always initiates** the connection and the request.
- The **server responds** to that request.
- The server does not speak first.

Examples: **HTTP, HTTPS, FTP, SMTP**. These are **unidirectional in initiation**: the client asks, the server answers, and the exchange ends.

**WebSocket** is also client-server, but it behaves differently (see below).

---

## 8. HTTP and HTTPS

**HTTP** handles the content of your web requests: asking for a web page, submitting login information (email and password), placing an order, and so on. On an e-commerce site, these requests travel over HTTP.

### HTTP is unencrypted

HTTP sends data as plain text, so anyone who can observe the traffic could read or tamper with it. This invites attacks.

### HTTPS adds security

**HTTPS = HTTP + Secure.** It layers an **encryption protocol** on top of HTTP:

- **SSL / TLS** (TLS = **Transport Layer Security**) provides the encryption.
- Requests and responses travel as **encrypted data**.

| | HTTP | HTTPS |
|---|---|---|
| Encryption | None | Yes, via TLS |
| Data on the wire | Readable plain text | Encrypted |
| Typical use | Local or low-risk internal traffic | All serious public applications |

Every large application uses HTTPS.

---

## 9. WebSockets

HTTP, HTTPS, FTP and SMTP are **unidirectional**: the client starts, the server answers, the connection closes.

**WebSocket** is **bidirectional** and **persistent**:

- After a WebSocket connection is established, **client and server can send messages to each other independently**.
- The server does **not** have to wait for a client request. If it has something for the client, it can push it right away.

```
 HTTP:       Client ── request ──► Server
             Client ◄─ response ── Server        (connection ends)

 WebSocket:  Client ◄══════ persistent ══════► Server
             either side can send at any time
```

**Use cases:** WhatsApp-style and Telegram-style chat apps, live notifications, collaborative editing, live scores and trading dashboards.

> **Common confusion:** WebSocket is **not** a peer-to-peer protocol. It is a **client-server** protocol (just bidirectional). In a chat app, two users A and B each hold a WebSocket connection to the *server*, and the server relays messages between them.

---

## 10. Other Client-Server Protocols: FTP and SMTP

| Protocol | Purpose |
|---|---|
| **FTP** (File Transfer Protocol) | Transferring files |
| **SMTP** (Simple Mail Transfer Protocol) | Sending email |

Both follow the client-initiated request/response model.

---

## 11. Peer-to-Peer Protocols and WebRTC

In **client-server** protocols, two clients cannot talk to each other directly. They need a server in between.

In **peer-to-peer** protocols there is **no discrimination between client and server**. Every participant can send messages to any other participant independently, and **Client A and Client B can communicate directly**.

```
 Client-Server:   A ◄──► Server ◄──► B

 Peer-to-Peer:    A ◄──────────────► B
```

The main application-layer P2P protocol covered here is **WebRTC**, used for real-time audio/video and direct data exchange between browsers or devices.

> In practice WebRTC still uses helper servers for *signaling* (introducing peers to each other) and NAT traversal, but the actual media or data flows peer to peer.

---

## 12. What "HTTP Is TCP-Based" Really Means

You will often hear: *"HTTP and HTTPS are TCP-based protocols."*

It means:

1. The application-layer protocol (HTTP/HTTPS) defines **what** is sent: the requests and responses.
2. The transport-layer protocol (TCP) provides **the reliable connection** underneath.
3. So before any HTTP request is sent, client and server perform a **TCP three-way handshake** to set up a reliable connection. Then the HTTP/HTTPS request flows over that connection.

> Newer versions of HTTP (HTTP/3) run over QUIC, which is built on UDP, as a way to cut connection setup latency. The classic and still most common mental model is HTTP over TCP.

---

## 13. Choosing the Right Protocol

| Need | Choose |
|---|---|
| Standard web request/response, REST APIs | HTTPS (over TCP) |
| Real-time two-way messaging (chat, live feed) | WebSocket |
| Direct browser-to-browser audio/video | WebRTC |
| Real-time media where loss is tolerable | UDP-based transport |
| File transfer | FTP (or secure variants) |
| Sending email | SMTP |
| Anything that touches user credentials | Always HTTPS/TLS |

---

## 14. Interview Questions

1. What is a networking protocol?
2. Which layers do IPv4/IPv6, TCP/UDP and HTTP/WebSocket belong to, and what does each layer solve?
3. Explain the TCP three-way handshake.
4. How does TCP guarantee reliability and ordering?
5. Why is UDP faster than TCP? Name three places it is used and why.
6. What is the difference between HTTP and HTTPS? What do SSL/TLS do?
7. What does it mean that "HTTP is TCP-based"?
8. How is WebSocket different from HTTP?
9. Is WebSocket peer-to-peer? Why or why not?
10. How do client-server and peer-to-peer protocols differ? Where does WebRTC fit?
11. For a live video call, would you choose TCP or UDP? Why?

---

## 15. Quick Revision Sheet

- **Protocol** = rules governing data transmission across a network.
- **Network layer** (IPv4/IPv6) → find the device, route packets.
- **Transport layer** (TCP/UDP) → create the path between client and server.
- **Application layer** (HTTP, HTTPS, WebSocket, FTP, SMTP, WebRTC) → define the conversation and deliver the message.
- **TCP:** connection-oriented, 3-way handshake, ACKs, retransmission, ordered, reliable, slower.
- **UDP:** connectionless, no ACK, no retry, no ordering, unreliable, fast. Used for calls, streaming, games.
- **HTTP** = plaintext. **HTTPS** = HTTP + TLS encryption.
- **WebSocket** = bidirectional, persistent, client-server (not P2P).
- **WebRTC** = peer-to-peer.
- **FTP** = files. **SMTP** = email.
- **"TCP-based"** = a TCP handshake happens before HTTP data flows.

---

**Previous:** [Chapter 5 — Proxy vs Reverse Proxy](../Chapter-05-Proxy-vs-Reverse-Proxy/README.md) · **Next:** [Chapter 7 — Scaling](../Chapter-07-Scaling-Horizontal-vs-Vertical/README.md)
