---
layout: post
title:  "Request Hedging"
date:   2020-09-24
description: This post is on tail latencies and how request hedging can help curtail those. I also present one of my experiences of implementing it to combat the "tail at scale".
absolute_image: https://user-images.githubusercontent.com/12811812/94265735-af355600-ff56-11ea-8a58-da44799b3610.png
---


> Latency from a general point of view is a time delay between the cause and the effect of some physical change in the system being observed.
> 

Request rate, error rate and request duration or latencies ([RED](https://www.weave.works/blog/the-red-method-key-metrics-for-microservices-architecture/)) are some of the key metrics which are commonly monitored in any online system or service. 

### Tail Latencies
Latencies are generally plotted and monitored at percentiles of p90, p95, p99 etc. But, as you move towards the higher end of the spectrum, the tail latencies keep increasing. [The Tail at Scale](https://cacm.acm.org/magazines/2013/2/160173-the-tail-at-scale/fulltext) paper by Google, in fact, talks in detail about this. It also mentions various techniques to counter it. One of the techniques discussed is **Request Hedging**.

This post in on tail latencies, and how request hedging was used to curtail tail latencies in one of the high scale services at [Razorpay](https://razorpay.com/tech/). 

The service in picture here is the notifications' platform. Let's talk a little bit about it before jumping into the problem and its solution.

### The Notifications' Service
The notifications' service in Razorpay is a high throughput service, which receives requests to send out various types of notifications like webhooks, SMS, emails etc. at a peak rate of approximately 2000 requests/sec.

The complete architecture of the service is not really in the scope of this post, but to give a high level overview, the API layer of this service receives the requests and pushes the messages to SQS. The workers can then consume messages from SQS to hit the webhook or providers' endpoints, post which, the status is updated in the database. There is also a job which runs periodically to do exponential retries in case of failures.

![Notifications' Service](https://user-images.githubusercontent.com/12811812/94187938-51592d80-fec6-11ea-9e42-6cc63b1a81a9.png)

### The Problem Statment
The clients of this notifications' service had a strict timeout of 350ms in making any API call, but many a times, we would notice client timeouts which was not desirable. On further debugging, the tail latencies in pushing to SQS turned out to be the culprit. The p99.9 latencies would sometimes go upto 600ms!

*PS: We do have fallbacks in place in the clients so as not to loose any event.*

![SQS Push Latencies](https://user-images.githubusercontent.com/12811812/94189577-8a929d00-fec8-11ea-82db-267c9a2876c1.png)

### Request Hedging
Before talking about the solution we employed, let's look at how the Google paper defines **Hedged Requests**.

>A simple way to curb latency variability is to issue the same request to multiple replicas and use the results from whichever replica responds first. We term such requests "hedged requests" because a client first sends one request to the replica believed to be the most appropriate, but then falls back on sending a secondary request after some brief delay. The client cancels remaining outstanding requests once the first result is received.
>

### The Solution
While the Google paper talks about Hedged Requests primarily in the context of read requests, we used it in the write flow and piggybacked on the database and the cron job setup, which was already in place, to write the request to the database if SQS push doesn't succeed within the defined timeout period. One of the drawbacks of this approach is that it can lead to duplicate deliveries, but that was acceptable as we anyway promise [at least once delivery semantics](https://razorpay.com/docs/webhooks/#idempotency). 

![Hedged Requests](https://user-images.githubusercontent.com/12811812/94265735-af355600-ff56-11ea-8a58-da44799b3610.png)

### The Implementation
The implementation involved `Enqueue`ing the message in a different thread or goroutine with a strict timeout something similar to this:
{% highlight go %}
func (bq BaseQueue) enqueueWithSoftTimeout(msg string, timeoutInMs int, q Queue) (string, error) {

  // the channel holding the result
	c := make(chan struct {
		id  string
		err error
	}, 1)
	
	// an async goroutine to enqueue the message
	go func() {
		id, err := q.Enqueue(msg)
		c <- struct {
			id  string
			err error
		}{id: id, err: err}
	}()
	
	timeout := time.Duration(timeoutInMs) * time.Millisecond
	
	// wait till timeout for the result to appear in the result channel
	// else return
	select {
	case result := <-c:
		return result.id, result.err
	case <-time.After(timeout):
		return "", fmt.Errorf("enqueue timed out after %d ms", timeoutInMs)
	}
}
{% endhighlight %}

### Why not use http `transport` timeouts?
Now, this is an interesting question and an alternate approach to solve this problem could have been to use [http transport](https://golang.org/pkg/net/http/#Transport) timeouts like [Dialer Timeout](https://golang.org/pkg/net/#Dialer.Timeout), [TLS Handshake Timeout](https://golang.org/pkg/net/http/#Transport.TLSHandshakeTimeout) and ResponseHeaderTimeout. 

[Here](https://blog.cloudflare.com/the-complete-guide-to-golang-net-http-timeouts/) is a good primer on `net/http` timeouts in golang. 

![Client Timeouts](https://blog.cloudflare.com/content/images/2016/06/Timeouts-002.png)

Using these transport timeouts would have meant:
1. The sum of Dialer Timeout, TLS Handshake Timeout and ResponseHeaderTimeout would need to be less than the desired value of 350ms. 
2. The initial connection establishment which includes fetching IAM credentials, DNS resolution, SSL handshake and connection establishment, even before the payload can be sent and acknowledged, within 350ms would have been a close call.
3. It can even lead to connections never getting established in case of some minor degradation in any of the aforementioned phases.
4. Keeping relaxed transport timeouts along with a strict timeout in the application helps to mitigate this issue.

**Is there a better approach to achieve the same? Please do provide your suggestions in the comments :)** 



