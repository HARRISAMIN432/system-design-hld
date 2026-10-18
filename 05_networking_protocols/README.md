# Chapter 6 — Network Protocols: Network, Transport, and Application Layers

> **Goal:** Understand networking protocols from first principles, trace a message through the networking stack, distinguish TCP from UDP, understand HTTP/HTTPS, WebSockets, and WebRTC, and choose the right communication protocol for real-world system design.

## Table of Contents

1. [What Is a Networking Protocol?](#1-what-is-a-networking-protocol)
2. [Understanding the Networking Layers](#2-understanding-the-networking-layers)
3. [How a Message Travels Through the Network](#3-how-a-message-travels-through-the-network)
4. [Transport Layer: TCP](#4-transport-layer-tcp)
5. [Transport Layer: UDP](#5-transport-layer-udp)
6. [TCP vs UDP](#6-tcp-vs-udp)
7. [Application-Layer Communication Models](#7-application-layer-communication-models)
8. [HTTP and HTTPS](#8-http-and-https)
9. [WebSockets](#9-websockets)
10. [FTP, SMTP, and Other Application Protocols](#10-ftp-smtp-and-other-application-protocols)
11. [Peer-to-Peer Communication and WebRTC](#11-peer-to-peer-communication-and-webrtc)
12. [What Does "HTTP Runs Over TCP" Mean?](#12-what-does-http-runs-over-tcp-mean)
13. [Choosing the Right Protocol](#13-choosing-the-right-protocol)
14. [Common Misconceptions](#14-common-misconceptions)
15. [Interview Questions and Answers](#15-interview-questions-and-answers)
16. [Practice Exercises](#16-practice-exercises)
17. [Quick Revision Sheet](#17-quick-revision-sheet)

---

## 1. What Is a Networking Protocol?

A **networking protocol** is a defined set of rules that determines how devices communicate, exchange data, and interpret the information they receive.

Imagine two people communicating. They need a shared language, an agreed way to start a conversation, and rules for understanding each other's messages. Computers face the same problem, except their communication consists of bits and bytes rather than spoken words.

Protocols define things such as:

- **Message format:** How is the information structured?
- **Addressing:** Where should the information go?
- **Transmission:** How is data sent across the network?
- **Reliability:** What happens when data is lost?
- **Ordering:** How should data be reassembled?
- **Security:** How can the parties authenticate each other and protect data?
- **Communication rules:** When can each side send information, and how should it respond?

Different protocols solve different parts of the problem. No single protocol is responsible for everything.

### Example: Sending a chat message

Suppose you send this message:

```text
Hi, how are you?
```

Several questions must be answered:

1. Which device should receive the data?
2. Which application on that device should receive it?
3. How should the data be transmitted?
4. What happens if some data is lost?
5. How does the receiving application understand the message?
6. How is the message protected from unauthorized observers?

These questions involve different networking mechanisms and protocols working together.

**Key idea:** Networking is a layered system. Each layer solves a particular problem and relies on lower layers to provide the services it needs.

---

## 2. Understanding the Networking Layers

The OSI model defines seven conceptual layers. For practical system design, three important layers are the **network layer, transport layer, and application layer**.

### 2.1 The three important layers

| Layer | Examples | Main responsibility |
|---|---|---|
| Application | HTTP, DNS, SMTP, FTP, WebSocket | Defines application-level messages and communication rules |
| Transport | TCP, UDP | Provides communication between processes on different hosts |
| Network | IPv4, IPv6 | Provides logical addressing and routing between networks |

A useful way to remember them:

- **Network layer:** Where should the packet go?
- **Transport layer:** Which process should receive the data, and what transport guarantees apply?
- **Application layer:** What does the data mean, and how should the applications communicate?

### 2.2 The relationship between the layers

```text
┌───────────────────────────────────────────┐
│ Application Layer                         │
│ HTTP, DNS, SMTP, WebSocket                 │
│ Defines application messages and behavior │
├───────────────────────────────────────────┤
│ Transport Layer                           │
│ TCP, UDP                                  │
│ Process-to-process communication          │
├───────────────────────────────────────────┤
│ Network Layer                             │
│ IPv4, IPv6                                │
│ IP addressing and packet routing          │
└───────────────────────────────────────────┘
```

These layers are conceptual responsibilities, not three separate physical machines. A typical computer implements them through its operating system, networking libraries, and applications.

### 2.3 The network layer: IPv4 and IPv6

The network layer is responsible for moving IP packets from a source host toward a destination host across interconnected networks.

Its main responsibilities include:

- Logical IP addressing.
- Routing packets between networks.
- Forwarding packets toward their destinations.

For example:

```text
Source IP:      192.168.1.10
Destination IP: 203.0.113.20
```

The source and destination IP addresses help the network deliver packets toward the intended host.

However, an IP address alone does not identify the particular application that should receive the data. That is where transport-layer ports become important.

**Important:** IP provides best-effort delivery. It does not guarantee that packets arrive, arrive in order, or arrive only once.

### 2.4 The transport layer: TCP and UDP

The transport layer enables communication between application processes running on different hosts.

For example:

```text
Client IP:       192.168.1.10
Client port:     51520

Server IP:       203.0.113.20
Server port:     443
```

The IP addresses identify the hosts, while the port numbers identify transport endpoints on those hosts.

TCP and UDP provide different transport services:

- **TCP:** A reliable, ordered byte stream.
- **UDP:** Independent datagrams without built-in delivery or ordering guarantees.

Neither protocol encrypts application data by itself.

### 2.5 The application layer

The application layer defines how applications format information and interpret received data.

Examples:

| Protocol | Primary purpose |
|---|---|
| HTTP | Web requests and responses |
| DNS | Resolving domain names to information such as IP addresses |
| SMTP | Sending email |
| FTP | File transfer |
| WebSocket | Persistent, bidirectional client-server messaging |

WebRTC is a real-time communication technology and protocol suite rather than one simple application protocol. It combines multiple protocols to support media and data communication between peers.

### 2.6 A critical distinction: protocols do not all operate at the same level

Consider opening a website:

```text
HTTPS request
    ↓
HTTP application semantics protected by TLS
    ↓
TCP transport for HTTP/1.1 or HTTP/2
    ↓
IP packets
    ↓
Network interfaces and physical links
```

This is the traditional model for HTTPS using HTTP/1.1 or HTTP/2.

HTTP/3 uses a different transport arrangement:

```text
HTTP/3
   ↓
QUIC
   ↓
UDP
   ↓
IP
```

We will explore this distinction in Section 12.

---

## 3. How a Message Travels Through the Network

Understanding the entire journey of a message is more useful than memorizing protocol definitions independently.

Let's follow a message from a chat application.

### 3.1 The architecture

Suppose Alice sends Bob the message `Hi, how are you?`.

```text
┌─────────────┐       ┌─────────────────┐       ┌─────────────┐
│ Alice's App │──────►│  Chat Backend   │──────►│ Bob's App   │
│             │       │                 │       │             │
│ WebSocket   │       │ Auth, routing,  │       │ WebSocket   │
│ connection  │       │ message delivery│       │ connection  │
└─────────────┘       └─────────────────┘       └─────────────┘
```

In this architecture, Alice and Bob each maintain a connection to the backend. The backend receives Alice's message and forwards it to Bob.

### 3.2 Step 1: The application creates a message

The application may represent the message as structured data:

```json
{
  "type": "chat_message",
  "recipientId": "bob",
  "message": "Hi, how are you?"
}
```

This JSON is only an illustrative application payload. A real application could use JSON, Protocol Buffers, another serialization format, or a custom binary protocol.

The application decides what the message means and what fields it contains.

### 3.3 Step 2: The transport layer carries the data

If the application uses WebSockets over a traditional HTTPS-compatible transport, the WebSocket connection typically runs over TCP, with TLS protecting the connection when using `wss://`.

TCP provides an ordered byte stream between the client and server.

It does not understand the meaning of `"recipientId": "bob"`. It only transports bytes.

### 3.4 Step 3: The network layer routes packets

TCP segments are carried inside IP packets.

Each packet has source and destination IP information. Routers forward packets toward their destinations.

A simplified packet looks like this:

```text
┌───────────────────────────────────────────────┐
│ IP Header                                     │
│ Source IP | Destination IP                    │
├───────────────────────────────────────────────┤
│ TCP Header                                    │
│ Source Port | Destination Port | Sequence No. │
├───────────────────────────────────────────────┤
│ Application Data                              │
│ Encrypted WebSocket payload                   │
└───────────────────────────────────────────────┘
```

The exact encapsulation depends on the protocols in use. For HTTPS, TLS records are carried within the transport stream, so the application payload is not generally readable from the network packets without access to the appropriate decryption keys.

### 3.5 Step 4: The receiving machine processes the data

At the destination:

1. The network stack processes the IP packet.
2. The transport layer processes the TCP segment.
3. TCP reconstructs the ordered byte stream.
4. TLS decrypts and authenticates protected records, when TLS is used.
5. The WebSocket implementation processes WebSocket frames.
6. The application interprets the message payload.

The application can then process the chat message.

### 3.6 Encapsulation and decapsulation

The process of adding protocol information as data moves down the stack is called **encapsulation**.

The reverse process at the receiver is called **decapsulation**.

```text
Sender                                  Receiver

Application data                        Application data
      │                                        ▲
      ▼                                        │
TCP segment / UDP datagram             TCP segment / UDP datagram
      │                                        ▲
      ▼                                        │
IP packet                               IP packet
      │                                        ▲
      ▼                                        │
Link-layer frame  ───────────────────► Link-layer frame
                  Network transmission
```

In practice, packets may be forwarded through many routers and link-layer networks. Each link uses its own framing mechanism, while IP provides the logical addressing used across networks.

### 3.7 Does the message travel directly from Alice to Bob?

Not necessarily.

In a typical centralized chat architecture, the message travels from Alice to a backend and then from the backend to Bob. That involves two separate network communication paths.

```text
Alice ── connection A ──► Backend
Backend ── connection B ──► Bob
```

The backend may also store the message, verify permissions, enforce rate limits, and retry delivery when Bob is offline.

**Key takeaway:** A single application message can depend on multiple protocols and multiple network connections.

---

## 4. Transport Layer: TCP

**TCP (Transmission Control Protocol)** provides reliable, ordered, full-duplex communication through a connection-oriented byte stream.

TCP is commonly used when the application needs the transport layer to handle loss recovery and ordering.

Examples include:

- HTTP/1.1 and HTTP/2.
- Traditional HTTPS connections.
- SSH.
- Many database connections.
- Many email and file-transfer connections.

### 4.1 TCP's main properties

#### 1. Connection-oriented

TCP establishes connection state between the communicating endpoints before normal data transfer.

For a typical connection, the endpoints use a three-way handshake.

#### 2. Reliable delivery

TCP uses acknowledgements, sequence numbers, retransmission, and other mechanisms to recover from data loss.

Under normal operating conditions, TCP attempts to deliver the transmitted byte stream reliably. If the connection fails permanently, however, the application may receive an error rather than successful delivery.

#### 3. Ordered delivery

TCP uses sequence numbers to reconstruct bytes in the correct order.

Suppose data is transmitted in segments:

```text
Segment 1: "Hello "
Segment 2: "World"
```

If Segment 2 arrives first, TCP can retain its data until the missing earlier bytes arrive, then provide the reconstructed stream to the application in order.

This can introduce **head-of-line blocking**: later data on the same TCP byte stream may have to wait for missing earlier data.

#### 4. Flow control

TCP prevents a sender from overwhelming the receiving endpoint's available receive buffer.

The receiver advertises a receive window, which helps regulate how much unacknowledged data the sender can have in flight.

Flow control protects the receiver. It is distinct from congestion control, which helps prevent excessive traffic from overwhelming the network.

#### 5. Congestion control

TCP adjusts its sending behavior based on network congestion signals and other observations.

This helps share network capacity and reduce congestion collapse.

#### 6. Full-duplex communication

Both sides of a TCP connection can send data independently.

TCP is not inherently a one-way protocol. An application can still choose a request-response communication pattern on top of it.

### 4.2 The TCP three-way handshake

Before normal TCP data transfer begins, the endpoints establish connection state through a three-way handshake.

```text
Client                                      Server
  │                                            │
  │──────────── SYN ──────────────────────────►│
  │                                            │
  │◄────────── SYN + ACK ─────────────────────│
  │                                            │
  │──────────── ACK ──────────────────────────►│
  │                                            │
  │          Connection established            │
  │                                            │
  │◄══════════ Data transfer ═════════════════►│
```

The three messages are:

1. **SYN:** The client requests connection establishment and communicates its initial sequence number.
2. **SYN-ACK:** The server acknowledges the client's sequence number and communicates its own.
3. **ACK:** The client acknowledges the server's sequence number.

The handshake establishes synchronized connection state. It is not a security handshake and does not, by itself, encrypt the connection or authenticate the server's identity.

TLS can be established afterward when the application requires a secure connection.

### 4.3 How TCP provides reliability

TCP uses several mechanisms together.

**Sequence numbers:** Identify the positions of bytes in the stream.

**Acknowledgements (ACKs):** Tell the sender which data has been received. TCP acknowledgements are not necessarily sent one-for-one for every segment; they may cumulatively acknowledge multiple bytes, and implementations can use additional acknowledgement mechanisms.

**Retransmission:** If the sender infers that data has been lost, it can retransmit it.

**Checksums:** Help detect corruption in TCP segments. Checksums are error-detection mechanisms, not cryptographic integrity protection.

Example:

```text
Sender                              Receiver
  │                                    │
  │──────── Segment A ────────────────►│
  │◄──────────── ACK ─────────────────│
  │                                    │
  │──────── Segment B ──────── X       │
  │          (lost)                    │
  │                                    │
  │──────── Retransmit B ─────────────►│
  │◄──────────── ACK ─────────────────│
```

The exact recovery behavior depends on timers, acknowledgement information, congestion-control state, and the implementation.

### 4.4 TCP is a byte stream, not a message protocol

This is an important distinction.

If an application writes:

```text
write("Hello")
write("World")
```

TCP does not guarantee that the receiver will read exactly two chunks, `"Hello"` and `"World"`.

The receiver might read:

```text
"HelloWorld"
```

or:

```text
"Hel"
"loWor"
"ld"
```

The data remains ordered, but application write boundaries are not preserved.

If an application needs discrete messages, its application protocol must define message boundaries. It can use length prefixes, delimiters, fixed-size records, or another framing strategy.

WebSocket, for example, defines its own message framing on top of its underlying transport.

### 4.5 When is TCP a good choice?

TCP is a good default when the application needs:

- Reliable, ordered byte delivery.
- Automatic retransmission and recovery from ordinary packet loss.
- A mature transport abstraction.
- Compatibility with protocols and infrastructure built around TCP.

Examples include downloading a file, submitting a payment request, and querying a database.

**Important:** TCP reliability does not guarantee that a business operation executes exactly once. If a client sends a payment request and loses the response, it may not know whether the server completed the payment. Application-level idempotency, transaction handling, and retry rules are still necessary.

---

## 5. Transport Layer: UDP

**UDP (User Datagram Protocol)** provides connectionless datagram delivery with minimal built-in transport machinery.

UDP does not establish a TCP-style connection before sending datagrams.

It does not provide built-in guarantees of delivery, ordering, or duplicate suppression.

### 5.1 UDP's main properties

- **Connectionless:** No TCP-style three-way handshake.
- **Datagram-oriented:** Each send operation represents a separate datagram.
- **No built-in retransmission:** Lost datagrams are not automatically resent by UDP.
- **No built-in ordering guarantee:** Datagrams may arrive out of order.
- **No built-in congestion-control algorithm:** Applications and higher-level protocols must use appropriate congestion-control behavior.
- **Low protocol overhead:** UDP has a smaller basic header and fewer built-in transport mechanisms than TCP.

UDP datagrams include source and destination ports, a length, and a checksum field. UDP does not mean that the application must ignore errors or accept unreliable behavior in every respect; applications can implement their own reliability mechanisms when needed.

### 5.2 Why would anyone use an unreliable transport?

Consider a live video call.

The sender produces media continuously:

```text
Time ─────────────────────────────────────►

Frame 1 ──► Frame 2 ──► Frame 3 ──► Frame 4
```

Suppose Frame 2 is lost.

With a latency-sensitive media application, waiting a long time to recover an old frame may be less useful than continuing with newer frames.

The application may choose to tolerate some loss, conceal missing media, or request limited recovery when it can arrive in time.

The objective is not simply maximum raw speed. It is **useful information delivered within the application's latency budget**.

UDP is well suited to systems that need control over these trade-offs.

### 5.3 Common UDP use cases

| Use case | Why UDP can be useful |
|---|---|
| Real-time audio/video | Avoids transport-level retransmission delays for every lost datagram |
| Online games | Supports time-sensitive state updates |
| DNS queries | Enables compact request-response exchanges with low overhead |
| QUIC | Provides a datagram transport foundation for reliability, encryption, and multiplexing implemented above UDP |
| Custom telemetry | Can suit applications that can tolerate loss or implement their own recovery |

UDP is also used by applications that do require reliability at a higher layer. QUIC, for example, uses UDP as its carrier but implements reliable streams, congestion control, encryption, and other transport functions itself.

### 5.4 Is UDP always faster than TCP?

No.

UDP has fewer built-in transport mechanisms, but end-to-end performance depends on the network, application, message size, loss rate, congestion, and implementation.

A UDP application that implements retransmission, congestion control, encryption, and reliable ordering may have substantial complexity of its own.

Likewise, TCP can perform extremely well on reliable, high-bandwidth networks.

The better question is:

> Which transport behavior best matches the application's latency, reliability, and congestion-control requirements?

---

## 6. TCP vs UDP

| Feature | TCP | UDP |
|---|---|---|
| Communication model | Connection-oriented | Connectionless datagrams |
| Initial handshake | TCP three-way handshake | No TCP-style handshake |
| Data abstraction | Ordered byte stream | Individual datagrams |
| Delivery guarantee | Reliable stream delivery while the connection remains viable | No built-in delivery guarantee |
| Ordering | Preserves byte order | No built-in ordering guarantee |
| Retransmission | Built in | Must be implemented above UDP if required |
| Flow control | Built in | Not built in |
| Congestion control | Built in | Not built in at the UDP layer |
| Header size | Minimum 20 bytes, excluding options | 8 bytes |
| Message boundaries | Not preserved | Preserved as datagram boundaries |
| Common uses | Web traffic over HTTP/1.1 and HTTP/2, databases, SSH | DNS, real-time media, QUIC |

### 6.1 Choosing between them

Use TCP when the application benefits from reliable, ordered byte delivery and does not require the transport to expose individual datagrams.

Use UDP when the application benefits from datagrams and needs to control its own reliability, ordering, and timing behavior, or when a higher-level protocol such as QUIC supplies those features.

### 6.2 Why do games often use UDP?

Imagine a game client sends player positions:

```text
Update 101: Position = (10, 20)
Update 102: Position = (11, 20)
Update 103: Position = (12, 21)
```

If Update 101 is lost but Update 103 arrives, the game may be able to use the newer position immediately. Recovering an obsolete position might be less valuable than sending the next update.

However, a game may still use reliable communication for login, inventory changes, purchases, or other operations that must not be silently lost.

The protocol choice can vary by message type, not just by application.

### 6.3 Why do file transfers often use TCP?

A file needs its bytes reconstructed correctly. Missing or reordered sections must be recovered before the file can be considered complete.

TCP provides reliable, ordered delivery, which makes it a natural choice for many file-transfer protocols.

A UDP-based file-transfer design is also possible, but it must implement suitable reliability and congestion-control behavior itself or use a protocol that already provides those features.

---

## 7. Application-Layer Communication Models

Transport protocols describe how data moves between endpoints. Application protocols describe how applications exchange meaningful messages.

Two useful architectural patterns are client-server and peer-to-peer.

### 7.1 Client-server communication

In the client-server model, a client communicates with a service that processes requests or manages a shared resource.

```text
Client A ─────► Server ◄───── Client B
```

Examples include:

- A browser requesting a web page.
- A mobile application requesting account information.
- A client sending a message to a chat backend.
- An email client submitting a message to an email server.

The server may maintain a long-lived connection, send asynchronous events, or support multiple communication patterns. Client-server describes the architecture, not a requirement that every interaction must be a single request followed by a single response.

### 7.2 Request-response communication

HTTP commonly uses a request-response model:

```text
Client                           Server
  │                                 │
  │──────── Request ───────────────►│
  │                                 │
  │◄─────── Response ───────────────│
```

For example:

```http
GET /users/42 HTTP/1.1
Host: api.example.com
```

A server might return:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{"id":42,"name":"Alice"}
```

These examples illustrate HTTP/1.1 message syntax. HTTP/2 and HTTP/3 use different on-the-wire encodings.

The important distinction is that HTTP defines the semantics of requests and responses. It does not require every request to use a new transport connection.

### 7.3 Request-response does not mean one request per connection

HTTP connections can be reused for multiple requests.

For example:

```text
One connection:
    Request 1 → Response 1
    Request 2 → Response 2
    Request 3 → Response 3
```

HTTP/1.1 supports persistent connections. HTTP/2 and HTTP/3 additionally support multiplexing multiple streams over a connection.

A client may also have several connections or streams in flight concurrently.

### 7.4 Bidirectional communication

Some applications need the server to send updates without waiting for a new client request.

Examples include:

- Live chat.
- Real-time notifications.
- Collaborative editing.
- Live dashboards.
- Multiplayer game events.

Possible solutions include polling, Server-Sent Events (SSE), WebSockets, and WebRTC, depending on the requirements.

WebSockets are particularly useful when both the client and server need to send messages over an established connection.

### 7.5 Client-server versus peer-to-peer

```text
Client-server:

       ┌──────────┐
       │  Server  │
       └──────────┘
          ▲    ▲
          │    │
          │    │
       ┌──┘    └──┐
       │          │
    Client A   Client B


Peer-to-peer:

    Peer A ◄────────────► Peer B
```

In client-server architecture, a central service participates in the communication path or manages the interaction.

In peer-to-peer (P2P) architecture, peers can communicate directly when the network environment permits.

P2P systems may still use servers for authentication, discovery, signaling, indexing, or relaying traffic when direct communication is impossible.

---

## 8. HTTP and HTTPS

### 8.1 What is HTTP?

**HTTP (Hypertext Transfer Protocol)** is an application-layer protocol used to transfer web resources and exchange messages between clients and servers.

HTTP defines concepts such as:

- Methods, including `GET`, `POST`, `PUT`, `PATCH`, and `DELETE`.
- Resource identifiers and request targets.
- Headers.
- Status codes.
- Message content.
- Caching and content negotiation.
- Authentication mechanisms and related application semantics.

For example, a client might request a product:

```http
GET /products/123 HTTP/1.1
Host: shop.example.com
Accept: application/json
```

The server may respond:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": 123,
  "name": "Keyboard",
  "price": 49.99
}
```

HTTP does not itself define the business meaning of a product or decide whether a user is allowed to purchase it. The application implements those rules.

### 8.2 Is HTTP encrypted?

Ordinary HTTP without TLS does not provide transport encryption, authentication of the server, or cryptographic integrity protection.

An observer who can access the relevant traffic may be able to read its contents or modify data in transit.

This is why public applications generally use HTTPS.

### 8.3 What is HTTPS?

**HTTPS is HTTP protected by TLS (Transport Layer Security).**

TLS provides cryptographic mechanisms that can provide:

- **Confidentiality:** Protects application data from being read by ordinary network observers.
- **Integrity:** Detects unauthorized modification of protected records.
- **Authentication:** Typically allows the client to verify the server's identity using certificates.

The certificate-validation process, trust configuration, and application behavior all matter. Encryption alone does not prove that an application is trustworthy or that the server is authorized to perform a particular action.

A simplified HTTPS connection is:

```text
Client                                 Server
  │                                       │
  │──── TCP connection setup ────────────►│
  │                                       │
  │◄════ TLS handshake and negotiation ══►│
  │                                       │
  │──── Encrypted HTTP request ──────────►│
  │                                       │
  │◄─── Encrypted HTTP response ──────────│
```

This is the traditional HTTPS flow using HTTP/1.1 or HTTP/2 over TCP.

HTTP/3 uses QUIC over UDP and integrates its security handshake differently.

### 8.4 HTTP vs HTTPS

| Feature | HTTP without TLS | HTTPS |
|---|---|---|
| Application protocol | HTTP | HTTP |
| Transport security | None provided by HTTP | TLS-based protection |
| Confidentiality | No | Yes, for protected traffic |
| Integrity protection | No cryptographic protection from HTTP alone | Provided by TLS |
| Server authentication | Not provided by HTTP itself | Typically provided through certificate validation |
| Appropriate for sensitive public traffic | No | Yes, when configured and used correctly |

HTTPS is not a different application protocol that replaces HTTP's request-response semantics. It is HTTP used with TLS-based security.

### 8.5 Is HTTPS only for login pages?

No. HTTPS should protect the entire public application, not just login forms.

Other pages may expose session cookies, private API data, personal information, or actions that attackers could manipulate.

Modern applications should use HTTPS consistently, with appropriate cookie settings, certificate validation, secure redirects, and other relevant controls.

### 8.6 What about SSL?

You may encounter the phrase "SSL certificate."

SSL (Secure Sockets Layer) is the historical predecessor of TLS. SSL is obsolete and should not be enabled for modern secure connections.

The modern protocol is TLS, although the term "SSL certificate" remains common in everyday usage.

---

## 9. WebSockets

### 9.1 The problem with ordinary request-response interactions

Suppose a browser needs to receive live chat messages.

With polling, the browser repeatedly asks the server whether new messages exist:

```text
Client ── "Any new messages?" ──► Server
Client ◄── "No new messages" ──── Server

Client ── "Any new messages?" ──► Server
Client ◄── "No new messages" ──── Server
```

Frequent polling can waste resources and introduce delays. Long polling reduces some unnecessary requests but still requires request-response handling.

For applications that need continuous bidirectional messaging, WebSockets can be a better fit.

### 9.2 What is WebSocket?

**WebSocket is a protocol for persistent, bidirectional communication between a client and a server over a single established connection.**

After the WebSocket connection is established:

- The client can send messages at any time.
- The server can send messages at any time.
- Neither side needs to initiate a new HTTP request for every message.
- Both sides can continue exchanging messages while the connection remains open.

```text
Client                                      Server
  │                                            │
  │──── Connection establishment ────────────►│
  │                                            │
  │◄══════════ WebSocket open ════════════════►│
  │                                            │
  │──── Chat message ─────────────────────────►│
  │◄─── New message notification ──────────────│
  │──── Typing indicator ─────────────────────►│
  │◄─── Delivery acknowledgement ──────────────│
  │                                            │
  │          Connection remains open           │
```

### 9.3 How is a WebSocket connection established?

A traditional WebSocket connection begins with an HTTP-based opening handshake.

For example, with the HTTP/1.1 upgrade mechanism, the client sends a request containing headers that indicate a WebSocket upgrade.

Conceptually:

```text
HTTP handshake
      │
      ▼
WebSocket connection established
      │
      ▼
WebSocket frames exchanged in both directions
```

After a successful upgrade, the connection carries WebSocket protocol frames rather than ordinary HTTP request-response messages.

WebSockets can also be used with HTTP/2 through the extended CONNECT mechanism. Therefore, the HTTP/1.1 upgrade mechanism is not the only way to establish WebSocket communication.

### 9.4 `ws://` versus `wss://`

| Scheme | Meaning |
|---|---|
| `ws://` | WebSocket without TLS protection |
| `wss://` | WebSocket protected by TLS |

For public applications, `wss://` is generally the appropriate choice.

### 9.5 Where are WebSockets useful?

- Real-time chat.
- Live notifications.
- Collaborative editors.
- Live trading dashboards.
- Multiplayer game communication.
- Real-time operational monitoring.

WebSockets are not automatically the best option for every live-update feature. For one-way server-to-client updates, Server-Sent Events (SSE) may be simpler. Ordinary HTTP requests are often preferable for standard request-response APIs.

### 9.6 Are WebSockets peer-to-peer?

No. WebSockets are commonly used for **client-server communication**.

In a chat application:

```text
Alice's Browser ◄════► Chat Server ◄════► Bob's Browser
```

Alice and Bob each maintain a WebSocket connection to the server. The server routes messages between them.

WebSockets are bidirectional at the connection level, but bidirectional communication does not imply peer-to-peer architecture.

### 9.7 WebSockets and scaling

Persistent connections introduce system-design challenges.

For example:

- A server must maintain state for active connections.
- Idle connections consume some memory and other resources.
- The system needs a strategy for routing messages to the server holding the recipient's connection.
- Deployments and failures require reconnect behavior.
- Large fan-out workloads may require a message broker or distributed event system.

A common architecture is:

```text
Clients
   │
   ▼
Load Balancer
   │
   ├────────► WebSocket Server A ─────┐
   │                                 │
   └────────► WebSocket Server B ────┤
                                     ▼
                              Message Broker
                                     │
                                     ▼
                          Recipient Connection Registry
```

The exact architecture varies. The essential challenge is that the server handling a sender may not be the server holding the recipient's connection.

---

## 10. FTP, SMTP, and Other Application Protocols

Not every application needs HTTP.

Different protocols have evolved to solve different communication problems.

### 10.1 FTP — File Transfer Protocol

FTP was designed for transferring files between a client and a server.

Traditional FTP commonly uses separate control and data connections, which affects firewall and network configuration.

Standard FTP does not encrypt credentials or file contents by default.

For secure file transfer, common alternatives include:

- **SFTP:** SSH File Transfer Protocol, which operates over SSH.
- **FTPS:** FTP protected using TLS.

SFTP and FTPS are different protocols. SFTP is not simply FTP with encryption added.

### 10.2 SMTP — Simple Mail Transfer Protocol

SMTP is used to submit and relay email.

A simplified email flow:

```text
Sender's Mail Client
         │
         │ Submit email
         ▼
Outgoing Mail Server
         │
         │ Relay email
         ▼
Recipient's Mail Server
         │
         │ Retrieve or synchronize
         ▼
Recipient's Mail Client
```

SMTP handles email submission and transfer between mail servers. It is not generally the protocol used by an email client to retrieve messages.

Protocols such as IMAP and POP3 serve email access and retrieval use cases.

### 10.3 DNS — Domain Name System

DNS helps resolve domain names into information used by applications and network infrastructure, including IP addresses.

For example:

```text
api.example.com
       │
       ▼
      DNS
       │
       ▼
203.0.113.20
```

DNS commonly uses UDP for queries, but TCP is also used in cases such as zone transfers and responses that require it. Modern DNS deployments may also use encrypted transports such as DNS over TLS or DNS over HTTPS.

DNS resolution is an important part of the process of accessing a service, but DNS itself does not carry out the application's HTTP request.

### 10.4 Quick comparison

| Protocol | Primary responsibility |
|---|---|
| HTTP | Web and API communication |
| WebSocket | Persistent bidirectional messaging |
| DNS | Name resolution |
| SMTP | Email submission and relay |
| IMAP | Accessing and synchronizing email |
| POP3 | Retrieving email |
| FTP | Traditional file transfer |
| SFTP | File transfer over SSH |
| FTPS | FTP protected with TLS |

---

## 11. Peer-to-Peer Communication and WebRTC

### 11.1 What is peer-to-peer communication?

In a peer-to-peer architecture, two participants can communicate directly rather than sending every application message through a central application server.

For example, a browser-to-browser audio call may use a direct media path:

```text
Browser A ◄════════ Media ════════► Browser B
```

However, direct communication across the public internet is not always possible because of NAT, firewalls, and network policies.

Consequently, practical P2P systems often require supporting infrastructure.

### 11.2 What is WebRTC?

**WebRTC (Web Real-Time Communication)** is a collection of browser APIs and underlying protocols that support real-time audio, video, and data communication.

It is commonly used for:

- Browser-based video calls.
- Voice communication.
- Screen sharing.
- Peer-to-peer data channels.
- Real-time communication between supported applications.

WebRTC is not one standalone transport protocol. It combines several technologies that handle peer discovery, connectivity, security, media, and data transport.

### 11.3 How WebRTC establishes a connection

A simplified connection process looks like this:

```text
Browser A             Signaling Server             Browser B
    │                         │                         │
    │──── Offer ─────────────►│                         │
    │                         │──── Forward offer ─────►│
    │                         │                         │
    │                         │◄──── Answer ────────────│
    │◄──── Forward answer ────│                         │
    │                         │                         │
    │◄──── ICE candidates exchanged via signaling ────►│
    │                                                   │
    │══════ Direct media/data path if possible ════════►│
```

The signaling server is used to exchange connection information. WebRTC does not prescribe one universal signaling protocol; applications can use WebSockets, HTTP, or other mechanisms.

Once the peers have exchanged the necessary information, they attempt to establish a suitable network path.

### 11.4 What are ICE, STUN, and TURN?

These terms are central to understanding WebRTC networking.

**ICE — Interactive Connectivity Establishment**

ICE coordinates the process of gathering candidate network addresses and testing possible connectivity paths.

**STUN — Session Traversal Utilities for NAT**

STUN helps a device discover information about its public-facing network address as observed by an external server. It can help establish direct connectivity, but it does not solve every NAT or firewall configuration.

**TURN — Traversal Using Relays around NAT**

TURN provides a relay when a direct path cannot be established or is otherwise unsuitable.

```text
Preferred path:

Peer A ◄══════════════════════► Peer B


Fallback when direct connectivity fails:

Peer A ◄════════ TURN Relay ═════════► Peer B
```

A TURN relay can consume substantial bandwidth because media or data flows through the relay rather than directly between peers.

### 11.5 How does WebRTC transport media and data?

WebRTC uses different protocols for different purposes.

- Real-time audio/video commonly uses secure RTP (SRTP).
- WebRTC data channels use SCTP over DTLS over UDP.
- DTLS provides cryptographic protection for the relevant transport.
- ICE helps establish the connectivity path.
- TURN may relay traffic when direct connectivity is unavailable.

This is more accurate than saying that WebRTC simply "uses UDP." Its behavior involves a protocol suite, and its actual path may be direct or relayed.

### 11.6 WebSocket versus WebRTC

| Feature | WebSocket | WebRTC |
|---|---|---|
| Typical communication model | Client-server | Peer-to-peer when connectivity permits |
| Common use | Chat, notifications, live dashboards | Audio, video, screen sharing, peer data |
| Connection setup | WebSocket opening handshake | Signaling plus ICE connectivity checks |
| Media support | Application must define media transport and handling | Built for real-time media and data use cases |
| Supporting server | WebSocket server handles the connection | Signaling infrastructure; STUN/TURN may be required |
| Network path | Client to WebSocket server | Direct peer path or relayed path |

### 11.7 Which should you choose?

Choose WebSockets when your application needs persistent bidirectional messaging with a server.

Choose WebRTC when your application needs real-time audio/video or peer data communication with suitable latency characteristics.

A real application may use both:

```text
                 Signaling
Browser A ◄────────────────────► Signaling Server
    │                                  │
    │                                  │
    └══════ Real-time media/data ══════► Browser B
```

For example, WebSockets might carry signaling messages while WebRTC carries the actual audio and video.

---

## 12. What Does "HTTP Runs Over TCP" Mean?

This phrase describes the relationship between an application protocol and its transport protocol.

HTTP defines application-level semantics. TCP, when used underneath HTTP, transports the bytes reliably and in order.

### 12.1 HTTP/1.1 and HTTP/2 over TCP

For a traditional HTTPS connection using HTTP/1.1 or HTTP/2:

```text
Application
    │
    ▼
HTTP
    │
    ▼
TLS
    │
    ▼
TCP
    │
    ▼
IP
    │
    ▼
Network
```

The simplified connection sequence is:

1. The client resolves the server's domain name if necessary.
2. The client establishes a TCP connection.
3. TLS is negotiated to protect the connection.
4. The client sends HTTP requests.
5. The server sends HTTP responses.

There are implementation-specific details and optimizations, but this model explains the main relationship.

### 12.2 HTTP/3 is different

HTTP/3 uses QUIC rather than TCP.

```text
HTTP/3
   │
   ▼
QUIC
   │
   ▼
UDP
   │
   ▼
IP
```

QUIC is a transport protocol built on UDP. It provides features such as reliable streams, congestion control, encryption, and multiplexing.

Therefore, saying "HTTP always uses TCP" is incorrect.

A better statement is:

- HTTP/1.1 and HTTP/2 are commonly carried over TCP.
- HTTP/3 uses QUIC over UDP.

### 12.3 Why was QUIC introduced?

TCP has useful reliability and ordering properties, but some transport behaviors can affect latency.

For example, loss of a TCP segment can delay delivery of later bytes on that same stream. QUIC provides independent streams with separate stream-level ordering, reducing this form of cross-stream head-of-line blocking.

QUIC also integrates TLS 1.3-based security into its connection establishment, enabling useful reductions in setup latency under appropriate conditions.

QUIC does not eliminate all latency or packet-loss problems, and loss can still affect streams that depend on the missing data.

### 12.4 A common system-design interview trap

**Question:** If HTTP/3 uses UDP, does that mean HTTP/3 is unreliable?

**Answer:** No. UDP itself does not provide reliable delivery, but QUIC implements reliability for its streams, along with congestion control, encryption, and other transport features. The properties of HTTP/3 therefore cannot be inferred from UDP alone.

The important distinction is between the underlying datagram carrier and the transport protocol implemented above it.

---

## 13. Choosing the Right Protocol

Protocol selection should begin with the application's requirements, not a simplistic rule that one protocol is always faster.

Ask these questions:

1. Does the application need reliable, ordered data delivery?
2. Does it need to preserve message boundaries?
3. Does it need low-latency, bidirectional communication?
4. Is the communication client-server or peer-to-peer?
5. Does the application need encryption and authentication?
6. Does the data become useless if it arrives late?
7. How should the application behave when packets are lost or a connection breaks?

### 13.1 Practical decision table

| Requirement | Common choice | Reason |
|---|---|---|
| REST API | HTTPS | Standard request-response communication with transport security |
| Traditional web browsing | HTTPS using HTTP/1.1, HTTP/2, or HTTP/3 | Web resource transfer and security |
| Real-time server-to-client updates only | SSE over HTTPS | Simple one-way event streaming |
| Bidirectional client-server chat | WebSocket over TLS | Persistent, bidirectional messaging |
| Browser-to-browser audio/video | WebRTC | Real-time media and connectivity mechanisms |
| Reliable database communication | The database's supported protocol, commonly over TCP or TLS | Reliable transport and database-specific semantics |
| Secure remote administration | SSH | Encrypted remote login and command execution |
| Email submission and relay | SMTP with appropriate TLS support | Email transfer |
| Secure file transfer | SFTP or FTPS | File transfer with security |
| Latency-sensitive multiplayer updates | Often UDP-based networking | Application-controlled handling of timing and loss |
| Reliable streams over a UDP carrier | QUIC | Reliability, congestion control, and multiplexed streams above UDP |

### 13.2 Scenario: Designing a chat application

Suppose you are designing a chat application with millions of users.

A possible architecture is:

```text
Mobile/Web Clients
        │
        │ HTTPS: login, history, profile
        ▼
   API Gateway
        │
        ▼
    Chat Backend
        ▲
        │ WebSocket connections
        │
Mobile/Web Clients

Chat Backend
     │
     ▼
Message Broker / Database
```

A practical protocol strategy might be:

- Use HTTPS for authentication, account operations, and message-history APIs.
- Use WebSockets for real-time client-server events.
- Use a database or durable messaging system to store and distribute messages.
- Use TLS to protect public network traffic.
- Implement application-level acknowledgements, retries, and idempotency where required.

The WebSocket connection alone does not guarantee message persistence, delivery to offline users, or exactly-once business processing. Those behaviors belong to the broader application architecture.

### 13.3 Scenario: Designing a video-calling application

A possible strategy:

- Use HTTPS for authentication and call management.
- Use a signaling channel, such as WebSocket, to exchange call setup information.
- Use WebRTC to establish secure real-time media connections.
- Use STUN to assist with connectivity discovery.
- Use TURN relays when direct communication cannot be established.

This combination separates the control plane (setting up the call) from the media/data plane (carrying the actual communication).

### 13.4 Scenario: Designing a payment API

For a payment API:

- Use HTTPS to protect credentials and payment data in transit.
- Validate authorization and input on the server.
- Use application-level idempotency keys or equivalent safeguards for safe retries.
- Store payment state durably and manage transactions correctly.
- Return a clear result to the client.

TCP helps transport request bytes reliably, but it cannot guarantee that a payment is performed exactly once. That requires application-level design.

---

## 14. Common Misconceptions

### Misconception 1: TCP guarantees that every request succeeds

**Incorrect.** TCP provides reliable, ordered byte delivery while the connection remains viable. It does not guarantee that an application receives a response, commits a transaction, or successfully completes a business operation.

A client may lose its connection after a server commits a payment but before the response arrives.

### Misconception 2: TCP acknowledges every packet individually

**Incorrect.** TCP uses acknowledgements to communicate received byte ranges, commonly using cumulative acknowledgements. Acknowledgement behavior is more nuanced than one ACK per data segment.

### Misconception 3: UDP is always faster than TCP

**Incorrect.** UDP has less built-in transport functionality, but actual performance depends on the application, network, loss, congestion, and protocol implementation.

### Misconception 4: UDP means the application cannot be reliable

**Incorrect.** An application can implement acknowledgements, retransmissions, ordering, or other reliability mechanisms above UDP. QUIC is a prominent example of a reliable transport protocol built on UDP.

### Misconception 5: HTTP closes the connection after every response

**Incorrect.** HTTP supports persistent connections. HTTP/2 and HTTP/3 also support multiplexing multiple streams over a connection.

### Misconception 6: HTTPS is a different application protocol from HTTP

**Misleading.** HTTPS is HTTP protected by TLS. The HTTP semantics remain, while TLS adds security properties.

### Misconception 7: WebSocket is peer-to-peer because both sides can send messages

**Incorrect.** Bidirectional communication does not imply a peer-to-peer architecture. WebSockets commonly connect clients to a server that handles their messages.

### Misconception 8: WebRTC always establishes a direct peer-to-peer connection

**Incorrect.** Direct connectivity is not always possible. WebRTC can use TURN relays when a direct path cannot be established or is unsuitable.

### Misconception 9: WebRTC is just UDP

**Incorrect.** WebRTC uses a suite of protocols and mechanisms for signaling, connectivity, security, media, and data. UDP is part of common transport paths, but it is not the whole system.

### Misconception 10: An IP address identifies the exact application

**Incorrect.** IP addresses identify hosts or network interfaces in the relevant addressing context. Transport ports help direct data to the appropriate transport endpoint, and the operating system delivers it to the relevant socket.

### Misconception 11: A successful TCP send means the server processed the data

**Incorrect.** A successful local write generally indicates that the local networking stack accepted the data for transmission. It does not prove that the remote application processed it or committed the corresponding operation.

### Misconception 12: Encryption alone makes an application secure

**Incorrect.** TLS protects traffic in transit, but applications still need authentication, authorization, input validation, secure session handling, and appropriate data-protection practices.

---

## 15. Interview Questions and Answers

### Q1. What is a networking protocol?

A networking protocol is a set of rules governing how communicating devices format, transmit, receive, and interpret data. Different protocols solve different problems, such as routing, reliable delivery, encryption, and application messaging.

### Q2. What is the difference between the network, transport, and application layers?

- The **network layer** provides IP addressing and routing.
- The **transport layer** provides process-to-process communication, with TCP and UDP offering different behaviors.
- The **application layer** defines the messages and communication semantics understood by applications.

### Q3. What is the difference between an IP address and a port number?

An IP address identifies a network endpoint or host address. A transport port identifies a transport endpoint on that host.

Together, IP addresses and ports help identify the endpoints of a TCP or UDP communication.

### Q4. Explain the TCP three-way handshake.

1. The client sends SYN with its initial sequence number.
2. The server responds with SYN-ACK, acknowledging the client's sequence number and providing its own.
3. The client sends ACK, acknowledging the server's sequence number.

The handshake establishes TCP connection state and synchronizes sequence-number information.

### Q5. How does TCP provide reliable delivery?

TCP uses sequence numbers, acknowledgements, retransmissions, checksums, and other mechanisms to detect and recover from transmission problems. It also provides ordered byte-stream delivery.

Reliability is subject to connection viability; TCP cannot guarantee that a permanently failed connection will deliver all pending data.

### Q6. What is the difference between TCP and UDP?

TCP provides a reliable, ordered byte stream with built-in flow control and congestion control. UDP provides datagrams without built-in delivery or ordering guarantees.

UDP can be useful for latency-sensitive applications or as the foundation for a higher-level transport such as QUIC.

### Q7. Why is UDP commonly used for real-time media?

Late media may be less useful than newer media. UDP allows applications to choose how to handle packet loss, timing, and recovery rather than requiring TCP's ordered byte-stream delivery.

Real-time applications still need appropriate congestion control and media-recovery strategies.

### Q8. Is TCP full-duplex?

Yes. Both endpoints can send data independently over the same TCP connection. TCP is not inherently a request-only or one-way protocol.

### Q9. Does TCP preserve application message boundaries?

No. TCP is a byte stream. Application writes and reads do not necessarily correspond one-to-one.

An application protocol must define framing if it needs discrete messages.

### Q10. What is the difference between HTTP and HTTPS?

HTTP defines web request-response semantics. HTTPS uses HTTP with TLS protection, providing confidentiality, integrity, and typically server authentication when certificates are validated correctly.

### Q11. What is the difference between HTTP and WebSocket?

HTTP commonly uses request-response semantics, while WebSocket provides persistent bidirectional messaging after connection establishment.

Both can use client-server architecture, and WebSocket establishment can involve HTTP mechanisms.

### Q12. Is WebSocket peer-to-peer?

No. WebSocket is a communication protocol commonly used between a client and a server. The server can relay messages between connected clients.

### Q13. What is WebRTC?

WebRTC is a collection of browser APIs and protocols for real-time audio, video, and data communication. It uses mechanisms such as ICE, DTLS, and SRTP, and may use TURN relays when direct connectivity is unavailable.

### Q14. Why does WebRTC need STUN and TURN?

STUN helps discover public-facing network-address information and can assist with direct connectivity. TURN relays traffic when a direct path cannot be established or is unsuitable.

### Q15. Does HTTP always run over TCP?

No. HTTP/1.1 and HTTP/2 are commonly carried over TCP, while HTTP/3 uses QUIC over UDP.

### Q16. If HTTP/3 uses UDP, how can it provide reliability?

QUIC implements reliable streams, congestion control, and other transport features above UDP. HTTP/3's behavior depends on QUIC, not on UDP alone.

### Q17. Can UDP be used for reliable file transfer?

Yes. An application can implement acknowledgements, retransmission, ordering, and congestion control above UDP, or use an existing protocol such as QUIC. The application must account for the reliability requirements it needs.

### Q18. What happens if a TCP connection fails after the server processes a request?

The client may not know whether the operation completed. It must use application-level recovery mechanisms, such as querying operation status or retrying with an idempotency key.

This is a distributed-systems problem, not something TCP alone can solve.

### Q19. What is the difference between flow control and congestion control?

- **Flow control** prevents a sender from overwhelming the receiver.
- **Congestion control** helps prevent excessive traffic from overwhelming the network.

TCP provides mechanisms for both.

### Q20. How would you choose between WebSockets and Server-Sent Events?

Use Server-Sent Events when the main requirement is a server-to-client event stream and ordinary HTTP request-response communication is sufficient for client-to-server actions.

Use WebSockets when both sides need to send application messages independently over a persistent connection.

---

## 16. Practice Exercises

Use these exercises to check whether you can apply the concepts instead of simply memorizing definitions.

### Exercise 1: Identify the layer

For each item, identify its primary layer or role.

- IPv6
- TCP
- UDP
- HTTP
- DNS
- TLS
- WebSocket
- QUIC

### Exercise 2: Trace an HTTPS request

A browser opens `https://api.example.com/users/42`.

Explain the roles of:

1. DNS resolution.
2. IP addressing and routing.
3. TCP connection establishment for HTTP/1.1 or HTTP/2.
4. TLS negotiation.
5. HTTP request and response processing.

Then explain how the transport arrangement changes when the browser uses HTTP/3.

### Exercise 3: Design a chat system

Design a basic chat application with the following requirements:

- Users can log in.
- Messages are delivered in real time.
- Messages should remain available when recipients are offline.
- The application must support reconnecting after network failures.

Explain which protocols you would use and which requirements must be implemented at the application or storage layer.

### Exercise 4: Diagnose a slow video call

A video call becomes choppy whenever a packet is lost.

Explain why TCP's ordered byte-stream behavior might contribute to latency. Then describe how UDP-based media transport and WebRTC's recovery mechanisms can help, including the trade-offs involved.

### Exercise 5: Understand a failed payment request

A client sends a payment request. The server commits the payment, but the connection breaks before the client receives the response.

Answer these questions:

- Did TCP necessarily fail to deliver the request?
- Does the client know whether the payment succeeded?
- How could an idempotency key help?
- Why is transport reliability different from exactly-once business execution?

### Exercise 6: Explain the protocol stack

Draw a diagram showing how a WebSocket message sent over `wss://` reaches a server, including the roles of WebSocket, TLS, TCP, IP, and link-layer framing.

Then explain why TCP cannot interpret the message's application-level meaning.

### Exercise 7: Choose the transport

Choose an appropriate approach for each case and justify your answer:

| Scenario | What matters most? |
|---|---|
| Downloading a large file | Complete, correctly ordered data |
| Sending frequent game-position updates | Freshness and latency |
| Sending a payment request | Security, correct processing, safe retries |
| Browser-based video call | Real-time media and connectivity |
| Live stock-price dashboard | Efficient server-to-client updates |
| Browser chat | Persistent bidirectional messaging |

Do not assume that each scenario has exactly one valid solution. Explain the requirements and trade-offs that influence the decision.

---

## 17. Quick Revision Sheet

### Networking layers

- **Network layer:** IPv4 and IPv6 provide logical addressing and routing.
- **Transport layer:** TCP and UDP provide process-to-process communication with different guarantees.
- **Application layer:** HTTP, DNS, SMTP, and WebSocket define application-level communication rules.

### TCP

- Connection-oriented.
- Uses a three-way handshake.
- Provides a reliable, ordered byte stream while the connection remains viable.
- Uses acknowledgements, sequence numbers, and retransmissions.
- Provides flow control and congestion control.
- Does not preserve application message boundaries.
- Does not guarantee successful business-operation execution.

### UDP

- Connectionless datagrams.
- No built-in delivery, ordering, or retransmission guarantees.
- Lower basic protocol overhead than TCP.
- Useful for real-time media, DNS, and protocols such as QUIC.
- Can support reliable communication when an appropriate higher-level protocol implements it.

### Application protocols

- **HTTP:** Web and API request-response semantics.
- **HTTPS:** HTTP protected by TLS.
- **WebSocket:** Persistent, bidirectional client-server messaging.
- **WebRTC:** Real-time audio/video and data communication using a protocol suite.
- **DNS:** Domain-name resolution.
- **SMTP:** Email submission and relay.
- **FTP:** Traditional file transfer.
- **SFTP:** File transfer over SSH.
- **SSE:** Server-to-client event streaming over HTTP.

### Essential distinctions

- **IP address vs port:** Host/network endpoint addressing vs transport endpoint identification.
- **TCP reliability vs business correctness:** Reliable byte delivery does not guarantee that an application operation succeeded exactly once.
- **WebSocket vs WebRTC:** Persistent client-server messaging vs real-time peer communication when connectivity permits.
- **UDP vs QUIC:** UDP provides datagrams; QUIC implements additional transport capabilities above UDP.
- **HTTP/1.1 and HTTP/2 vs HTTP/3:** Commonly use TCP for the former two; HTTP/3 uses QUIC over UDP.
- **Encryption vs authentication:** Encryption protects confidentiality; authentication helps establish identity and trust.
- **Flow control vs congestion control:** Protecting the receiver vs managing network congestion.

---

## Final Mental Model

When analyzing a networking problem, ask three questions:

1. **Where does the data need to go?** Think IP addressing and routing.
2. **What transport behavior does the application need?** Think TCP, UDP, or a higher-level transport such as QUIC.
3. **What do the messages mean, and how should the participants communicate?** Think HTTP, WebSocket, DNS, SMTP, or a real-time communication protocol suite.

Then consider the requirements that cut across these layers: security, latency, reliability, scalability, and recovery from failure.

Mastering these distinctions makes it easier to reason about API design, load balancing, distributed systems, real-time applications, and network troubleshooting.

---
