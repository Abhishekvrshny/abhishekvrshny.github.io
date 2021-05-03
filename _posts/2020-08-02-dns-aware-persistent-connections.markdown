---
layout: post
title: DNS-aware Persistent Connections
date: 2020-08-02 00:00:00
description: How should clients handle persistent connections when DNS IPs change? This post covers various aspects ranging from http1/2 and load balancers to answer this question.
absolute_image: >-
  http://user-images.githubusercontent.com/12811812/89064961-8777a680-d388-11ea-9145-d84c1b2149ff.png
---

This post is about an interesting discussion that came up recently. There was a problem faced to which **DNS-aware persistent http connections** seemed to be a probable solution. I can understand if you have never heard of such a concept before, probably because its not so common or probably it doesn't even exist\! We'll find that out soon.

![](https://user-images.githubusercontent.com/12811812/89064961-8777a680-d388-11ea-9145-d84c1b2149ff.png)

#### What exactly is the problem statement?

So, let me try to present a simplistic version of the problem. Let's assume there is a DNS configuration which points to a set of resources for a domain or a subdomain. The configured domain is used by an application to establish persistent http connections to the resources. The question is,

> What happens to the persistent http connections from the application, in case the DNS `A records` change? Should the client re-connect to the new set of IPs? Yes or NO, and why? Should this scenario be handled by the client or by the server? and how?

#### Is that even a valid problem?

Before I answer valid or not valid, let's see if this question has been raised elsewhere.

1. **golang `net/http`**\: There is a similar issue raised on golang's `net/http` package. [https://github.com/golang/go/issues/23427](https://github.com/golang/go/issues/23427)
2. **`square/okhttp`**\: Again similar issue with some discussions. [https://github.com/square/okhttp/issues/3374](https://github.com/square/okhttp/issues/3374)
3. **`akka/akka-http`**\: [https://github.com/akka/akka-http/issues/1226](https://github.com/akka/akka-http/issues/1226)
4. **`pgbouncer/pgbouncer`**\:A PostgreSQL connection pooler has similar issue reported. [https://github.com/pgbouncer/pgbouncer/issues/383](https://github.com/pgbouncer/pgbouncer/issues/383)
5. **cURL**\: A good discussion in `cURL`'s mailing list. [https://curl.haxx.se/mail/lib-2017-06/0021.html](https://curl.haxx.se/mail/lib-2017-06/0021.html)

Going through the discussions on these threads and mailing lists, there are two things which are clear:

1. This is indeed an issue or a question which is faced by many.
2. There is no clear solution that is being suggested or implemented anywhere.

#### Let's demo the issue

A rudimentary `golang` script to create persistent connections to `api.razorpay.com`

{% highlight go %}
package main

import (
	"io"
	"io/ioutil"
	"log"
	"net"
	"net/http"
	"time"
)
func main() {
	tr := &http.Transport{
		DialContext: (&net.Dialer{
			Timeout:   5 * time.Second,
			KeepAlive: 30 * time.Second,
			DualStack: true,
		}).DialContext,
		TLSHandshakeTimeout: 5 * time.Second,
		IdleConnTimeout:     100 * time.Second,
	}
	go func(rt http.RoundTripper) {
		for {
			time.Sleep(1 * time.Second)
			c := http.Client{Transport:tr}
			resp, err := c.Get("http://api.razorpay.com")

			if err != nil {
				log.Printf("Failed to do request: %v", err)
				continue
			}
	
			io.Copy(ioutil.Discard, resp.Body)
			resp.Body.Close()
			log.Printf("resp status: %v", resp.Status)
		}
	}(tr)
	ch := make(chan struct{})
	<-ch
}
{% endhighlight %}

Start this program, and we see it getting 200 OK.

{% highlight bash %}
2020/08/02 01:17:45 resp status: 200 OK
2020/08/02 01:17:46 resp status: 200 OK
2020/08/02 01:17:48 resp status: 200 OK
2020/08/02 01:17:50 resp status: 200 OK
{% endhighlight %}

Now, let's update `/etc/hosts` to simulate a change in DNS.

{% highlight bash %}
127.0.0.1 api.razorpay.com
{% endhighlight %}

And you would notice that the code keeps running as is. Why? Because DNS resolution just happens once and a change in DNS entries has no impact on already established connections as expected.

You would now say

![](https://i.imgflip.com/4a4tr2.jpg)

#### The solution

The solution to this problem, in my opinion, is simple.

The application or the client can do very little to solve this and it's the responsibility of the server to drain and terminate the connections for an IP if it wishes to recycle it. Terminating the connections would lead to the clients establishing a new connections by doing fresh DNS lookups, in which case they would get the new set of IPs. The server should wait for DNS to be propagated before terminating existing connections due to obvious reasons.

#### Why is this a responsibilty of the server to terminate connections?

This is because:

1. DNS resolved IPs corresponding to a domain or a sub domain can change due to variety of reasons, like, load balancer rotating IPs, weighted DNS entries etc. and not every time the older IPs may stop working.
2. The way http connection works is by resolving the DNS IPs before establishing connections to those IPs. DNS never comes into picture once a connection has been established and is `kept alive`.

#### Where do load balancers fit into the picture here?

The problem becomes more acute when we have load balancers or reverse proxies in between. A resource server wanting to get out of rotation can terminate existing http connections to it but the load balancer or the reverse proxy would return a `502 Bad Gateway` to the client in such a scenario. Although new requests would not land up in this server if its marked unhealthy and the healthcheck removes it from the load balancers set of backend targets, but would the existing connections be pro-actively terminated too in this case? I dont think all load balancers or reverse proxies do that.

#### What about http2?

With multiplexing in http2, the problem becomes more prominent and this brings a different perspective to the problem. Let's talk a little about L7 http2 load balancers, [envoy](https://www.envoyproxy.io/) for example. By the way, there is a good [blog post](https://blog.envoyproxy.io/introduction-to-modern-network-load-balancing-and-proxying-a57f6ff80236) by envoy on modern load balancers. 

#### `Strict DNS` based service discovery in envoy

Envoy supports two modes of DNS based [service discovery](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/service_discovery), `logical DNS` and `strict DNS`. With both these modes, envoy always does async DNS resolution for zero DNS blocking in the forwarding path, which is indeed a good optimisation. But the difference between strict and logical DNS is that, in `strict DNS`, envoy drains connections to any host which is not returned in the DNS query, while draining of existing connections does not happen in case of `logical DNS`.

`Strict DNS` capability in envoy thus provides **DNS-aware persistent connections**

> Should application clients then implement strict DNS, the way it works in envoy?

The answer to this question, in my opinion, is NO. The reason is again mentioned in envoy's documentation itself. This is what it says for `logical DNS`

> This service discovery type is optimal for large scale web services that must be accessed via DNS. Such services typically use round robin DNS to return many different IP addresses. Typically a different result is returned for each query. If strict DNS were used in this scenario, Envoy would assume that the cluster’s members were changing during every resolution interval which would lead to draining connection pools, connection cycling, etc. Instead, with logical DNS, connections stay alive until they get cycled. 

Strict DNS can be implemented in clients, only if, DNS starts sending the need for it in its responses through some mechanics. This can be a RFC btw :)

**UPDATE**: There is actually an RFC for [DNS Push Notifications](https://tools.ietf.org/html/rfc8765)

#### The 421 (Misdirected Request) status code in http2

A slightly related aspect is `421 (Misdirected Request)` status code in http2. [Section 9.1.2 of RFC 7540](https://tools.ietf.org/html/rfc7540#section-9.1.2) describes `421 (Misdirected Request)` as

> The 421 (Misdirected Request) status code indicates that the request was directed at a server that is not able to produce a response. This can be sent by a server that is not configured to produce responses for the combination of scheme and authority that are included in the request URI. Clients receiving a 421 (Misdirected Request) response from a server MAY retry the request – whether the request method is idempotent or not – over a different connection.

The 421 status code can be used to force clients to connect over a different connection, but this again puts the onus on the resource server to start sending 421 once it's out of rotation. I am not very sure if this is the right use of this status code and I haven't come across any implementations of this either. Also, is the client expected to terminate the existing connection if it receives a 421 is again not clear to me. `golang` http2 client still doesn't support 421. Some discussions here: [https://github.com/golang/go/issues/18341](https://github.com/golang/go/issues/18341).

#### But I would still want to handle this at my client.

There is a partial solution available in some languages/frameworks which can help mitigate this if you still need to handle it at the client. The `ESTABLISHED` connections can be limited by time (as done by [akka-http](https://doc.akka.io/docs/akka-http/current/common/timeouts.html#connection-lifetime-timeout) for example) or they can be limited by number of requests (as done by [envoy](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/cluster.proto) for example)

#### Summary

1. DNS aware persistent connectoons is a common topic raised at multiple places.
2. There is no standard client-side implementation to handle this.
3. Envoy's `strict DNS` pattern falls closer to a DNS-aware client for persistent connections, but it's not advisable to be used.
4. The only available client-side solutions are to use time bound or number of requests bound persistent connections available in some languages or frameworks.

**Your comments are welcome**
