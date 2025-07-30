---
layout: post
title: "🔬 The Brains Behind the Balance: A Deep Dive into Load Balancing Algorithms"
subtitle: "From Simple Cycles to Resourceful Decisions: Understand How Digital Traffic Cops Decide Where Your Requests Go"
tags: ["system-design"]
readtime: true
---

In the last post, we discussed how a load balancer acts as a highly efficient traffic cop for your servers. It stands in front of a group of servers (often called a **"server farm"** or **"server pool"**) and **intelligently** decides which server will handle each incoming request.

Continuing our discussion, let's delve into the various strategies, or **"algorithms,"** load balancers employ to distribute traffic effectively.

## Static Algorithms
The defining characteristic of **static algorithms** is their **reliance on pre-defined rules** when making distribution decisions. They operate **without** considering the current state or load of the servers, meaning they don't actively monitor metrics like server CPU usage, memory, or network traffic. This approach makes them **simpler to implement** and often **faster**, but also **less adaptable to fluctuating server conditions**.

### 1. Round-Robin
This is arguably the **simplest** and **most straightforward** algorithm available. For an array of backend servers, the load balancer forwards incoming requests in a **sequential, cyclical manner**.

Imagine that we have three servers (A, B, and C), and five incoming requests, `R1` through `R5`, hitting our load balancer. If the load balancer is configured to use the Round-Robin routing algorithm, the requests would be distributed as follows:

| Server          | Request     |
| --------------- | ----------- |
| Server A        | `R1`, `R4`  |
| Server B        | `R2`, `R5`  |
| Server C        | `R3`        |


This algorithm is undeniably easy to understand and implement. However, its primary weakness lies in **treating all requests as if they are the same size and all servers as if they have identical capacity**. For example, if Server C in our scenario were significantly more powerful than Server A, both would still receive an equal number of requests. This can lead to an inefficient distribution where more powerful servers are underutilized, while slower ones become overutilized.

Furthermore, Round Robin can present challenges when **session stickiness** is a requirement.

### 2. Weighted Round-Robin
Weighted Round Robin serves as a significant enhancement to the basic Round Robin algorithm. It directly addresses the limitation of treating all servers equally by assigning a **"weight"** to each server. This weight typically represents a **server's processing capacity** or its ability to handle more requests relative to others. Consequently, servers with higher weights receive a proportionally larger share of incoming requests. It's important to note that these weights must be **manually determined** and assigned, a process that may require initial testing and fine-tuning.

In the above example, if we manually assign the following weights to the servers:
- **Server A** (New, Powerful): Weight = 4
- **Server B** (Standard): Weight = 2
- **Server C** (Older, Less Powerful): Weight = 1

Then, the load distribution looks like the following for request `R1` to `R8`:

| Server          | Request                       |
| --------------- | ----------------------------- |
| Server A        | `R1`, `R2`, `R3`, `R4`, `R8`  |
| Server B        | `R5`, `R6`                    |
| Server C        | `R7`                          |


Over time, Server A will handle roughly four times as many requests as Server C, and twice as many as Server B. This represents a much more efficient distribution, assuming those weights accurately reflect the servers' true capacities.

Weighted Round Robin effectively allows us to leverage our more powerful servers, **preventing them from being underutilized** while weaker servers are overloaded. Furthermore, the algorithm is almost as simple to set up as the basic Round Robin.

However, a key limitation remains: Weighted Round Robin still **treats all requests as equal** in terms of processing demands. Moreover, the algorithm remains **static**, meaning it **won't dynamically adjust weights** based on **real-time server load**. These two disadvantages can lead to a scenario where, for instance, Server A suddenly becomes bogged down by a particularly complex task. Despite this, Weighted Round Robin would continue to send it its allocated share of requests, as the algorithm has **no concept of "heavy" versus "light" requests** and **lacks real-time server load monitoring**. This can potentially lead to significant performance issues for Server A and, by extension, for the users it serves.

### 3. IP Hash (Source IP Hash)
With the IP Hash algorithm, a **cryptographic hash function** is applied to the **incoming client's IP address** to determine which server should handle the request. The output of this hash function is then used to map the request to a specific server within the server pool. This ensures that all subsequent requests originating from the same client IP address are consistently directed to the same backend server.

The IP Hash algorithm is primarily utilized for achieving **session persistence**, often referred to as **session stickiness**. Many web applications necessitate that a user's entire "session", such as logging in, adding items to a shopping cart, or filling out a form, remains tied to the same server. If subsequent requests from the same user were routed to different servers, the application could easily lose track of the user's state, resulting in a broken or frustrating experience. Consequently, this approach effectively helps us **avoid the need for external storage solutions for sticky sessions**.

However, a significant drawback arises if numerous clients are operating behind a single **Network Address Translation (NAT)** device. For example, all users within an office building or many mobile users sharing a common cellular gateway IP will appear to originate from the same source IP address. In such cases, all their traffic will be directed to a single server, potentially overwhelming it while other servers remain idle. This scenario creates an **undesirable "hotspot" server**.

Furthermore, **adding or removing a server** from the pool can be **disruptive**. When the server pool changes, the hashing logic recalculates, potentially causing many active sessions to be redirected to different servers. This often leads to **lost session data** for those affected users. 

Additionally, if the server assigned to a particular IP address becomes unavailable, the client will experience an outage until the load balancer can detect the failure and potentially rehash or re-route the client. Even then, this process often involves a **timeout and re-connection**, which is **far from seamless**, and after clients are reconnected to a different backend server, their original session data might be irretrievably lost.

To mitigate the rebalance disruption on adding or removing backend servers, more advanced implementations of IP Hashing often employ a technique called **Consistent Hashing**. Consistent Hashing is designed to minimize the number of keys (in this case, client IPs) that re-map to a different server when the size of the server pool changes. Instead of a **complete rehash**, only a small fraction of sessions would be affected, leading to **far less disruption** for users. While it adds complexity to the load balancer's logic, consistent hashing is vital for maintaining session stickiness in **dynamic server environments**.

## Dynamic Algorithms
Static load balancing algorithms, though **simple in their implementation**, suffer from a significant inherent limitation: they are **"blind" to the real-time operational state of your servers**. They possess **no awareness** of whether a server is currently overloaded, underutilized, or even on the verge of failure. This crucial gap in intelligence is precisely where dynamic load balancing algorithms prove invaluable.

Let's revisit our traffic cop analogy. A dynamic load balancer doesn't merely dispatch vehicles along a pre-determined route. Instead, it actively monitors traffic cameras, processes real-time radio reports, and vigilantly checks for accidents or road closures. Armed with this up-to-the-minute information, it then intelligently directs traffic towards the clearest and fastest routes available at that precise moment.

### 1. Least Connections
The Least Connections algorithm directs an incoming request to the server currently handling the **fewest active connections**. The core assumption behind this approach is that the server with the lowest number of ongoing connections is the **least busy** and, consequently, best positioned to efficiently process a new request. An **"active connection"** in this context typically refers to an established **busy TCP connection** between the load balancer and the backend server.

To achieve this, the load balancer continuously monitors and maintains a **real-time count of active connections** for each server in its pool. Upon the arrival of a new request, the load balancer queries its internal state to pinpoint the server with the lowest active connection count. This constant tracking and updating of connection counts introduce a **slight computational overhead** compared to simpler, static methods.

Using our ongoing example, consider the following current internal state of the load balancer:
- **Server A**: 5 Active Connections
- **Server B**: 4 Active Connections
- **Server C**: 6 Active Connections

Given this state, subsequent requests `R1` to `R4` might be distributed as follows (assuming no connections are closed while these four requests are being distributed):

| Server          | Request                       |
| --------------- | ----------------------------- |
| Server A        | `R3`                          |
| Server B        | `R1`, `R2`                    |
| Server C        | `R4`                          |


This routing strategy is particularly effective for applications where connections can remain open for **extended periods** (e.g., chat applications, streaming services, or database connections), as it balances the load based on **actual concurrent activity**. 

Unlike static methods, the load balancer now **actively reacts** to the real load on servers, leading to significantly **better resource utilization**. If a mix of short and long running requests is present, Least Connections helps to balance the ongoing workload effectively.

However, the main weakness of this algorithm is its assumption that all active connections consume the same amount of server resources. For instance, a server with eight active connections might be largely idle because all those requests are waiting for an external database query response. Conversely, a server with only four active connections could be fully utilizing all its compute resources to actively serve its responses.

### 2. Weighted Least Connections
Similar to Weighted Round Robin, the **"weight"** in Weighted Least Connections signifies a server's relative capacity or processing power. However, instead of merely distributing requests proportionally to these weights, this algorithm calculates a **"load score"** for each server. This score intelligently combines both the number of active connections and the server's designated capacity to identify the truly least loaded server. 

A common way of computing the "load score" for a server involves dividing the number of active connections by its assigned weight to normalize the perceived load:
```
Formula (common interpretation): Load Score = Active Connections / Weight
```

Each incoming request is then directed to the server possessing the lowest "Load Score." Establishing **appropriate weights** is paramount to the algorithm's effectiveness and typically necessitates initial calibration and ongoing monitoring.

Let's use our previous example. If the current internal state of the load balancer is as follows:
- **Server A**: 5 Active Connections; Weight = 4
- **Server B**: 4 active connections; Weight = 2
- **Server C**: 6 active connections; Weight = 1

Then the current load score of each server is:
- **Server A**: load score = 1.25
- **Server B**: load score = 2
- **Server C**: load score = 6

Based on these scores, an incoming request would be routed to Server A, subsequently changing the load scores to:
- **Server A**: load score = 1.5
- **Server B**: load score = 2
- **Server C**: load score = 6

This sophisticated method ensures that more **powerful servers** can competently handle a **proportionally higher number of active connections** without being erroneously considered "more loaded" than less powerful servers with fewer connections. This effectively **prevents powerful servers from being underutilized** while simultaneously preventing less powerful ones from becoming overloaded. Thus, the algorithm skillfully combines the **benefits of dynamic load monitoring** with the **crucial ability to account for individual server capabilities**.

While Weighted Least Connections significantly improves upon its predecessors by factoring in server capacity, it still operates under the **assumption that all active connections on a given server consume resources equally**. As discussed with the basic Least Connections algorithm, this assumption may not always hold true.


### 3. Least Response Time
The Least Response Time algorithm (sometimes also referred to as **Least Latency**) routes new incoming requests to the server that is currently **responding the fastest**, whether to **health checks or existing requests**. The fundamental idea behind this strategy is to direct traffic to the server capable of delivering content most quickly, thereby effectively **minimizing user-perceived latency**. If two servers exhibit the same response time, the request is then directed to the server with the fewest active connections.

This algorithm directly aims to **reduce latency for end-users**, resulting in faster page loads and a more responsive application experience. It rapidly adapts to performance fluctuations on individual servers, automatically diverting traffic away from those that are slow to respond. Since a slow response time typically signals an overloaded or unhealthy server, this method inherently performs a type of **implicit health check**. However, the **continuous health checks** and **constant response time measurements** do introduce **additional overhead** for the load balancer itself.

> 📝 Note: Accurately defining "response time" can be complex. Is it merely the time taken to establish a connection, the time until the first byte of data is received, or the time until the entire response is complete? This specific choice can significantly impact the algorithm's overall effectiveness.

In scenarios where one server is exceptionally fast compared to all others, we might observe an **oscillatory effect**, potentially leading to a **"Thundering Herd"** phenomenon. The fastest server could receive a disproportionately large number of requests, leading to it becoming overloaded and subsequently slowing down. This slowdown would then cause the load balancer to shift traffic away. After some time, as the fastest server's load decreases, it becomes responsive again, prompting the load balancer to once again direct incoming traffic to it, thus restarting the cycle of potential overload.

### 4. Least Bandwidth
The Least Bandwidth algorithm directs new requests to the server that is currently utilizing the **least amount of network bandwidth** (e.g., measured in Mbps or MBps). The underlying principle is that **a server with lower bandwidth usage is less saturated** and therefore better equipped to handle new network traffic. This algorithm proves particularly useful for applications characterized by **heavy network I/O**, such as media streaming services, large file transfers, or Content Delivery Network (CDN) nodes.

Consider a file-sharing service operating with three servers, all capable of delivering files:
- **Server F**: Currently sending/receiving 100 Mbps
- **Server G**: Currently sending/receiving 50 Mbps
- **Server H**: Currently sending/receiving 120 Mbps

When a new request (for instance, a large file download) arrives, Server G, with its lowest bandwidth usage (50 Mbps), would receive the request. To function effectively, this algorithm necessitates that the load balancer continuously collects and analyzes real-time bandwidth usage data from each server.

> 📝 Note: Network bandwidth usage can fluctuate very rapidly. This volatility might lead to frequent changes in the identified "least bandwidth" server, potentially causing instability if the measurement isn't adequately smoothed or averaged over time.

While the algorithm directly optimizes for **network throughput**, ensuring that servers aren't **choked by excessive data transfer**, it does have a significant limitation. A server displaying lower bandwidth usage might, for example, be primarily streaming data to a user but the server might be performing intensive computations in parallel to generate each data segment to be sent. This implies that a server might have low bandwidth usage but be heavily loaded on CPU or memory due to complex processing. The Least Bandwidth algorithm **does not account for these other resource constraints**, potentially leading to performance issues on such a server despite its low network utilization.

### 5. Resource-Based (Adaptive/Predictive)
Resource-Based load balancing, often known as **Adaptive or Predictive load balancing**, represents the most advanced category among dynamic algorithms. Instead of relying on a single metric like active connections or response time, this approach makes routing decisions based on a **comprehensive set of server resource metrics**. The goal is to obtain a truly holistic view of each server's health and load.

This sophisticated method typically involves installing a **lightweight "agent" or daemon** directly on each server within the pool. These agents are responsible for **continuously collecting various critical server metrics**, including CPU usage, memory usage, disk I/O, network I/O, and the number of running processes. The collected metrics are then **periodically reported** by these agents to the load balancer or to a **central monitoring system** that the load balancer consults.

The load balancer then utilizes these combined metrics to compute an **overall "load score" or "health score"** for each server. This process might involve complex algorithms, weighted averages, or even machine learning techniques to predict future load patterns. New requests are subsequently directed to the server identified as having the lowest load score. By meticulously considering multiple metrics, this algorithm effectively prevents a server from becoming overloaded on its CPU, even if its network or memory usage is low (and vice-versa).

Consider the following scenario:
- **Server A**: 80% CPU, 30% Memory, 10 Mbps Network, Low Disk I/O
- **Server B**: 20% CPU, 90% Memory, 5 Mbps Network, High Disk I/O
- **Server C**: 40% CPU, 50% Memory, 80 Mbps Network, Moderate Disk I/O

If an incoming request is known to be CPU-intensive (e.g., for image processing), the algorithm would likely route it to Server B, despite its high memory and disk usage, because its CPU resources are relatively free. Conversely, if the request is memory-intensive (e.g., a complex data query), the algorithm would intelligently avoid Server B and send the request to Server A or C. In a general web serving scenario, the algorithm's objective would be to balance all these metrics to ensure that no single resource becomes a bottleneck on any server.

Ultimately, this algorithm offers **unparalleled adaptability** to diverse workloads and varying server performance characteristics. It strives to estimate the **most accurate view of server health and load**, thereby leading to **optimal resource utilization** across the entire server pool.

However, implementing Resource-Based load balancing demands more **sophisticated load balancers** and often requires deploying agents on individual servers, which significantly **increases the complexity of setup, configuration, and ongoing maintenance**. Furthermore, the constant collection and analysis of multiple metrics can introduce **notable overhead** on both the servers themselves and the load balancer. A **critical vulnerability** also exists: if an agent fails or reports inaccurate data, it can lead directly to incorrect load balancing decisions, potentially impacting service quality. In very large environments, the sheer scale of collecting and processing metrics from thousands of servers can indeed become a formidable challenge.

## Use Cases of Load Balancing Algorithms 
<br>

| **Algorithm**                  | **Application Examples**                                                                      |
| ------------------------------ | --------------------------------------------------------------------------------------------- |
| **Round Robin**                | - Basic web servers (serving static content)<br>- DNS servers (for simple domain resolution)<br>- Stateless API services (e.g., microservices where any instance can handle any request without session data)        |
| **Weighted Round Robin**       | - Web applications deployed on heterogeneous server hardware (some powerful, some less so)<br>- Hybrid cloud services  |
| **Least Connections**          | - Long-polling applications (e.g., instant messaging, push notifications)<br>- Real-time data streaming services<br>- Database connection pools (where the LB manages persistent connections)<br>- Stateful applications where connection duration varies greatly |
| **Weighted Least Connections** | - Similar to Least Connection, but with varied backend server capacities (e.g., a powerful chat server vs. a less powerful one in the same pool) |
| **IP Hash**                    | - E-commerce applications (e.g., shopping carts requiring session stickiness where cookies are not an option or as a fallback)<br>- Web applications with user login sessions that store state on the server<br>- Payment gateways (to ensure consistent routing for a transaction)  |
| **Least Response Time**        | - Real-time gaming platforms (e.g., multiplayer game servers)<br>- VoIP or video chat apps |
| **Least Bandwidth** | - Video streaming services<br>- Large file download services (e.g., software repositories)<br>- Content Distribution Networks (CDNs) where egress data is a primary concern |
| **Custom (Application-Aware)** | - Fintech fraud detection systems<br>- AI inference workloads with dynamic routing |

<br>
From dumb traffic cops to AI-powered directors, it's clear some algorithms are just smarter than others. In the next post, we'll see how to load balance load balancers themselves and how load balancers work at global level.

Thanks for reading!


