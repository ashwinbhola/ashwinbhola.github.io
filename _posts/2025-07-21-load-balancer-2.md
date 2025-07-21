---
layout: post
title: "Load Balancers"
subtitle: "Load Balancing algorithms"
tags: ["system-design"]
readtime: true
---

In the last post, we say how a load balancer is a highly efficient traffic cop for your servers. It stands in front of a group of servers (often called a "server farm" or "server pool") and decides which server should handle each incoming request.

Continuing our discussion, let's look at the different strategies, or "algorithms," for deciding how to distribute traffic.

## Static Algorithms
The key characteristic of static algorithms is that they make their decisions based on pre-defined rules, without taking into account the current state or load of the servers. They don't actively monitor server CPU usage, memory, or network traffic. This makes them simpler to implement and often faster, but also less adaptable to fluctuating server conditions.

### 1. Round-Robin
This is the simplest and the most straightforward algorithm of all. For an array of backend servers, the load balancer forwards the request in a sequential cyclical manner.

Imagine that we have 3 servers A, B, C, and 5 requests `R1` to `R5` hitting our load balancer. If the load balancer is configured to use round robin routing algorithm, the requests would get distributed as:

| Server          | Request     |
| --------------- | ----------- |
| Server A        | `R1`, `R4`  |
| Server B        | `R2`, `R5`  |
| Server C        | `R3`        |

It's very easy to understand and implement. It's biggest weakness is that it treats all requests as same size and all servers as same capacity. For example: If Server C above was a more powerful server than Server A, they still got treated equally. With equal number of requests to all servers, the powerful ones are underutilized and the slower ones are overutilized.

Also, if session stickiness is needed, Round Robin can be problematic.

### 2. Weighted Round-Robin
Weighted Round Robin is an enhancement to the basic Round Robin algorithm. It addresses the limitation of treating all servers equally by assigning a "weight" to each server. This weight represents the server's processing capacity or its ability to handle more requests relative to other servers. Servers with higher weights receive a proportionally larger share of the incoming requests. These weights need to be manually determined and assigned, which might require some initial testing and tuning.

In the above example, if we manually assign the following weights to the servers:
- Server A (New, Powerful): Weight = 4
- Server B (Standard): Weight = 2
- Server C (Older, Less Powerful): Weight = 1

Then, the load distribution looks like the following for request `R1` to `R8`:

| Server          | Request                       |
| --------------- | ----------------------------- |
| Server A        | `R1`, `R2`, `R3`, `R4`, `R8`  |
| Server B        | `R5`, `R6`                    |
| Server C        | `R7`                          |

Over time, Server A will handle roughly four times as many requests as Server C, and twice as many as Server B, which is a much more efficient distribution if those weights accurately reflect their capacities.

Weighted Round Robin allows us to leverage our more powerful servers more effectively, preventing them from being underutilized while weaker servers are overloaded. The algorithm is just as easy to set up as the naive round robin. 

We are still treating all requests as equal in weighted round robin. Also, the algorithm is still static and won't dynamically adjust weights based on real-time server load. These two disadvantages can lead to the situation where Server A suddenly gets bogged down by a complex task, and Weighted Round Robin continues to send it its allocated share of requests as the algorithm has no concept of heavy vs light requests and no monitoring of real-time server load. This will potentially lead to performance issues for Server A and the users it serves.

### 3. IP Hash (Source IP Hash)
In this algorithm, a cryptographic hash function is applied on the incoming client's IP address to determine which server should handle the request. The result of the hash function is then used to map the request to a specific server in the server pool. This means that all requests from the same IP will go to the same backend server.

IP Hash algorithm is primarily used for session persistence or session stickiness. Many web applications require that a user's entire "session" (e.g., logging in, adding items to a shopping cart, filling out a form) remains on the same server. If subsequent requests from the same user were sent to different servers, the application might lose track of the user's state, leading to a broken or frustrating experience. Thus, this approach helps us avoid the need for an external storage for sticky sessions.

If many clients are behind a single NAT device (like all users in an office building or many mobile users sharing a common cellular gateway IP), all their requests will appear to come from the same source IP address. This means all their traffic will be directed to a single server, potentially overwhelming it while other servers remain idle. This creates a "hotspot" server.

If you add or remove a server from the pool, the hashing logic changes, potentially causing many active sessions to be redirected to different servers, leading to lost session data for those users. Also, if the assigned server for a particular IP address goes down, the client will experience an outage until the load balancer can detect the failure and potentially rehash or re-route the client (though this often involves a timeout and re-connection, which isn't seamless). After the clients are reconnected to a different backend server, their session data might be lost.

## Dynamic Algorithms
Static load balancing algorithms, while simple, have a significant limitation: they are "dumb" regarding the real-time state of your servers. They don't know if a server is overloaded, underutilized, or even if it's about to fall over. This is where dynamic load balancing algorithms come into play.

Think of our traffic cop analogy again. A dynamic load balancer isn't just sending cars down a pre-determined route. It's actively watching traffic cameras, listening to radio reports, and checking for accidents or road closures, then directing traffic to the clearest, fastest routes available at that very moment.

### 1. Least Connections
The load balancer sends an incoming request to the server that has the least number of active connections at the time. The underlying assumption is that the server with the fewest active connections is the least busy and therefore best equipped to handle a new request efficiently. An "active connection" typically refers to a busy TCP connection between the load balancer and the server.

To accomplish this, the load balancer continuously monitors the number of active connections to each server in its pool. When a new request arrives, the load balancer queries its internal state to identify the server with the lowest count of active connections. This requires the load balancer to actively track and update active connection counts, adding a slight computational overhead compared to static methods.

In our running example, if this is the current internal state of the load balancer:
- Server A: 5 Active Connections
- Server B: 4 active connections
- Server C: 6 active connections

Then, the subsequent requests `R1` to `R4` might be distributed as follows (assuming no connection is closed during distribution of these 4 requests):

| Server          | Request                       |
| --------------- | ----------------------------- |
| Server A        | `R3`                          |
| Server B        | `R1`, `R2`                    |
| Server C        | `R4`                          |

This routing strategy is particularly effective for applications where connections can remain open for extended periods (e.g., chat applications, streaming services, or database connections), as it balances the load based on actual concurrent activity. Unlike static methods, the load balancer now reacts to the actual load on servers, leading to better resource utilization. If some requests are short and others are long, Least Connection helps balance the ongoing burden.

The main weakness is that the algorithm assumes that all active connections are consuming the same amount of server resources. A server with 8 active connections can be idle as all the requests might be waiting on some external DB query response while a server with 4 active connections might be utilizing all its compute resources to serve the response.

### 2. Weighted Least Connections
Similar to Weighted Round Robin, the weight indicates a server's relative capacity or processing power. However, instead of simply distributing requests proportionally to weights, this algorithm calculates a "score" for each server using both the number of active connections and the server's capacity to determine the least loaded server. This is often done by dividing the number of active connections by its weight (or a similar calculation to normalize the load).
```
Formula (common interpretation): Load Score = Active Connections / Weight
```

Each incoming request then directed to the server with the lowest "Load Score." Setting appropriate weights can be crucial and might require initial calibration and monitoring.

In the above example, if this is the current internal state of the load balancer:
- Server A: 5 Active Connections; Weight = 4
- Server B: 4 active connections; Weight = 2
- Server C: 6 active connections; Weight = 1

Then the current load score of each server is:
- Server A: load score = 1.25
- Server B: load score = 2
- Server C: Weiload scoreght = 6

Thus, an incoming request will go to Server A, changing the load scores to the following:
- Server A: load score = 1.5
- Server B: load score = 2
- Server C: Weiload scoreght = 6

This method ensures that more powerful servers can handle a proportionally higher number of active connections without being considered "more loaded" than less powerful servers with fewer connections. This prevents more powerful servers from being underutilized and less powerful ones from being overloaded. Thus, the algorithm combines the benefits of dynamic monitoring with the ability to account for server capabilities.

While it accounts for server capacity, it still assumes that connections on a given server consume resources equally, which may not always be true (as discussed for Least Connection).


### 3. Least Response Time
The Least Response Time (sometimes called Least Latency) algorithm directs new requests to the server that is currently responding fastest to health checks or existing requests. The idea is to send traffic to the server that can deliver content most quickly, thereby minimizing user perceived latency. If 2 servers have the same latency, the server with the fewest connections gets the request.

The algorithm directly aims to reduce latency for end-users, leading to faster page loads and a more responsive application. It responds quickly to performance fluctuations on individual servers, automatically diverting traffic away from slow-responding servers. A slow response time usually indicates an overloaded or unhealthy server, so this method implicitly does health checks as well. But the continuous health checks and response time measurement adds overhead to the load balancer.

> 📝 Note: Defining "response time" accurately can be complex. Is it just the time to establish a connection, or the time until the first byte of data is received, or the time until the entire response is complete? This choice can impact effectiveness.

In the scenario where one server is blazingly fast compared to all other servers, we might observe an oscillatory effect as the algorithm might create a "Thundering Herd". The fastest server might receive an disproportionately large number of requests, potentially leading to it becoming overloaded and then slowing down, causing the load balancer to shift traffic. After some time, the fastest server is not overloaded and the load balancer will again direct the incoming traffic to it, making it again overloaded.

### 4. Least Bandwidth
The Least Bandwidth algorithm directs new requests to the server that is currently utilizing the least amount of network bandwidth (e.g., measured in Mbps or MBps). The idea is that a server with less bandwidth usage is less saturated and thus better able to handle new network traffic. This is particularly useful for applications that are heavy on network I/O, such as media streaming, large file transfers, or CDN (Content Delivery Network) nodes. 

Consider a file-sharing service with three servers, all capable of delivering files:
- Server F: Currently sending/receiving 100 Mbps
- Server G: Currently sending/receiving 50 Mbps
- Server H: Currently sending/receiving 120 Mbps

New Request (e.g., for a large file download) arrives: Server G has the lowest bandwidth usage (50 Mbps). So, the new request goes to Server G. The algorithm requires the load balancer to continuously collect and analyze bandwidth usage data from each server.

> 📝 Note: Bandwidth usage can fluctuate rapidly, which might lead to frequent changes in the "least bandwidth" server, potentially causing instability if not smoothed out.

The algorithm directly optimizes for network throughput, ensuring that servers aren't choked by excessive data transfer. But a server with lower bandwidth usage might be streaming data to the user after heavy computation. This means that a server might have low bandwidth usage but be heavily loaded on CPU or memory due to complex computations. Least Bandwidth doesn't account for this, potentially leading to performance issues on that server.

### 5. Resource-Based (Adaptive/Predictive)
Resource-Based load balancing (often referred to as Adaptive or Predictive load balancing) is the most advanced category of dynamic algorithms. Instead of relying on a single metric like connections or response time, it makes decisions based on a comprehensive set of server resource metrics. Thus, it tries to get a holistic view of server health and load.

This typically involves installing a lightweight "agent" or daemon on each server in the pool. These agents continuously collect various server metrics, such as CPU usage, Memory Usage, Disk I/O, Network I/O, and Process Count (Number of running processes). The agents periodically report these metrics to the load balancer (or a central monitoring system that the load balancer consults). 

The load balancer (or a sophisticated algorithm within it) then uses these combined metrics to calculate an overall "load score" or "health score" for each server. This might involve complex algorithms, weighted averages, or even machine learning to predict future load. New requests are directed to the server identified as having the lowest load score. By considering multiple metrics, it prevents a server from becoming overloaded on CPU even if its network or memory usage is low (and vice-versa).

Consider the following scenario:
- Server A: 80% CPU, 30% Memory, 10 Mbps Network, Low Disk I/O
- Server B: 20% CPU, 90% Memory, 5 Mbps Network, High Disk I/O
- Server C: 40% CPU, 50% Memory, 80 Mbps Network, Moderate Disk I/O

If an incoming request is known to be CPU-intensive (e.g., image processing), the algorithm would likely send it to Server B (even though it has high memory/disk usage, its CPU is free). If the request is memory-intensive (e.g., complex data query), it would avoid Server B and send it to Server A or C. In a general web serving scenario, the algorithm would aim to balance all these metrics to ensure no single resource becomes a bottleneck on any server.

Thus, the algorithm can adapt to diverse workloads and server performance characteristics. It tries to estimate the most accurate view of server health and load, leading to optimal resource utilization. 

But the algorithm requires more sophisticated load balancers and often involves deploying agents on servers, which adds complexity to setup, configuration, and maintenance. Also, constant collection and analysis of multiple metrics can introduce overhead on both the servers and the load balancer itself. Moreover, if an agent fails or reports inaccurate data, it can lead to incorrect load balancing decisions. In very large environments, collecting and processing metrics from thousands of servers can become a significant challenge.

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






