---
layout: default
title: "1.1 Networking Fundamentals"
parent: "Phase 1: Foundations"
nav_order: 1
---

# Networking Fundamentals
{: .no_toc }

<details open markdown="block">
  <summary>Table of Contents</summary>
  {: .text-delta }
1. TOC
{:toc}
</details>

---

## OSI Model

The OSI (Open Systems Interconnection) model is a conceptual framework that divides network communication into 7 layers. Each layer handles one specific responsibility and communicates only with the layers directly above and below it. In practice the internet uses the 4-layer TCP/IP model — but OSI vocabulary is universal in system design interviews.

### The 7 Layers

| Layer | Name | Responsibility | Protocols / Examples | PDU |
|:------|:-----|:---------------|:---------------------|:----|
| 7 | **Application** | User-facing protocols and data formats | HTTP, gRPC, DNS, SMTP, FTP | Message |
| 6 | **Presentation** | Encoding, encryption, compression | TLS/SSL, JSON, XML, Base64 | Message |
| 5 | **Session** | Session establishment and maintenance | RPC session management, NetBIOS | Message |
| 4 | **Transport** | End-to-end delivery, port multiplexing | TCP, UDP, QUIC | Segment |
| 3 | **Network** | Logical addressing and routing | IP (IPv4/IPv6), ICMP, BGP | Packet |
| 2 | **Data Link** | Physical addressing, error detection | Ethernet, Wi-Fi (802.11), ARP | Frame |
| 1 | **Physical** | Bit transmission over a medium | Fiber, coaxial, radio, RJ45 | Bit |

### OSI vs TCP/IP

The internet uses the **TCP/IP model** (4 layers). OSI is a reference model — useful for reasoning, not implementation:

```
OSI                     TCP/IP
─────────────────────────────────────────────
7. Application    ─┐
6. Presentation   ─┤──  Application  (HTTP, DNS, TLS, gRPC)
5. Session        ─┘
4. Transport      ────  Transport    (TCP, UDP, QUIC)
3. Network        ────  Internet     (IP, ICMP, BGP)
2. Data Link      ─┐
1. Physical       ─┘──  Network Access (Ethernet, Wi-Fi)
```

### How a Packet Flows (Encapsulation)

Data is **encapsulated** going down the stack on the sender and **de-encapsulated** going up on the receiver:

```
Sender side (going down):

  Application (7):  [ HTTP: GET /api/users ]
  Transport (4):    [ TCP: src:54321 dst:443 | HTTP ]
  Network (3):      [ IP: src:1.2.3.4 dst:5.6.7.8 | TCP | HTTP ]
  Data Link (2):    [ Ethernet: src-MAC dst-MAC | IP | TCP | HTTP | FCS ]
  Physical (1):     101100100110...  (bits on wire or radio)

Receiver side (going up): each layer strips its own header and passes the payload up.
```

### Why OSI Matters for System Design

The layer a device operates at determines what it can inspect and act on:

| Device / Component | OSI Layer | Capability |
|:-------------------|:----------|:-----------|
| **L4 Load Balancer** (AWS NLB) | Transport (4) | Routes by IP + port only. Cannot see URL or headers. |
| **L7 Load Balancer** (AWS ALB, Nginx) | Application (7) | Routes by URL path, Host header, cookies. Terminates TLS. |
| **Stateful Firewall** | Transport (4) | Allows/blocks by IP, port, TCP state. |
| **WAF** (Web Application Firewall) | Application (7) | Inspects HTTP payload. Blocks SQLi, XSS, CSRF. |
| **Router** | Network (3) | Forwards packets by IP. Unaware of port or application. |
| **Switch** | Data Link (2) | Forwards frames by MAC address. Unaware of IP. |

{: .note }
**Interview shorthand:** L4 = IP + port visibility only; L7 = full HTTP visibility. This distinction drives most load balancer, firewall, and API gateway design decisions.

---

## TCP vs UDP

### TCP (Transmission Control Protocol)

**What:** Connection-oriented, reliable, ordered byte stream.

**How it works — 3-Way Handshake:**
```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server
    C->>S: SYN (Synchronize)
    S->>C: SYN-ACK
    C->>S: ACK (Acknowledgement)
    Note over C,S: Connection established — data flows
```

**Connection Teardown (4-Way):**
```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server
    C->>S: FIN (Finish) (client done sending)
    S->>C: ACK (Acknowledgement)
    S->>C: FIN (Finish) (server done sending)
    C->>S: ACK (Acknowledgement)
    Note over C: TIME_WAIT: 2×MSL ≈ 120 s
```

{: .important }
**TIME_WAIT and high-connection servers:** At massive scale (e.g., 65K ports), `TIME_WAIT` exhausts ephemeral ports. Solutions: `SO_REUSEADDR`, `SO_REUSEPORT`, or connection pooling.

**TCP Features relevant to system design:**

| Feature | Mechanism | System Design Impact |
|:--------|:----------|:---------------------|
| Reliability | ACKs + retransmission | Adds latency on packet loss |
| Ordering | Sequence numbers | Head-of-line blocking within a connection |
| Flow Control | Receive window (rwnd) | Slow receiver can throttle sender |
| Congestion Control | CWND (Cubic, BBR) | Bandwidth utilization in WAN |
| Nagle's Algorithm | Batch small writes | Can hurt latency — disable with `TCP_NODELAY` |

### UDP (User Datagram Protocol)

**What:** Connectionless, unreliable, no ordering guarantee. Just sends datagrams.

**Why it exists:** Zero connection overhead, no head-of-line blocking, lower latency.

**Use UDP when:**
- Latency matters more than reliability: gaming, VoIP, video calls
- You implement reliability yourself at a higher layer (QUIC does this)
- Broadcast/multicast: service discovery (mDNS), DNS queries
- Streaming where dropping old frames is better than delaying (live video)

**TCP vs UDP at a glance:**

| Concern | TCP | UDP |
|:--------|:----|:----|
| Connection setup | 3-way handshake | None |
| Reliability | Guaranteed | Best-effort |
| Ordering | Yes | No |
| Head-of-line blocking | Yes (within stream) | No |
| Overhead | ~20 bytes header | ~8 bytes header |
| Use cases | HTTP, SSH, DB connections | DNS, gaming, QUIC, video |

---

## HTTP/1.1 vs HTTP/2 vs HTTP/3

### HTTP/1.0

Released 1996. The first widely-used version.

**Key characteristics:**
- **One request per TCP connection** — a new TCP 3-way handshake for every single resource
- **No persistent connections** — connection closed immediately after each response
- **No virtual hosting** — no `Host` header, so one IP = one website
- **No chunked transfer** — full `Content-Length` required before sending; couldn't stream responses

**Request/Response model:**
```
Client                          Server
  |--- TCP SYN ----------------->|
  |<-- TCP SYN-ACK ---------------|
  |--- TCP ACK ----------------->|   ← 1 RTT to establish connection
  |                               |
  |--- GET /index.html ---------->|
  |<-- 200 OK (HTML) -------------|   ← 1 RTT for request + response
  |--- TCP FIN ----------------->|   ← connection closed
  |                               |
  |--- TCP SYN ----------------->|   ← NEW connection for next resource!
  |<-- TCP SYN-ACK ---------------|
  |--- TCP ACK ----------------->|   ← 1 RTT wasted again
  |--- GET /style.css ----------->|
  |<-- 200 OK (CSS) --------------|
  |--- TCP FIN ----------------->|
```

**Cost on a 100-asset page:** 100 TCP handshakes × 1 RTT = 100 extra RTTs just for connection setup.

### HTTP/1.1

Released 1997. Dramatically improved HTTP/1.0 but still has fundamental limitations.

**What it added over HTTP/1.0:**

**1. Persistent connections (`Keep-Alive`) — default on**

The same TCP connection is reused for multiple requests. Eliminates per-request handshake overhead.

```
Client                          Server
  |--- TCP SYN ----------------->|
  |<-- TCP SYN-ACK ---------------|
  |--- TCP ACK ----------------->|   ← 1 RTT (once per connection)
  |                               |
  |--- GET /index.html ---------->|
  |<-- 200 OK (HTML) -------------|   ← request 1
  |--- GET /style.css ----------->|
  |<-- 200 OK (CSS) --------------|   ← request 2 (same connection!)
  |--- GET /app.js ------------->|
  |<-- 200 OK (JS) --------------|   ← request 3
```

**2. Chunked Transfer Encoding**

Server can start sending before it knows the total size. Critical for streaming responses.

```
HTTP/1.1 200 OK
Transfer-Encoding: chunked

1a\r\n                    ← chunk size in hex (26 bytes)
This is the first chunk\r\n
d\r\n                     ← next chunk (13 bytes)
Second chunk!\r\n
0\r\n                     ← zero-length chunk = end of body
\r\n
```

**3. `Host` header — mandatory**

Enables virtual hosting: one IP can serve multiple domains.

```
GET /path HTTP/1.1
Host: api.example.com      ← server uses this to route to the right site
```

**4. Pipelining (theoretical)**

Client sends multiple requests without waiting for responses. Responses must come back in the same order.

```
Client → GET /a
Client → GET /b         ← sent without waiting for /a response
Client → GET /c

Server → Response /a    ← must respond in order
Server → Response /b
Server → Response /c
```

**Problem: Head-of-Line (HoL) blocking** — if `/a` is slow (large file, DB query), `/b` and `/c` wait even if they're ready. In practice, pipelining was disabled in most browsers due to this.

**The real workaround — domain sharding:**

Browsers open **6 parallel TCP connections per hostname**. Developers spread assets across subdomains (`cdn1.example.com`, `cdn2.example.com`) to get 12–18 parallel connections. A hack on top of a hack.

**HTTP/1.1 request/response format:**

```
GET /api/users HTTP/1.1
Host: api.example.com
User-Agent: Mozilla/5.0 (Windows NT 10.0) Chrome/120.0
Accept: application/json
Accept-Language: en-US,en;q=0.9
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...   ← repeated on EVERY request!
Cookie: session_id=abc123; preferences=dark       ← repeated on EVERY request!

HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 342
Cache-Control: max-age=60
Date: Thu, 01 May 2026 12:00:00 GMT

{"users": [...]}
```

**Fundamental HTTP/1.1 problems:**

| Problem | Root Cause | Scale Impact |
|:--------|:-----------|:-------------|
| HoL blocking | Sequential responses on one connection | Slow resources stall fast ones |
| Redundant headers | No compression, stateless text protocol | +400–800 bytes overhead per request |
| Domain sharding workaround | Need multiple TCP connections | Multiple handshakes, no prioritization |
| No server push | Request-response only | Client must discover sub-resources |

**Problem:** A page with 100 assets required either 100 sequential requests or 6×100/6 ≈ 17 rounds per connection, each with full uncompressed headers repeated verbatim.

### HTTP/2

Released 2015. Solves HTTP/1.1 multiplexing problem.

**Key innovations:**
- **Multiplexing:** Multiple streams over a single TCP connection. Frames interleaved.
- **Header Compression (HPACK):** Huffman encoding + header table. 80–90% header size reduction.
- **Stream Prioritization:** Assign weights (1–256) and dependencies to streams.
- **Server Push:** Server can proactively send CSS/JS before client requests it.
- **Binary framing layer:** Data split into frames (HEADERS, DATA, SETTINGS, PUSH_PROMISE...)

**HTTP/2 Stream multiplexing:**
```
TCP Connection
├── Stream 1: GET /api/users     (frames interleaved)
├── Stream 3: GET /api/posts
├── Stream 5: GET /static/app.js
└── Stream 7: GET /static/app.css
```

**Still has a problem:** TCP HoL blocking. A lost TCP packet blocks ALL streams while waiting for retransmission — even streams that have no data in the lost packet.

### HTTP/2 on Mobile Networks

HTTP/2's multiplexing, designed to improve performance, actually makes things **worse** on mobile networks compared to HTTP/1.1.

**Why mobile networks are lossy:**

| Network Type | Typical Packet Loss Rate |
|:------------|:------------------------|
| Fiber/LAN | <0.01% |
| Broadband | 0.01–0.1% |
| LTE (good signal) | 0.5–2% |
| LTE (weak signal / handoff) | 2–10% |
| Switching towers (handoff) | 5–30% burst |

**The multiplexing paradox — HTTP/1.1 vs HTTP/2 on lossy connections:**

```
HTTP/1.1 with domain sharding (6 parallel TCP connections):
┌─────────────────────────────────────────┐
│ Connection 1: stream A  ← packet lost   │ ← only this connection stalls
│ Connection 2: stream B  ← OK            │ ← continues normally
│ Connection 3: stream C  ← OK            │ ← continues normally
│ Connection 4: stream D  ← OK            │ ← continues normally
│ Connection 5: stream E  ← OK            │ ← continues normally
│ Connection 6: stream F  ← OK            │ ← continues normally
└─────────────────────────────────────────┘
Impact: 1/6 of streams blocked

HTTP/2 with a single TCP connection:
┌─────────────────────────────────────────┐
│ Stream 1: /api/users                    │
│ Stream 3: /api/posts   ← packet lost   │ ← triggers TCP HoL blocking
│ Stream 5: /static/app.js               │
│ Stream 7: /static/css                  │
└─────────────────────────────────────────┘
Impact: ALL 4 streams stall until the lost packet is retransmitted
```

**TCP retransmission timeline on mobile:**

```
t=0ms:  Server sends packet #1000 (belongs to Stream 3)
t=5ms:  Packet #1000 lost in transit (cell tower handoff)
t=5ms:  Streams 1, 5, 7 data arrives, but TCP buffer holds it —
        cannot deliver out-of-order data to HTTP/2
t=205ms: Retransmission timeout fires (RTO ≈ 200ms typical)
t=205ms: Packet #1000 retransmitted
t=210ms: Packet #1000 received — TCP can now deliver buffered data
t=210ms: All streams unblock simultaneously

Total stall: 205ms for a single dropped packet
```

**Why this is worse than HTTP/1.1 on mobile:**

HTTP/1.1's 6 parallel connections mean each stream is independent at the TCP layer. A dropped packet in connection 1 only delays that connection's stream — the other 5 connections keep flowing. HTTP/2's single-connection design concentrates all traffic, making any TCP-level loss a global stall.

**The more streams, the worse the problem:**

```
On a page with 50 resources on a 1% loss connection:
  HTTP/1.1: ~8–9 stall events, each affecting 1/6 of streams
  HTTP/2:   ~50 stall events (more streams = more packets = more chances),
            each affecting ALL streams
```

This is why HTTP/3 (QUIC) was designed — it moves to UDP and implements per-stream reliability, so a lost packet for Stream 3 only stalls Stream 3.

**Interview callout — when HTTP/2 is worse than HTTP/1.1:**
- High packet loss networks (mobile, satellite, long-distance WAN)
- Many small resources (more packets = more loss opportunities)
- Streams with vastly different sizes (one slow/lossy stream stalls all fast ones)

### HTTP/3 (QUIC)

Released 2022 (RFC 9114). Built on QUIC (UDP-based transport).

**Why QUIC over TCP:**
- **No TCP HoL blocking:** Each QUIC stream is independent. Packet loss in stream 1 doesn't block stream 3.
- **0-RTT connection resumption:** If you've connected before, first request goes in the same packet as the handshake.
- **Connection migration:** Connection identified by Connection ID (not IP:port), so mobile switching Wi-Fi → LTE keeps the connection alive.
- **TLS 1.3 built-in:** Encryption is mandatory and integrated, not layered on top.

**Connection setup comparison:**

| Protocol | New Connection RTTs | Resumed Connection RTTs |
|:---------|:--------------------|:------------------------|
| HTTP/1.1 + TLS 1.2 | 3 RTT (TCP + TLS) | 2 RTT |
| HTTP/2 + TLS 1.3 | 2 RTT (TCP + TLS 1.3) | 1–2 RTT |
| HTTP/3 (QUIC + TLS 1.3) | 1 RTT | 0 RTT |

{: .note }
**When does HTTP/2 > HTTP/3?** On low-latency, reliable networks (e.g., same datacenter), HTTP/2 is often faster — QUIC's UDP overhead isn't worth it. HTTP/3 shines on lossy/mobile connections.

---

## DNS Resolution Deep Dive

### The DNS Hierarchy

```
Root Nameservers (13 clusters worldwide)
    └── TLD Nameservers (.com, .org, .io)
            └── Authoritative Nameservers (your domain's NS records)
                    └── Your records (A, AAAA, CNAME, MX, TXT...)
```

### Resolution Flow

```mermaid
flowchart TD
    Browser --> OS[OS Resolver]
    OS --> Rec["Recursive Resolver\nISP / 8.8.8.8"]
    Rec -->|cache hit| Ret[Return A record + TTL]
    Rec -->|cache miss| Root["Root NS\n'Ask .com TLD NS'"]
    Root --> TLD[".com TLD NS\n'Ask ns1.example.com'"]
    TLD --> Auth["Authoritative NS\napi.example.com → 1.2.3.4"]
    Auth --> Ret
    Ret --> Browser2[Browser → 1.2.3.4 → HTTP request]
```

### DNS Record Types

| Record | Purpose | Example |
|:-------|:--------|:--------|
| **A** | Domain → IPv4 | `api.example.com → 1.2.3.4` |
| **AAAA** | Domain → IPv6 | `api.example.com → 2001:db8::1` |
| **CNAME** | Alias to another domain | `www → api.example.com` |
| **MX** | Mail server | `example.com → mail.example.com` |
| **NS** | Nameserver delegation | `example.com → ns1.example.com` |
| **TXT** | Arbitrary text | SPF, DKIM, domain verification |
| **SRV** | Service location (port + host) | Used by Kubernetes, gRPC |
| **PTR** | Reverse DNS (IP → domain) | Used by email servers |

### DNS Zones

A **DNS zone** is an administrative unit of authority within the DNS namespace. A zone is managed by one or more authoritative nameservers and contains all resource records for that portion of the hierarchy.

**Zone vs Domain:** A domain is a subtree of the DNS namespace (`example.com` and everything under it). A zone is the portion *managed by a specific set of nameservers*. Subdomains can be split off into separate zones, each delegated to a different team or provider.

```
example.com zone (managed by platform team):
  example.com          → A 1.2.3.4
  www.example.com      → A 1.2.3.4
  api.example.com      → A 1.2.3.5

dev.example.com zone (managed by dev team — separate zone):
  staging.dev.example.com → A 10.0.1.2
  prod.dev.example.com    → A 10.0.1.3
```

#### Zone File Structure

Every zone has a **SOA (Start of Authority)** record that carries metadata, followed by resource records.

```
$ORIGIN example.com.
$TTL 3600

; SOA — one per zone, defines zone metadata
@  IN SOA  ns1.example.com. admin.example.com. (
       2024050101  ; Serial (YYYYMMDDNN) — increment on every change
       3600        ; Refresh — secondary polls primary every 3600 s
       900         ; Retry — retry after 900 s if refresh fails
       604800      ; Expire — secondary stops answering after 7 days
       300         ; Negative TTL — NXDOMAIN cached for 5 min
   )

; Authoritative nameservers for this zone
@    IN NS   ns1.example.com.
@    IN NS   ns2.example.com.

; A / CNAME / MX records
@    IN A    1.2.3.4
www  IN A    1.2.3.4
api  IN A    1.2.3.5
cdn  IN CNAME example.cdn-provider.com.
@    IN MX 10 mail.example.com.

; Delegation — dev subdomain is a separate zone
dev  IN NS  ns1.dev.example.com.
dev  IN NS  ns2.dev.example.com.
```

#### Zone Types

| Type | Description |
|:-----|:------------|
| **Primary (master)** | Writable, holds the authoritative zone file |
| **Secondary (slave)** | Read-only replica synced from primary via zone transfer |
| **Stub** | Holds only NS records of a delegated child zone |
| **Forward** | Not authoritative — forwards queries upstream |

#### Zone Transfer (AXFR / IXFR)

Secondary nameservers replicate the zone from the primary:

- **AXFR** — full zone transfer: sends the entire zone file. Used for initial sync.
- **IXFR** — incremental transfer: sends only changes since the last known serial. Preferred for large zones.

```
Secondary polls primary at Refresh interval:
  Secondary → Primary: SOA query (current serial?)
  If primary serial > secondary serial:
    Secondary → Primary: IXFR (give me changes since serial N)
    Primary  → Secondary: changed/added/deleted records

Primary can also push proactively via DNS NOTIFY (RFC 1996)
```

{: .warning }
**Zone transfer security:** An unrestricted AXFR hands an attacker a complete map of every hostname in your zone. Always restrict with `allow-transfer { secondary-ip; }` or the equivalent in your DNS provider's ACL settings.

#### Zone Delegation

```mermaid
flowchart TD
    ROOT["Root Zone (.)"]
    COM[".com TLD Zone"]
    EX["example.com Zone\nns1.example.com · ns2.example.com"]
    DEV["dev.example.com Zone\nns1.dev.example.com\n(separate team, separate NS)"]
    API["api.example.com → 1.2.3.5\n(record inside example.com zone)"]

    ROOT -->|"NS delegation"| COM
    COM  -->|"NS delegation"| EX
    EX   -->|"NS delegation"| DEV
    EX   -->|"A record"| API
    DEV  -->|"A record"| STG["staging.dev.example.com → 10.0.1.2"]
```

The parent zone's only responsibility is to hold NS records pointing to the child zone's nameservers — it stores no records *from* the child zone.

#### Split-Horizon DNS

Return different IP addresses for the same hostname depending on the source of the query. This avoids hairpin NAT and keeps internal traffic on the internal network.

```
Query for api.example.com from 10.0.0.0/8 (internal):
  → 10.0.1.5  (private IP — direct path, no LB transit)

Query for api.example.com from external IP:
  → 1.2.3.5   (public IP — through CDN / load balancer)
```

```yaml
# BIND named.conf — two views for the same zone
view "internal" {
    match-clients { 10.0.0.0/8; };
    zone "example.com" {
        type primary;
        file "zones/internal/example.com";   # private IPs
    };
};

view "external" {
    match-clients { any; };
    zone "example.com" {
        type primary;
        file "zones/external/example.com";   # public IPs
    };
};
```

**Split-horizon use cases:** Internal service mesh (same hostname, private routing), geo-based routing (return nearest datacenter IP), staging vs production separation under one domain.

---

### DNS in System Design

**DNS-based load balancing:**
- Return multiple A records for same domain (round-robin)
- TTL controls how long clients cache the IP
- Low TTL (30–60s) = faster failover but more DNS load
- High TTL (300–3600s) = faster resolution but slower failover

**Anycast DNS (CDNs):** Same IP announced from multiple PoPs. BGP routes to nearest. Cloudflare's 1.1.1.1 uses Anycast.

**DNS and CDNs:** CDNs use DNS to direct users to the nearest edge node. `GeoDNS` returns different IPs based on client's resolver location.

{: .warning }
**Negative caching:** Failed DNS lookups are cached too (NXDOMAIN TTL). If you deploy a new service and DNS isn't ready, clients cache the failure for the NXDOMAIN TTL period.

---

## TLS/SSL Handshake

### TLS 1.3 Handshake (1-RTT)

TLS 1.3 simplified the handshake from TLS 1.2's 2-RTT to 1-RTT:

```
Client                                      Server
  |                                           |
  |--- ClientHello (supported TLS ver,    --->|
  |    cipher suites, client random,          |
  |    key_share with DH public key)          |
  |                                           |
  |<-- ServerHello (chosen cipher,        ----|
  |    server random, key_share),             |
  |    EncryptedExtensions,                   |
  |    Certificate, CertificateVerify,        |
  |    Finished                               |
  |                                           |
  |--- [Application Data] ------------------>|  ← Now encrypted!
  |--- Finished                          --->|
  |                                           |
```

**Key: DH (Diffie-Hellman) key exchange provides forward secrecy.** Even if the server's private key is later compromised, past sessions can't be decrypted.

### TLS 1.3 0-RTT (Session Resumption)

If client has a **pre-shared key (PSK)** from a previous session:
- Client sends application data immediately in the first flight
- No roundtrip needed for handshake
- **Risk:** 0-RTT data has no replay protection (acceptable for GET, not POST)

### Certificate Validation

```
Your cert → Intermediate CA cert → Root CA cert (trusted by OS/browser)
```

Browser checks:
1. Certificate chain valid (each cert signed by the next)
2. Cert not expired
3. Cert matches the hostname (SAN extension)
4. Cert not revoked (CRL or OCSP)

**OCSP Stapling:** Server attaches a fresh OCSP response to the TLS handshake. Client doesn't need to contact the CA. Reduces latency and CA load.

### mTLS (Mutual TLS)

Both client AND server present certificates. Used for:
- Service-to-service auth in microservices (Istio/Envoy sidecar)
- Zero Trust network architecture
- API authentication where strong identity is needed

---

## Real-Time Communication Patterns

### WebSockets

**Full-duplex** persistent connection over a single TCP connection. Upgraded from HTTP/1.1.

**Handshake:**
```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server
    C->>S: GET /ws HTTP/1.1\nUpgrade: websocket\nSec-WebSocket-Key: ...
    S->>C: 101 Switching Protocols\nUpgrade: websocket
    Note over C,S: Full-duplex channel open
    C->>S: Frame (client message)
    S->>C: Frame (server push)
    S->>C: Frame (server push)
```

**Use when:** Bidirectional, low-latency: chat, gaming, collaboration tools, live dashboards.

**Java WebSocket (Spring):**
```java
@ServerEndpoint("/chat")
public class ChatEndpoint {
    @OnMessage
    public void onMessage(String message, Session session) {
        // broadcast to all connected sessions
        session.getOpenSessions()
               .forEach(s -> s.getAsyncRemote().sendText(message));
    }
}
```

### Server-Sent Events (SSE)

**Server → Client only** (unidirectional), built on HTTP. Client opens a long-lived connection; server streams events.

```
GET /events HTTP/1.1
Accept: text/event-stream

HTTP/1.1 200 OK
Content-Type: text/event-stream

data: {"type": "order_update", "id": 123}\n\n
data: {"type": "price_change", "symbol": "AAPL"}\n\n
```

**Advantages:** Works with HTTP/2 multiplexing. Auto-reconnect built into browser EventSource API. Simple.

**Use when:** Notifications, live feeds, progress bars, stock tickers.

### Long Polling

Client sends a request; server holds it open until data is available (or timeout). Client immediately re-requests.

```
Client: GET /messages?timeout=30s
[30 seconds later]
Server: {"messages": [...]}  ← returns when message arrives or timeout
Client: GET /messages?timeout=30s  ← immediately reconnects
```

**Use when:** Simple to implement, works everywhere, acceptable latency (1–2s). Falls back well. Used by Slack, older chat systems.

### Comparison

| Feature | WebSocket | SSE | Long Polling |
|:--------|:----------|:----|:-------------|
| Direction | Bidirectional | Server→Client | Server→Client |
| Protocol | Upgraded HTTP | HTTP/1.1, HTTP/2 | HTTP |
| Auto-reconnect | Manual | Built-in | Manual |
| Overhead | Low (after handshake) | Low | High (per request) |
| HTTP/2 friendly | No (uses own framing) | Yes | Yes |
| Firewall friendly | Sometimes (port 80/443) | Always | Always |
| Best for | Chat, gaming, collab | Live feeds, notifications | Simple notifications |

---

## NAT and Load Balancing

### NAT (Network Address Translation)

#### Why NAT Exists

IPv4 uses 32-bit addresses — a theoretical maximum of ~4.3 billion unique IPs. By the early 1990s it was clear this would run out as the internet scaled. The solution was **RFC 1918 private address space**: three blocks reserved for internal networks that are never routed on the public internet.

| Private range | CIDR | Hosts |
|:--------------|:-----|:------|
| `10.0.0.0 – 10.255.255.255` | 10.0.0.0/8 | ~16.7 million |
| `172.16.0.0 – 172.31.255.255` | 172.16.0.0/12 | ~1 million |
| `192.168.0.0 – 192.168.255.255` | 192.168.0.0/16 | ~65,000 |

Every home router, corporate network, and cloud VPC uses private IPs internally. NAT is what lets millions of devices behind a single ISP each reach the public internet using just one (or a small pool of) public IPs.

```
Home network (192.168.1.0/24)          Internet
  Laptop  192.168.1.10  ─┐
  Phone   192.168.1.11  ─┤── NAT router (203.0.113.1) ──▶  example.com
  TV      192.168.1.12  ─┘        (one public IP)
```

**Without NAT:** each device would need a globally unique public IPv4, which ran out in 2011 (IANA exhaustion). NAT effectively multiplied the usable IPv4 space and delayed IPv6 adoption by decades.

**How NAT works (NAPT — Port Address Translation):**

1. Laptop (`192.168.1.10:54321`) sends a packet to `93.184.216.34:443`.
2. NAT router rewrites source to `203.0.113.1:41000` and stores the mapping in its connection table.
3. Response arrives at `203.0.113.1:41000`; router rewrites destination back to `192.168.1.10:54321` and forwards.

```
Connection table entry:
  External          ↔  Internal
  203.0.113.1:41000 ↔  192.168.1.10:54321  (TCP, expires on FIN/RST or timeout)
```

NAT allows multiple private IPs to share one public IP. The NAT device maintains a **connection table** mapping (private IP:port) → (public IP:port).

**Why it matters for system design:**
- Your server never sees the real client IP — use `X-Forwarded-For` header
- Connection limits: a single NAT box has limited port numbers (~65K per public IP)
- UDP traversal for P2P requires STUN/TURN (used by WebRTC)

### Layer 4 vs Layer 7 Load Balancing

**L4 Load Balancer (Transport Layer):**
- Works on IP + TCP/UDP without looking inside packets
- Faster (less processing), lower latency
- NAT-based or DSR (Direct Server Return)
- Can't route based on URL, headers, or cookies
- Example: AWS NLB, HAProxy (TCP mode)

**L7 Load Balancer (Application Layer):**
- Terminates TCP/TLS, inspects HTTP headers
- Can route by path, hostname, headers, cookies
- SSL termination — servers get plain HTTP
- Can do intelligent routing (A/B testing, canary, sticky sessions)
- Higher CPU cost than L4
- Example: AWS ALB, Nginx, HAProxy (HTTP mode), Envoy

**L4 vs L7 decision:**

| Scenario | Use L4 | Use L7 |
|:---------|:-------|:-------|
| Non-HTTP protocols (gRPC, MQTT) | ✅ | ❌ |
| Lowest latency | ✅ | ❌ |
| Path-based routing (/api vs /static) | ❌ | ✅ |
| SSL termination | ❌ | ✅ |
| Request-level metrics | ❌ | ✅ |
| Sticky sessions (cookie-based) | ❌ | ✅ |
| WebSocket with many short connections | ✅ | ❌ |

### Load Balancing Algorithms

| Algorithm | How | Best For |
|:----------|:----|:---------|
| Round Robin | Rotate through servers | Uniform request weight |
| Weighted Round Robin | Proportion to server capacity | Heterogeneous servers |
| Least Connections | Send to server with fewest active connections | Variable-duration requests |
| IP Hash | Hash client IP → server | Session affinity (stateful) |
| Consistent Hashing | Hash request key → server ring | Cache affinity, even distribution |
| Random | Pick randomly | Simple, surprisingly effective |

### Passthrough Mode vs Proxy Mode

These two modes describe **how** the load balancer handles the TCP connection between client and backend — a decision that determines TLS ownership, client IP visibility, and performance characteristics.

#### Proxy Mode (Full Proxy)

The load balancer **terminates** the client TCP connection and opens a separate TCP connection to the backend. It is a man-in-the-middle at the transport level.

```
Client ──TCP conn 1──▶ [Load Balancer] ──TCP conn 2──▶ Backend
         (closed here)   terminates TLS     (new TCP)
                         reads HTTP headers
                         rewrites Host, adds X-Forwarded-For
```

- Used by: AWS ALB, Nginx (HTTP mode), HAProxy (HTTP mode), Envoy
- The LB can decrypt TLS, inspect HTTP headers/cookies/body, inject headers, and re-encrypt (or forward plain HTTP to the backend)
- Backend sees the LB's IP as the source — the real client IP is passed via `X-Forwarded-For` / `X-Real-IP` header
- Enables: SSL offloading, path-based routing, A/B testing, WAF integration, session persistence by cookie

#### Passthrough Mode (TCP Passthrough)

The load balancer **forwards raw TCP segments** without terminating the connection. It acts purely at L4, routing by IP + port. The TCP handshake goes end-to-end between client and backend.

```
Client ──────────────────────────────────────────────▶ Backend
         TCP handshake passes through the LB unchanged
         TLS terminated at backend (end-to-end encryption)
         LB reads only IP header and TCP port
```

- Used by: AWS NLB (TCP mode), HAProxy (TCP mode), Nginx (stream module)
- Backend receives the real client IP (no `X-Forwarded-For` needed; use Proxy Protocol for it)
- TLS is terminated at the backend — the LB never sees the decrypted payload
- Faster: fewer CPU cycles (no TLS, no HTTP parsing), lower latency
- **TLS passthrough use case:** regulatory requirement that the LB must not decrypt traffic (PCI-DSS card data, healthcare PHI) — backend holds the private key

#### Direct Server Return (DSR)

A specialized passthrough variant where the **response bypasses the load balancer entirely**. The LB rewrites only the destination MAC address; the backend sends its reply directly to the client using the LB's VIP as the source IP.

```
Client ──▶ [Load Balancer] ──▶ Backend
                                  │
Client ◀───────────────────────────┘  (response bypasses LB)
```

- Used by: LVS (Linux Virtual Server), HAProxy with DSR, hardware appliances (F5)
- Eliminates the LB as a bottleneck on the response path — critical when response >> request (video streaming, large file downloads)
- Requires the backend's loopback interface to be configured with the VIP so it can use it as source IP

#### Comparison

| Dimension | Proxy Mode | Passthrough Mode | DSR |
|:----------|:-----------|:----------------|:----|
| TLS termination | At LB (offloaded) | At backend (end-to-end) | At backend |
| OSI layer | L7 (Application) | L4 (Transport) | L4 |
| Client IP visibility | Via `X-Forwarded-For` header | Proxy Protocol or real IP | Real IP |
| HTTP routing (path, header) | ✅ Yes | ❌ No | ❌ No |
| CPU overhead | Higher (TLS + HTTP parse) | Lower | Lowest |
| Response path | Through LB | Through LB | Direct to client |
| AWS equivalent | ALB | NLB | Not natively available |

**Decision rule:**
- Use **proxy mode** when you need SSL termination, URL routing, header inspection, or WAF.
- Use **passthrough** when the backend must own TLS (compliance, mTLS), or when you need lowest latency for non-HTTP protocols (MQTT, gRPC with client certs, raw TCP).
- Use **DSR** when response bandwidth dwarfs request bandwidth and the LB would otherwise be a throughput bottleneck.

---

## Unicast, Multicast, and Broadcast

Every packet sent on a network has an addressing mode — it determines how many destinations receive the traffic and how the network delivers it.

### The Four Addressing Modes

| Mode | Sender | Receivers | IP range / address | Scale |
|:-----|:-------|:----------|:-------------------|:------|
| **Unicast** | One | One specific host | Any regular IPv4/IPv6 address | 1 copy per receiver |
| **Broadcast** | One | All hosts on a subnet | `255.255.255.255` or `<subnet>.255` | 1 copy → all on segment |
| **Multicast** | One | Subscribed group members | `224.0.0.0/4` (IPv4 Class D), `ff00::/8` (IPv6) | 1 copy per network segment |
| **Anycast** | One | Nearest node advertising the address | Regular IP, multi-site BGP announcement | Routed to geographically closest |

### Unicast

Standard TCP/IP communication — every HTTP request, gRPC call, and database connection is unicast. One source IP, one destination IP.

**Scale implication:** Sending to N receivers requires N independent copies of the data. A video stream delivered to 10,000 viewers via unicast means 10,000 separate TCP streams from the origin. CDNs solve this by terminating unicast streams at edge nodes close to viewers.

### Broadcast

One sender reaches every host on the same Layer 2 segment (broadcast domain). Routers do not forward broadcast traffic between subnets — it stays contained within the local segment.

**Layer 2 broadcast:** Ethernet sends to `FF:FF:FF:FF:FF:FF` — every NIC on the LAN processes the frame.

**Layer 3 broadcast:** IPv4 sends to `255.255.255.255` (limited) or `192.168.1.255` (directed, subnet-scoped).

**IPv6 removes broadcast entirely** — multicast replaces every use case.

**Protocols that rely on broadcast:**

```
ARP (Address Resolution Protocol):
  "Who has IP 192.168.1.1? Tell 192.168.1.10"
  → Sent to FF:FF:FF:FF:FF:FF (all devices on LAN respond if they own that IP)

DHCP Discovery:
  Client has no IP → sends to 255.255.255.255 → all DHCP servers hear it
  First server response wins; client sends unicast to accept

NetBIOS Name Resolution (legacy Windows):
  Broadcast to find hostname → replaced by DNS in modern networks
```

**Broadcast storms:** A bridging loop causes broadcast frames to circulate forever, each switch forwarding to all ports. Spanning Tree Protocol (STP) breaks loops by blocking redundant paths.

### Multicast

One sender delivers to a **group** of interested receivers. Multicast-aware routers replicate the packet only at branch points in the delivery tree — one copy traverses each network link, regardless of how many receivers are downstream.

```
                         [Sender]
                             │ (one stream)
                         [Router A]
                        /          \
                   [Router B]    [Router C]     ← one copy per link
                  /       \           \
            [Recv 1]   [Recv 2]    [Recv 3]
```

**Multicast group management:**
- **IGMP (IPv4):** Hosts join a group by sending an IGMP `Membership Report` to the router. Leave by sending a `Leave Group` message.
- **MLD (IPv6):** Equivalent protocol for IPv6 multicast.
- Routers build a multicast distribution tree using PIM (Protocol Independent Multicast).

**Reserved multicast ranges:**

| Range | Scope | Examples |
|:------|:------|:---------|
| `224.0.0.0 – 224.0.0.255` | Link-local (not routed) | `224.0.0.1` all-hosts, `224.0.0.2` all-routers, OSPF hello |
| `224.0.1.0 – 238.255.255.255` | Globally routable | Registered multicast applications |
| `239.0.0.0/8` | Administratively scoped (private) | Enterprise-internal multicast |

**System design uses of multicast:**

```
Financial market data feeds:
  NYSE, NASDAQ distribute real-time price feeds via multicast UDP.
  One feed → thousands of trading floor subscribers.
  Unicast alternative would require the exchange to maintain thousands of TCP connections.

Cluster heartbeat protocols:
  Corosync (Linux HA cluster) sends heartbeats to 239.x.x.x multicast group.
  All cluster nodes subscribe; only one heartbeat packet per interval regardless of cluster size.

mDNS / Zeroconf (service discovery on local networks):
  Apple Bonjour, Kubernetes with mDNS resolvers use 224.0.0.251:5353.
  "Who is printer.local?" → multicast query → printer responds with unicast.

SSDP (Simple Service Discovery Protocol):
  UPnP devices announce themselves to 239.255.255.250.
  Smart TVs, routers, and IoT devices use this for auto-discovery.
```

**Multicast in Java (UDP multicast socket):**

```java
// Sender: publish a heartbeat to a multicast group
try (MulticastSocket socket = new MulticastSocket()) {
    InetAddress group = InetAddress.getByName("239.1.2.3");
    byte[] data = "heartbeat:node-1".getBytes(StandardCharsets.UTF_8);
    DatagramPacket packet = new DatagramPacket(data, data.length, group, 5000);
    socket.send(packet);
}

// Receiver: subscribe to the multicast group
try (MulticastSocket socket = new MulticastSocket(5000)) {
    InetAddress group = InetAddress.getByName("239.1.2.3");
    socket.joinGroup(group);   // sends IGMP Membership Report to router

    byte[] buffer = new byte[256];
    DatagramPacket packet = new DatagramPacket(buffer, buffer.length);
    socket.receive(packet);    // blocks until a multicast packet arrives
    System.out.println(new String(packet.getData(), 0, packet.getLength()));

    socket.leaveGroup(group);  // sends IGMP Leave Group
}
```

### Anycast

The same IP address is announced from multiple geographic locations via BGP. Each router on the internet directs packets to the *topologically nearest* announcement. Anycast is not a separate IP range — it is a routing strategy applied to ordinary unicast addresses.

```
Cloudflare's 1.1.1.1 is announced from 300+ PoPs worldwide.
A user in Tokyo resolves 1.1.1.1 → BGP routes to Tokyo PoP.
A user in London resolves 1.1.1.1 → BGP routes to London PoP.
Same destination IP, completely different physical servers.
```

**Use cases:**

| Service | Why anycast |
|:--------|:-----------|
| DNS (1.1.1.1, 8.8.8.8) | Nearest resolver = lowest query latency |
| CDN edge nodes | Nearest PoP serves cached content |
| DDoS mitigation | Attack traffic absorbed by closest scrubbing center |
| NTP (time.cloudflare.com) | Nearest time server = lowest sync latency |

**Anycast vs Unicast load balancing:** Unicast load balancing is explicit (send to LB VIP, LB routes to backend). Anycast load balancing is implicit (BGP routes to nearest PoP, no LB required for geographic distribution).

### Addressing Mode Summary for System Design

| Decision | Guidance |
|:---------|:---------|
| API calls, database connections | Always unicast |
| Service discovery on a local subnet | Multicast (mDNS) or unicast with a registry |
| Real-time market data to many consumers | Multicast UDP (one sender, N receivers, efficient) |
| CDN / global DNS / DDoS scrubbing | Anycast via BGP |
| DHCP, ARP | Broadcast (protocol-mandated; no alternative) |
| Video to millions of viewers | Unicast via CDN edge (public internet doesn't route multicast) |

{: .note }
**Multicast on the public internet:** ISPs generally do not support multicast routing between ASes (Autonomous Systems). Multicast is reliable within a single datacenter or enterprise network. Internet-scale "multicast" (live video, CDN) is implemented as unicast to CDN edge nodes, which then unicast to end users.

---

## Key Takeaways for Interviews

1. **TCP trade-off:** Reliability costs latency. When designing real-time systems, UDP (or QUIC) may be better.
2. **HTTP/2 doesn't eliminate TCP HoL blocking** — HTTP/3/QUIC does.
3. **DNS TTL is your failover speed** — tune it before a planned migration.
4. **TLS 1.3 = 1-RTT** — essential for global services. Always enable OCSP stapling.
5. **WebSocket** for bidirectional; **SSE** for server-push over HTTP/2; **Long Polling** when WebSocket isn't available.
6. **L7 LB** for HTTP routing; **L4 LB** for non-HTTP or when latency is critical.
7. **Multicast is efficient for many-receivers, same-data** — but only within a datacenter or enterprise. Public internet video "multicast" is actually CDN unicast at scale.
8. **Anycast is geographic load balancing via BGP** — no application-level routing required. DNS resolvers and CDN PoPs use it to route to the nearest server transparently.

---

## References

- [Cloudflare: How HTTP/3 works](https://blog.cloudflare.com/http3-the-past-present-and-future/)
- [High Performance Browser Networking](https://hpbn.co/) — Ilya Grigorik (free online)
- [RFC 9114](https://www.rfc-editor.org/rfc/rfc9114) — HTTP/3
- [RFC 1112](https://www.rfc-editor.org/rfc/rfc1112) — IGMP: Host Extensions for IP Multicasting
- *Computer Networks* — Andrew Tanenbaum (Chapter 6–7)
