
- **Status line**: Version, status code, reason phrase
- **Headers**
- **Empty line**
- **Body**

## 3. HTTP Methods (RFC 9110)

| Method   | Description                          | Safe | Idempotent | Body | Common Use          |
|----------|--------------------------------------|------|------------|------|---------------------|
| GET      | Retrieve resource                    | Yes  | Yes        | No   | Fetch page          |
| HEAD     | Same as GET, no body                 | Yes  | Yes        | No   | Metadata check      |
| POST     | Submit data                          | No   | No         | Yes  | Form submit, API    |
| PUT      | Replace resource                     | No   | Yes        | Yes  | Update/replace      |
| DELETE   | Delete resource                      | No   | Yes        | Opt  | Remove item         |
| PATCH    | Partial update                       | No   | No         | Yes  | Modify resource     |
| OPTIONS  | List supported methods               | Yes  | Yes        | No   | CORS preflight      |
| TRACE    | Echo request (diagnostic)            | Yes  | Yes        | No   | Debugging           |
| CONNECT  | Establish tunnel (HTTPS proxy)       | No   | No         | No   | HTTPS over proxy    |

## 4. Status Code Classes

| Class | Meaning                  | Examples                     |
|-------|--------------------------|------------------------------|
| 1xx   | Informational            | 100 Continue, 101 Switching Protocols |
| 2xx   | Success                  | 200 OK, 201 Created, 204 No Content |
| 3xx   | Redirection              | 301 Moved Permanently, 302 Found, 304 Not Modified |
| 4xx   | Client Error             | 400 Bad Request, 401 Unauthorized, 404 Not Found |
| 5xx   | Server Error             | 500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable |

## 5. Key Headers

### General
- `Host` (mandatory in HTTP/1.1)
- `Content-Length` / `Transfer-Encoding: chunked`
- `Connection: keep-alive` / `close`

### Request
- `User-Agent`
- `Accept`
- `Authorization: Bearer <token>`
- `Cookie`
- `Referer` / `Referrer-Policy`

### Response
- `Server`
- `Set-Cookie`
- `Location` (redirects)
- `Cache-Control`
- `ETag` / `Last-Modified`

## 6. HTTP Versions

| Version   | Year | Key Features                              | Transport     |
|-----------|------|-------------------------------------------|---------------|
| HTTP/0.9  | 1991 | One-line GET, HTML only                   | TCP           |
| HTTP/1.0  | 1996 | Headers, status codes, methods            | TCP           |
| HTTP/1.1  | 1997 | Persistent connections, chunked, Host req | TCP           |
| HTTP/2    | 2015 | Binary, multiplexing, header compression, server push | TCP + TLS |
| HTTP/3    | 2022 | QUIC (UDP), 0-RTT, better loss recovery   | QUIC (UDP)    |

## 7. Connection Management

- **HTTP/1.0**: One request per TCP connection (default `Connection: close`).
- **HTTP/1.1**: Persistent connections (`Connection: keep-alive`), pipeline (deprecated).
- **HTTP/2**: Single TCP connection, multiple streams (multiplexing), flow control.
- **HTTP/3**: QUIC over UDP → eliminates head-of-line blocking, faster connection setup.

TCP handshake (3-way) required before HTTP data:
1. SYN
2. SYN-ACK
3. ACK

## 8. HTTP and the OSI 7-Layer Model

HTTP lives at **Layer 7**. Data flows downward on send, upward on receive.

| OSI Layer | Name            | PDU       | Protocols Involved in HTTP Flow                          | Role in HTTP Transaction                                                                 |
|-----------|-----------------|-----------|----------------------------------------------------------|------------------------------------------------------------------------------------------|
| 7         | Application     | Data      | HTTP, HTTPS                                              | Constructs/parses HTTP messages (methods, headers, body).                                |
| 6         | Presentation    | Data      | TLS (for HTTPS), gzip, charset                           | Encryption (TLS), compression (`Content-Encoding: gzip`), character encoding.            |
| 5         | Session         | Data      | TCP (session management), cookies, JWT                   | Manages logical session (persistent TCP connections, cookies for state).                 |
| 4         | Transport       | Segment   | TCP (HTTP/1.x, HTTP/2), QUIC/UDP (HTTP/3)                | Reliable delivery, port 80/443, congestion control, multiplexing (HTTP/2+).              |
| 3         | Network         | Packet    | IP (IPv4/IPv6), ICMP                                     | Routing, IP addressing, fragmentation.                                                   |
| 2         | Data Link       | Frame     | Ethernet, Wi-Fi, PPP, MPLS                               | MAC addressing, error detection (CRC), framing for local network.                        |
| 1         | Physical        | Bits      | Cables, fiber, radio signals, NIC hardware               | Actual bit transmission over medium (copper, fiber, wireless).                           |

**Encapsulation path (client → server)**:
HTTP message → TLS (if HTTPS) → TCP segment → IP packet → Ethernet frame → bits on wire.

**Decapsulation (server → client)**: reverse order.

**Example full stack for `https://example.com/api/users`**:
- Layer 7: `GET /api/users HTTP/2`
- Layer 6: TLS 1.3 handshake + encrypted payload
- Layer 5: TCP connection (port 443) + session cookies
- Layer 4: TCP segments (port 443 → ephemeral)
- Layer 3: IPv6 or IPv4 packets
- Layer 2: Ethernet frames (MAC src/dst)
- Layer 1: Physical transmission (e.g., Cat6 cable or 5G)

## 9. Common Implementations & Tools

- Clients: curl, browsers (Chrome, Firefox), Postman
- Servers: nginx, Apache, Caddy, Node.js http, Go net/http
- Proxies/CDNs: Cloudflare, AWS CloudFront (terminate TLS, cache)
- Debugging: Wireshark (capture all layers), tcpdump, mitmproxy

## 10. Security Considerations (HTTPS)

- Always use TLS 1.2+ (Layer 6/4)
- HSTS, CSP, CORS policies at Layer 7
- HTTP/2+ requires TLS (except h2c)

## References (RFCs)

- RFC 9110 – HTTP Semantics
- RFC 9112 – HTTP/1.1
- RFC 9113 – HTTP/2
- RFC 9114 – HTTP/3
- RFC 7230–7235 (legacy HTTP/1.1)

This document is self-contained for quick reference or inclusion in any README.md.
