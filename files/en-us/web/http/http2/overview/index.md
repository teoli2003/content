---
title: An HTTP/2 overview
slug: Web/HTTP/HTTP2/Overview
tags:
 - Guide
---
{{HTTPSidebar}}

HTTP/2 is an evolution of the HTTP/1.1 protocol designed to remove some of its predecessor's performance and security problems.

> **Note:** As HTTP/2 wants to improve the protocol's security, it doesn't work on non-{{Glossary("TLS")}} connections. It only works when using the `https:` protocol and not with `http:`.

As billions of web applications were using HTTP/1.1, it wasn't realistic to request each website to be adapted. Not only were they too numerous, but many of them are not maintained, and nobody will update them. HTTP/2 (and its successor, HTTP/3) are in-place replacements for HTTP/1.1, meaning that once a server and a client both support it,
they can immediately and seamlessly start using it. There is no need to adapt the web pages themselves, ensuring backward compatibility at the application layer level.

[schema of layers for HTTP/1 and HTTP/2-3, with "transport layer" instead of TCP]

## Problems of HTTP/1.1

Among the most common causes of performance limitation in HTTP/1.1 are:

- The absence of parallel requests. Each connection between a client and a server can handle only one request/response at a time. These necessary roundtrips gate the maximum throughput on the connection. Web developers worked around with ad-hoc techniques like {{Glossary("domain sharding")}}, but these impair optimizations at the transport layer (like TCP windowing). Furthermore, the cost of creating new connections is high (especially in HTTPS, where the TLS handshake requires the exchange of several messages). Over the years, engineers attempted several times to solve this problem at the session level, with standards like _HTTP pipelining_. That never worked.
- {{Glossary("Head-of-line blocking")}}. Even with domain sharding, if all connections are busy waiting for earlier transactions to complete, subsequent requests, which may have more importance, will be blocked. HTTP/1.1 is particularly subject to this because the client must receive the different packets of a message in the correct order and discards packets received too early, even if they are pertinent.
- Repetition of data sent. Headers in successive or parallel HTTP requests are often identical. Sending them each time increases the size of the request, increases overhead, and lowers the bandwidth available to data. Headers are not necessarily small: cookies, sent in an HTTP header, can be as large as 500, or even 1000, bytes.

Also, HTTP/1.1 suffered from security and privacy caveats:

- When using the `http:` protocol, the URL, cookies, as well as the data sent back are sent in cleartext, fully readable and temparable by an attacker.
- Even when using HTTP over TLS, the `https:` protocol, attacks like [CRIME](https://en.wikipedia.org/wiki/CRIME) will compromise the security of the communication, by stealing authentication cookies for example.

HTTP/2 was designed to solve and mitigate these issues.

## HTTP streams and frames

Designed to be an in-place replacement, an [HTTP message](Messages) in HTTP/2 is identical to a message in HTTP/1.1. They contain the same information and, that way, higher-level APIs don't need to be adapted.

![](httpmsg2.png)

HTTP/1.1 transmits an HTTP message as some human-readable text, while HTTP/2 transforms it into several _frames_, in a process called [binary framing](Binary_framing_in_HTTP).

An _HTTP frame_ is a binary structure that contains a part of an HTTP message. As it is clear whom HTTP message a given frame belongs to, frames bring much more flexibility than the text messages transmitted previously.

HTTP/2 includes multiplexing at its heart. A _stream_ is a succession of _frames_ that, once combined, represents an HTTP request and its response. Each frame belongs to a given stream and is aware of its relative position in it. Streams are bidirectional: a request and its response belong to the same stream.  **TODO: Stream reuse?**

The different frames don't need to arrive in the correct order. The loss of a packet, leading to a missing frame, doesn't call forth the discard of all the already-sent subsequent parts. Unlike HTTP/1.1, once the lost frame is received, the destination can reconstitute the message if it doesn't receive the packets in the proper order.

Each connection can handle several concurrent and intertwined streams, implementing multiplexing. Thus, a connection consists of a succession of frames belonging to different streams.

![](https://developers.google.com/web/fundamentals/performance/http2/images/multiplexing01.svg)

**TODO: recreate image rather than external link**

## Reducing overhead with header compression

If developers always perceived the data included in an HTTP message as significant, headers are not small either. Every request with a `Cookie` header will be well over 1kB in size on most websites.

At the application layer, in HTTP/1.1, most of the time, peers compress the content of a request or a response, using gzip ou brotli, whilst the headers are not. When using TLS, headers get compressed, but this technique proved dangerous from a security perspective by allowing CRIME-like attacks.

Also, servers answer many requests with a status of `304 Not Changed`: Many responses are much smaller than their requests, triggering unintended effects in the TCP layer and additional latency.

To mitigate this, HTTP/2 packs and compresses headers using HPACK. Engineers created an algorithm specially designed to compress HTTP headers, leading to better compression, and made it resilient to CRIME-like attacks. Also, repeated headers are only transmitted once.

**TODO: image of repeated headers33

[More information: _HTTP header compression_ article]

## Prioritization

Downloading resources in a specific order is paramount to a good user experience. Users don't like to wait, and some resources are critical for them to start using the web document, while others are less important.

Web developers can provide information directly inside the HTML content, like indicating if a script is urgent or can wait until more critical data (with the `defer` and `` attributes), but browsers had no way to coordinate with the server to make good use of the available bandwidth.

**TODO: Insert the image of an example of prioritization and connection sharing**

HTTP/2 provides a mechanism to prioritize the resources to download by giving three information to each stream:

the ID of another stream it depends on if any;
a number indicating its relative priority;
a flag indicating if this stream should use the whole available bandwidth or share the connection.
These three values represent a dependency tree between the resources.

Browsers determine such values for each stream using different algorithms. Conformant servers then send data according to the priority given.

> **Note:**
>
> - If the server or the client doesn't support this feature, all streams are equally prioritized and fairly share the connection.
> - Some servers, like [Cloudfare](), don't blindly obey the client-provided prioritization, though.

## Push notifications

> **Note:**  In practice, adoption for this feature has been extremely low due to its relative complexity and its advantages being meager. Google is [considering removing its support](https://groups.google.com/a/chromium.org/g/blink-dev/c/K3rYLvmQUBY/m/vOWBKZGoAQAJ?pli=1) in Chrome and Chromium. As push notifications are optional and clients are allowed to discard them altogether, such a removal would have no impact on the functionality of websites and a mild effect on their performance.

[More information: _HTTP push notification_ article]

> **Warning:** Developer should not try to use HTTP push notifications to send out-of-bands messages to a web page. This use case is covered by the [Push API](/en-US/docs/Web/API/Push_API).

## HTTP version negotiation

Even if they use the same protocol name, `https`, in the URL, not all websites use the latest version of the protocol; the two peers must have a mechanism to determine which HTTP version they will use.

The protocol negotiation happens during the initial TLS handshake, using the {{Glossary("Application-Layer Protocol Negotiation")}} (ALPN) extension to TLS, thus preventing the exchange of additional messages and avoiding additional roundtrips and the additional latency it induces.

> **Note:** Although the specification defines a way to negotiate HTTP/2 without TLS, on top of an unencrypted HTTP/1.1 connection, browsers do not use this possibility as they prevent HTTP/2 on non-secure connections.

## What HTTP/2 doesn't solve

HTTP/2 is a significant improvement over the venerable HTTP/1.1 protocol. It doesn't solve all the problems of its predecessor, sometimes because they were out of scope (e.g., HTTP/2 doesn't alter the transport used), or because problems were unknown or hidden by other effects during its development.

### Interaction with transport layer mechanisms

TCP recovers lost packets. The destination quickly refuses newer messages during the recovery process, stalling all multiplexed streams on that connection. It is a consequence of TCP not knowing of the multiplexing happening higher in the stack. It is not able to determine which packets depend on the lost one and which ones do not.

This behavior leads to non-optimal performance, especially when the network is not good, losing numerous packets. HTTP/3 is addressing this problem by changing the transport and moving away from TCP to QUIC.

Due to the ubiquity of mobile communication, poor connectivity is much more common than envisioned; HTTP/2 has not removed the need for application-level optimizations like domain sharding.

### Security risks inherent to TCP not being encrypted

TCP was designed in the 1980s, and its handshake consists of the exchange of cleartext messages. As TCP itself performs neither encryption nor authentication, an attacker can manipulate subsequent TCP messages in some cases. Even if HTTPS, by bringing encryption at the session layer, somewhat mitigates this, TCP stays prone to {{Glossary("MitM attacks")}}. Changing the transport to QUIC would also remove this risk.

### Head-of-line blocking in the transport layer

HTTP/2 multiplexing aims at preventing {{Glossary("head-of-line blocking")}} in the application layer. Though better than older techniques like domain sharding, it proved insufficient to avoid all head-of-line blocking: TCP implements congestion-preventing mechanisms that lead to additional head-of-line blocking of messages, this time in the transport layer. Changing the transport to a protocol preventing this from happening would improve the overall performance.

### Cost of establishing new connections

## Deployment history

## Specifications

| Specification                                                                                     |
| ------------------------------------------------------------------------------------------------- |
| [RFC 7540: Hypertext Transfer Protocol Version 2 (HTTP/2)](https://httpwg.org/specs/rfc7540.html) |
| [RFC 8740: Using TLS 1.3 with HTTP/2](https://httpwg.org/specs/rfc8740.html)                      |
| [RFC 7541: HPACK: Header Compression for HTTP/2](https://httpwg.org/specs/rfc7541.html)           |

## Related info

- Official [HTTP/2](https://http2.github.io) page and its [FAQ](https://http2.github.io/faq)
- [Introduction to HTTP/2, by Ilya Grigorik & Surma](https://developers.google.com/web/fundamentals/performance/http2)  **TBD: Add this link to the issue of jpmedley**
- [How does HTTP/2 solve the Head of Line blocking (HOL) issue](https://community.akamai.com/customers/s/article/How-does-HTTP-2-solve-the-Head-of-Line-blocking-HOL-issue?language=en_US)
- [Better HTTP/2 Prioritization for a Faster Web](https://blog.cloudflare.com/better-http-2-prioritization-for-a-faster-web/)
- [HPACK, the silent killer feature of HTTP/2](https://blog.cloudflare.com/hpack-the-silent-killer-feature-of-http-2/)
- Patrick McManus' message + HTTP/2 FAQ entry  ** TBD: **
