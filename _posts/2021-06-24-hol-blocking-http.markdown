---
layout: post
title: Head-of-line (HOL) blocking in HTTP/1 and HTTP/2
date: 2021-06-24 00:00:00
description: Pipelining and multiplexing, both suffer from head-of-line blocking. But what is head-of-line blocking and how does it impact the performance in HTTP?
absolute_image: https://user-images.githubusercontent.com/12811812/123430251-9640fa80-d5e5-11eb-8a05-f011f2e37fc7.png 
---

#### What is head-of-line (HOL) blocking?
Head-of-line blocking or HOL blocking is a generic term in computer science that refers to a performance-limiting phenomenon that occurs when a line of packets is held up by the first packet. It can occur in network devices, transport protocols (for eg, in HTTP pipelining), or distributed systems (for eg, slow consumption of a message from a kafka topic partition).

As part of this post, we would specifically be talking about the issue of HOL blocking in HTTP which impacts both HTTP/1 and HTTP/2. But understanding HOL blocking in HTTP also requires an understanding of some key concepts like pipelining and multiplexing in HTTP which we would cover along the way. Let's get started!

#### Persistent connections and pipelining
HTTP is a layer 7 protocol that uses TCP on layer 4. In HTTP/1.0, by default, only one HTTP request could be sent over a single TCP connection and its response received unless `Connection: keep-alive` header is used. Even if a persistent connection is used with HTTP/1.0, requests could only be made serially on a connection. This is not a performant approach when it involves downloading [a lot of](https://httparchive.org/reports/state-of-the-web#reqTotal) resources per page from the server, while most browsers only support a few concurrent connections per domain as also [mentioned in the RFC](https://www.w3.org/Protocols/rfc2616/rfc2616-sec8.html#sec8.1.4). These limitations made web developers resort to techniques like [domain sharding](https://developer.mozilla.org/en-US/docs/Glossary/Domain_sharding) and [css sprites](https://css-tricks.com/css-sprites/).

![](https://user-images.githubusercontent.com/12811812/123263788-f9b02700-d516-11eb-934d-fb7c901737c4.png)

*<center>One request per connection</center>{: style="color:gray; font-size: 90%; text-align: center;"}*

This changed with HTTP/1.1 which made persistent connections a default behavior without the need to specify the `Connection: keep-alive` header.

![](https://user-images.githubusercontent.com/12811812/123265379-9de69d80-d518-11eb-9c61-e4c75e298995.png){: .center-image}

*<center>Multiple serial requests on a persistent TCP connection</center>{: style="color:gray; font-size: 90%; text-align: center;"}*

HTTP/1.1 also introduced pipelining, which allows multiple HTTP requests to be sent over a single TCP connection without waiting for the corresponding responses.

![](https://user-images.githubusercontent.com/12811812/123274426-21a48800-d521-11eb-857c-37030ea0fc61.png){: .center-image}

*<center>HTTP Pipelining</center>{: style="color:gray; font-size: 90%; text-align: center;"}*

Pipelining improves the performance of loading times of web pages, but it suffers from HOL blocking issue. 

#### HOL blocking with pipelining
HOL blocking can happen with pipelining at different places:
1. Pipelining requires the clients to start sending subsequent requests only when previous requests were sent successfully. But these requests could be sent without waiting for their responses.
2. It also requires the server to send responses in the order in which the requests were received.
3. The responses received by the client also need to be processed in order.

What this means is that the client and the server would always require to process requests and responses sequentially with pipelining.

![](https://user-images.githubusercontent.com/12811812/123361486-c4deb700-d58c-11eb-9e06-426bef71e4d5.png){: .center-image}

*<center>With HTTP pipelining subsequent requests or responses can be processed only after the previous ones are completed</center>{: style="color:gray; font-size: 90%; text-align: center;"}*

This ordering of requests and responses is required otherwise there is no way for the client and the server to figure out the corresponding requests and responses. HOL blocking can happen here if, let say, the processing of one of the requests or responses takes longer than others or is delayed due to retransmission. It would make other requests/responses wait for the previous ones to be complete. This also makes the implementation of pipelining buggy and error-prone, especially with proxies in between, due to which pipelining is disabled by default in almost [all](https://www.chromium.org/developers/design-documents/network-stack/http-pipelining) [browsers](http://kb.mozillazine.org/Network.http.pipelining).

![](https://user-images.githubusercontent.com/12811812/123401517-8369fe00-d5c4-11eb-9412-1ffd1b215d67.png)

*<center>HOL blocking with pipelining</center>{: style="color:gray; font-size: 90%; text-align: center;"}*

HTTP/2 introduced multiplexing with `streams` which **did** solve this issue to some extent, but not completely.

#### Multiplexing in HTTP/2
HTTP/2 addresses HOL blocking issue through request multiplexing, which eliminates HOL blocking at the application layer (L7), but HOL still exists at the transport (TCP) layer.

HTTP/2 does not modify the application semantics of HTTP in any way. All the core concepts, such as HTTP methods, status codes, URIs, and header fields, remain in place. Instead, HTTP/2 modifies how the data is formatted (framed) and transported between the client and server, both of whom manage the entire process, and hides all the complexity from our applications within the new framing layer. HTTP/2 allows the interleaving of request and response messages on the same connection through multiplexing which was a limitation with pipelining earlier. Multiplexing allowed a bi-directional flow of request and response segments between client and server without any need for explicit ordering between them. This was enabled by introducing the concept of streams. A stream is a logical single request/response communication denoted by a common stream id.

![](https://user-images.githubusercontent.com/12811812/123402598-86b1b980-d5c5-11eb-8b82-f5530efae1de.png){: .center-image}

*<center>HTTP multiplexing</center>{: style="color:gray; font-size: 90%; text-align: center;"}*

#### Frames, messages, and streams in HTTP/2
Before diving into HOL blocking in HTTP/2, it's important to understand its building blocks  and how they function.

**Frame**: A frame is the smallest unit of communication in HTTP/2. A frame carries a specific type of data, such as HTTP headers or payloads. All frames begin with a fixed 9 octet header. The header field contains, amongst other things, a stream identifier field. A variable-length payload follows the header. 

**Message**: A message is a complete HTTP request or response message. A message is made up of one or more frames.

**Stream**: A stream is a bidirectional flow of frames between the client and the server. Some important features of streams are:

* A single HTTP/2 connection can have multiple concurrently open streams. The client and server can send frames from different streams on the connection. Streams can be shared by both the client and server. A stream can also be established and used by a single peer.

* Either endpoint can close the stream.

* An integer identifies streams. The identifying integer is assigned by the endpoint which initiated the stream.

**With this, since each frame has a stream id associated with it, it removes the requirement to process HTTP requests and responses in order by the clients and servers. The HOL blocking issue is thus resolved at the HTTP layer, but it now moves to the TCP layer.**

#### HOL Blocking at TCP layer
To understand why that is the case, recall that every TCP packet carries a unique sequence number when put on the wire, and the data must be passed to the receiver in order. If one of the packets is lost en route to the receiver, then all subsequent packets must be held in the receiver’s TCP buffer until the lost packet is retransmitted and arrives at the receiver. Because this work is done within the TCP layer, HTTP layer has no visibility into the TCP retransmissions or the queued packet buffers and must wait for the full sequence before it can access the data. Instead, it simply sees a delivery delay when it tries to read the data from the socket. It is quite possible that the already received packets can form complete request/response message for a different stream and can be delivered to the HTTP layer without waiting for the delayed packet. This effect is known as TCP head-of-line (HOL) blocking.

[![](https://user-images.githubusercontent.com/12811812/123430251-9640fa80-d5e5-11eb-8a05-f011f2e37fc7.png)](https://user-images.githubusercontent.com/12811812/123430251-9640fa80-d5e5-11eb-8a05-f011f2e37fc7.png){:.lightbox-image}

*<center>HOL blocking with multiplexing (click to enlarge)</center>{: style="color:gray; font-size: 90%; text-align: center;"}*

**HTTP/3 or QUIC solves this issue by leveraging UDP instead of TCP as the transport protocol. But that's a topic for some other day. Stay tuned!**
