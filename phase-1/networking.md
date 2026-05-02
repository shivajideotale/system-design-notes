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

## TCP vs UDP

### TCP (Transmission Control Protocol)

**What:** Connection-oriented, reliable, ordered byte stream.

**How it works — 3-Way Handshake:**
```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server
    C->>S: SYN
    S->>C: SYN-ACK
    C->>S: ACK
    Note over C,S: Connection established — data flows
```

**Connection Teardown (4-Way):**
```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server
    C->>S: FIN (client done sending)
    S->>C: ACK
    S->>C: FIN (server done sending)
    C->>S: ACK
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

---

## Key Takeaways for Interviews

1. **TCP trade-off:** Reliability costs latency. When designing real-time systems, UDP (or QUIC) may be better.
2. **HTTP/2 doesn't eliminate TCP HoL blocking** — HTTP/3/QUIC does.
3. **DNS TTL is your failover speed** — tune it before a planned migration.
4. **TLS 1.3 = 1-RTT** — essential for global services. Always enable OCSP stapling.
5. **WebSocket** for bidirectional; **SSE** for server-push over HTTP/2; **Long Polling** when WebSocket isn't available.
6. **L7 LB** for HTTP routing; **L4 LB** for non-HTTP or when latency is critical.

---

## References

- [Cloudflare: How HTTP/3 works](https://blog.cloudflare.com/http3-the-past-present-and-future/)
- [High Performance Browser Networking](https://hpbn.co/) — Ilya Grigorik (free online)
- [RFC 9114](https://www.rfc-editor.org/rfc/rfc9114) — HTTP/3
- *Computer Networks* — Andrew Tanenbaum (Chapter 6–7)
