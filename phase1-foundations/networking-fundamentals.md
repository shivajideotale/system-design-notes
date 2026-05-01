# Networking Fundamentals

This document covers essential networking concepts for system design, focusing on protocols, security, and performance considerations.

## TCP (Transmission Control Protocol)

TCP is a connection-oriented protocol that provides reliable, ordered, and error-checked delivery of data.

### Key Features
- **Connection Establishment**: Three-way handshake (SYN, SYN-ACK, ACK)
- **Reliability**: Acknowledgment, retransmission, sequencing
- **Flow Control**: Sliding window protocol
- **Congestion Control**: Slow start, congestion avoidance, fast retransmit

### System Design Implications
- Used for applications requiring guaranteed delivery (HTTP, FTP, SMTP)
- Overhead from connection setup and teardown
- Head-of-line blocking in early HTTP versions

### Code Example (Java)
```java
import java.net.Socket;

Socket socket = new Socket("example.com", 80);
// Connection established
```

## UDP (User Datagram Protocol)

UDP is a connectionless protocol that provides fast, unreliable delivery.

### Key Features
- No connection setup/teardown
- No reliability guarantees
- Lower latency than TCP
- Multicast support

### System Design Implications
- Ideal for real-time applications (VoIP, gaming, streaming)
- Used in DNS queries, DHCP, SNMP
- Requires application-level reliability if needed

### Code Example (Java)
```java
import java.net.DatagramSocket;

DatagramSocket socket = new DatagramSocket();
// Send datagram without connection
```

## HTTP/2 and HTTP/3

### HTTP/2
- Multiplexing: Multiple requests over single TCP connection
- Header compression (HPACK)
- Server push
- Binary protocol

### HTTP/3
- Based on QUIC protocol over UDP
- Improved performance, especially on poor networks
- Built-in encryption (TLS 1.3)
- Connection migration

### System Design Implications
- Reduces latency for web applications
- Enables efficient API communication
- Considerations for load balancing and proxying

## DNS (Domain Name System)

DNS translates domain names to IP addresses.

### Components
- **Recursive Resolver**: Client-side resolver
- **Authoritative Servers**: Hold zone data
- **Root Servers**: Top-level domain delegation

### Types of Records
- A: IPv4 address
- AAAA: IPv6 address
- CNAME: Canonical name
- MX: Mail exchange

### System Design Implications
- DNS caching for performance
- CDN integration
- Failover and load balancing
- Security: DNSSEC for authentication

## TLS (Transport Layer Security)

TLS provides encryption and authentication for network communications.

### Handshake Process
1. Client Hello
2. Server Hello + Certificate
3. Key Exchange
4. Finished messages

### Key Concepts
- **Certificates**: X.509 certificates for identity verification
- **Cipher Suites**: Algorithms for encryption, key exchange, MAC
- **Perfect Forward Secrecy**: Ephemeral keys prevent past session decryption

### System Design Implications
- Essential for secure APIs and web services
- Performance overhead (CPU for encryption)
- Certificate management and rotation
- TLS termination at load balancers

## WebSockets

WebSockets provide full-duplex communication over a single TCP connection.

### Protocol
- Starts with HTTP upgrade request
- Persistent connection after handshake
- Binary or text frames

### Use Cases
- Real-time applications (chat, notifications, live updates)
- Gaming, financial trading platforms
- IoT device communication

### System Design Implications
- Scalability challenges (connection state)
- Load balancing with sticky sessions
- Security considerations (origin checking, authentication)

## Key Takeaways

- Choose TCP for reliability, UDP for speed
- HTTP/2+ significantly improves web performance
- DNS design affects global application reach
- TLS is non-negotiable for production systems
- WebSockets enable real-time features but complicate scaling

## Resources

- [RFC 793: TCP Specification](https://tools.ietf.org/html/rfc793)
- [HTTP/2 Specification](https://httpwg.org/specs/rfc9113.html)
- [WebSocket Protocol](https://tools.ietf.org/html/rfc6455)
- [Cloudflare DNS Learning Center](https://www.cloudflare.com/learning/dns/)