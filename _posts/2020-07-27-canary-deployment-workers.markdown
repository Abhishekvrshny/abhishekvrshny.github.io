---
layout: post
title:  "Canary Deployment for Queue Workers"
date:   2020-07-27
description: Various strategies for canary deployment of queue workers and their pros and cons.
absolute_image: "https://user-images.githubusercontent.com/12811812/88756371-1ca84e80-d181-11ea-86a4-cb90a6b272e3.png"
---

>Canaries were once regularly used in coal mining as an early warning system. Toxic gases such as carbon monoxide, methane or carbon dioxide in the mine would kill the bird before affecting the miners. Signs of distress from the bird indicated to the miners that conditions were unsafe.

#### Deployment of Microservices
A very well understood and practised aspect in SDLC is *deployment* of your code or *deployment* of your service. The world of microservices mostly follow *rolling deployments*, where the new version of the code is gradually rolled out to all instances of the service without requiring a downtime.

#### Canary Deployments
Canary deployment is a pattern for rolling out releases to a subset of users or servers. The idea is to first deploy the change to a small subset of servers, test it, and then roll the change out to the rest of the servers. The canary deployment serves as an early warning indicator with less impact on downtime: if the canary deployment fails, the rest of the servers aren't impacted.

The basic steps of a canary deployment are:

1. Deploy to one or more canary servers.
2. Test, or wait until satisfied.
3. Deploy to the remaining servers.
 
The test phase of the canary deployment can work in many ways. You could run some automated tests, perform manual testing yourself, or even leave the server live and wait to see if problems are encountered by end users. In fact, all three of these approaches might be used. 

![Canary Deployment](https://user-images.githubusercontent.com/12811812/88699453-fbb21000-d124-11ea-9fa1-2ba02320b4c5.png)

The monitoring of a canary deployment can also be automated by comparing deviations in key metrics of the canary server with that of a baseline server. 

At Razorpay, we use [Spinnaker](https://spinnaker.io/) for deployments and [kayenta](https://github.com/spinnaker/kayenta) for canary analysis.

#### Canary Deployments with Web Nodes
The canary deployment model with nodes or pods serving web or HTTP/gRPC traffic is quite straight forward and it includes the following.
1. *A predefined and fixed number of canary web nodes:* These nodes get the new version of the code deployed first.
2. *A predefined and fixed number of baseline web nodes:* These nodes have older version of the code running and they are used for comparing metrics with the canary nodes.
3. *Regular nodes:* They serve majority of the traffic for the service and can be scaled up or down.

![Canary Deployment with Web Nodes](https://user-images.githubusercontent.com/12811812/88699474-040a4b00-d125-11ea-944b-32480bcb66fd.png)

The canary deployment flow is illustrated in the diagram below which is self explanatory in itself.

![Canary Flow](https://user-images.githubusercontent.com/12811812/88704385-a0375080-d12b-11ea-9c52-67d3bb1f4c24.png)

**PS:** Database schema migrations cannot be canary tested as they are atomic and any changes to the schema apply to all types of nodes alike, canary or non-canary.

#### Canary Deployments with Queue Worker Nodes
Canary deployment for an application which makes use of async processing through queues and queue workers becomes a little complex. Before we go into canary deployment for queue workers, lets discuss a few points

##### Standard Deployment
How should a standard deployment involving web and worker nodes happen?

> Should the web nodes be deployed first or the worker nodes?

There are actually just two options here:
1. Deploy the web nodes first, and then the worker nodes.
2. Deploy the worker nodes first, and then the web nodes.

With web nodes getting deployed first, there is a possibility that messages produced by new version of code might be consumed by the old version of code running on the workers. This imposes an additional constraint that the code should always be forward compatible. This, in my opinion, is hard to achieve.

On the other hand, deploying all the worker nodes first before deploying the web nodes makes this easier. This is beacuse, in this case, the worker code has to be only backward compatible, which is easier to achieve than forward compatibility. 

##### Canary Deployment with a Dedicated Canary Queue
Lets now talk about canary deployment for queue workers and one of the approaches available is the use of a dedicated canary queue. In this approach, the deployment happens as follows:
1. Deploy the canary worker node.
2. Compare the canary metrics of canary worker node with that of baseline worker node.
3. Proceed ahead if canary analysis passes, else revert the canary worker node.
3. Deploy the canary web node.
4. Compare canary metrics of web and worker canary nodes with that of baseline nodes.
5. Proceed if all fine, else revert the canary deployment.
6. Deploy rest of the worker nodes.
7. Deploy rest of the web nodes.

![Canary With Dedicated Queue](https://user-images.githubusercontent.com/12811812/88756371-1ca84e80-d181-11ea-86a4-cb90a6b272e3.png)

**Pros:**
1. Dedicated canary setup which ends up with deterministic testing of new code end to end.
2. It also helps in testing backward compatibility with 2 phase canary analysis.

**Cons**
1. Additional infrastructure components in terms of dedicated canary queues.
2. Additional logic on web nodes to selectively push messages to a canary queue. This becomes more complex when you have multiple queues for various use cases.

##### Canary Deployment with a Common or Shared Queue
Another approach to canary deployment of worker nodes is to route all messages through a common queue. In this approach, the deployment happens as follows:
1. Deploy the canary worker node.
2. Compare the canary metrics of canary worker node with that of baseline worker node.
3. Proceed ahead if canary analysis passes, else revert the canary worker node.
4. Deploy new version of code to all the worker nodes if canary analysis passes, else revert the canary worker node.
5. Deploy the canary web node.
6. Compare the canary metrics of canary web node with that of baseline web node.
7. Deploy the new version of code to all web nodes if canary analysis passes, else revert the canary web node.

![Canary With Common Queue](https://user-images.githubusercontent.com/12811812/88714211-15aa1d80-d13a-11ea-9a39-2ef39c61e997.png)

**Pros**
1. Tests backward compatibility of worker code when consuming older messages.
2. Simplistic setup with no additional and dedicated canary queues.

**Cons**
1. Does not test the happy flow deterministically i.e newer messages being consumed well by new version of the worker. Although `canary analysis 2` in `phase 2` does test this scenario, but the effectiveness would only depend on the volume of new messages coming to the canary and the baseline workers, which would be low due to canary web nodes getting a small percentage of traffic and this small percentage of traffic is again distributed amongst all the available worker nodes.

#### Summary
You can choose any of the above mentioned canary deployment strategies based on your requirements and needs. The dedicated queue approach is good for testing both the happy flow and the backward compatibility while the common queue approach is only good for testing backward compatibility in consuming messages. Testing backward compatibility is important as the deployment happens in a rolling fashion and a queue can always have residual messages from the older code. The shared queue approach is preferrable when you need simplicity in your code/infrastructure and you are confident that the happy flow is well tested through your functional/intergration tests. The dedicated queue approach would test all the scenarios but comes with its own complexity.


**Happy Deployments!**








