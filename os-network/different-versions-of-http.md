# HTTP/1, HTTP/1.1, and HTTP/2 Overview

## Introduction
HTTP (Hypertext Transfer Protocol) is the foundation of data communication on the World Wide Web. This document provides a comparative overview of HTTP/1, HTTP/1.1, and HTTP/2, highlighting key features, differences, and evolution.

## HTTP/1.0 (1996)
**First standardized version**

### Key Features
- **Connection per request**: Each request-response pair requires a separate TCP connection
- **Stateless protocol**: No persistent connection between requests
- **Basic methods**: GET, HEAD, POST
- **Status codes**: Introduced common status codes (200, 404, etc.)
- **Headers**: Basic request/response headers
- **Content negotiation**: Limited support via headers

### Limitations
- **High latency**: New TCP connection for each resource
- **No host headers**: Couldn't host multiple domains on same IP
- **No compression**: Headers sent uncompressed
- **No caching directives**: Limited caching capabilities

## HTTP/1.1 (1997, updated 1999)
**Major improvement still widely used today**

### Key Features
- **Persistent connections**: Multiple requests/responses per TCP connection (keep-alive)
- **Pipelining**: Multiple requests without waiting for responses (theoretically)
- **Chunked transfer encoding**: Stream responses without knowing content length upfront
- **Host header**: Virtual hosting support (multiple domains on single server)
- **Cache control**: Extensive caching headers (ETag, Cache-Control)
- **Byte ranges**: Partial content requests (Range header)
- **New methods**: PUT, DELETE, OPTIONS, TRACE
- **Content negotiation**: Improved with Accept headers

### Limitations
- **Head-of-line blocking**: Even with pipelining, responses must be in order
- **No header compression**: Headers sent repeatedly
- **Limited parallelism**: Browser workarounds (6-8 parallel connections per domain)
- **Complex parsing**: Text-based protocol with various edge cases

## HTTP/2 (2015)
**Major protocol overhaul focusing on performance**

### Key Features
- **Binary protocol**: Not human-readable but more efficient to parse
- **Multiplexing**: Multiple streams over single TCP connection
- **Header compression**: HPACK compression for headers
- **Stream prioritization**: Client can prioritize important resources
- **Server push**: Server can send resources before client requests them
- **Flow control**: Similar to TCP flow control but per stream
- **Request/response multiplexing**: Eliminates head-of-line blocking

### Technical Details
- **Frames**: Basic protocol unit (HEADERS, DATA, SETTINGS, etc.)
- **Streams**: Logical channels within connection
- **HPACK**: Header compression algorithm reducing redundancy
- **ALPN**: Application-Layer Protocol Negotiation for HTTPS upgrade

## Comparison Table

| Feature | HTTP/1.0 | HTTP/1.1 | HTTP/2 |
|---------|----------|----------|---------|
| **Connection** | One per request | Persistent | Single, multiplexed |
| **Protocol** | Text | Text | Binary frames |
| **Header format** | Uncompressed | Uncompressed | HPACK compressed |
| **Multiplexing** | No | Limited (pipelining) | Yes (true multiplexing) |
| **Server push** | No | No | Yes |
| **Priority** | No | No | Yes (stream prioritization) |
| **Head-of-line blocking** | Yes | Yes (request level) | No (stream level) |
| **Security** | Optional HTTPS | Optional HTTPS | Often requires HTTPS |

## Performance Implications

### HTTP/1.1 Issues
- **Domain sharding**: Workaround for connection limits
- **Resource concatenation**: Combining CSS/JS files
- **Image sprites**: Combining multiple images
- **Inefficient header transmission**: Same headers sent repeatedly

### HTTP/2 Improvements
- **Reduced latency**: Multiplexing eliminates head-of-line blocking
- **Efficient header transmission**: HPACK significantly reduces overhead
- **Better resource prioritization**: Important assets load first
- **Single connection**: Reduced TLS negotiation overhead

## Migration Considerations

### From HTTP/1.1 to HTTP/2
- **HTTPS required**: Most implementations require TLS
- **Optimization changes**: Some HTTP/1.1 optimizations are anti-patterns in HTTP/2
  - Domain sharding can hurt performance
  - Resource concatenation less beneficial
- **Server support**: Requires HTTP/2 compatible server (nginx, Apache, etc.)
- **Backward compatibility**: HTTP/2 includes fallback mechanisms

### Anti-patterns in HTTP/2
- Domain sharding (increases connection overhead)
- Excessive resource concatenation
- Image sprites (when HTTP/2 push could be used instead)

## Browser and Server Support

### HTTP/2 Support
- **Browsers**: All modern browsers (Chrome, Firefox, Safari, Edge)
- **Servers**: Nginx (1.9.5+), Apache (2.4.17+), IIS (Windows 10/Server 2016)
- **CDNs**: All major CDNs support HTTP/2

## Tools and Testing

### Check HTTP Version
```bash
# Using curl
curl -I --http2 https://example.com

# Using browser developer tools
# Network tab shows protocol column
```

### Testing Tools
- **h2load**: HTTP/2 benchmarking tool
- **nghttp2**: HTTP/2 client/server tools
- **Chrome DevTools**: Protocol information in Network tab
- **WebPageTest**: Includes HTTP/2 testing capabilities

## Security Considerations

### HTTP/2 Security
- **Mandatory TLS**: For most browser implementations
- **Reduced attack surface**: Binary protocol avoids some parsing attacks
- **Same security model**: No new security features, just transport improvements


