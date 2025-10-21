---
layout: post
title: "⚙️ Order in Chaos: How Rate Limiters Turn Traffic Storms into Steady Streams"
subtitle: "The Algorithms That Keep APIs (and Engineers) from Losing Their Minds"
tags: ["system-design"]
readtime: true
---

In today's interconnected digital landscape, web apps and services are constantly under pressure. One ond hand, apps need to serve thousands of requests per minute from overenthusiastic clients and on the other hand, apps also need to protect themselves from malicious bots trying to brute-force login into the app. Uncontrolled traffic can overwhelm even the most robust infrastructure, leading to degraded performance for legitimate users, unexpected outages, and inflated operational costs.

To prevent from drowning in a sea of requests, services rely on an elegant and powerful defense mechanism: the rate limiter.

A rate limiter enforces a cap on how many requests an entity (such as a user or client application) can make to an API or service within a defined time window. Think of it as a traffic cop for APIs ensuring that no single client monopolizes the system’s resources.

The key benefits of using a rate limiter include:
- DoS Protection: Denial-of-Service (DoS) and Distributed Denial-of-Service (DDoS) attacks flood a service with excessive requests to make it unavailable. By capping requests from specific sources, a rate limiter helps mitigate these threats.
- Resource Fairness: Rate limiters ensure fair resource distribution, preventing a few high-traffic clients from hogging CPU, memory, or bandwidth at the expense of others.
- Cost Management: For cloud-hosted services, request volume directly affects cost. Rate limiting helps control expenses by curbing unnecessary or abusive usage of expensive resources like API calls, database queries, or serverless function executions.

With the “why” established, let’s explore how rate limiters achieve this control by examining some of the algorithms commonly used by rate limiters to manage request flow.

#### 1. Token Bucket Algorithm

The Token Bucket algorithm is designed to strike a balance between flexibility and control. It allows short bursts of traffic while still enforcing a steady, average rate of requests over time.

At its core, the algorithm imagines a bucket filled with tokens, where each token represents permission to process one request. The bucket has a fixed capacity (known as the bucket size), defining the maximum number of tokens it can hold. Tokens are added to the bucket at a constant refill rate (for example, 10 tokens per second). If the bucket is already full, any new tokens are simply discarded.

Whenever a request arrives, the system checks the bucket. If at least one token is available, the system removes a token and processes the request. If the bucket is empty, the request must wait or be rejected. 

This mechanism ensures that the system can handle short-term spikes in traffic but not sustained overload. The bucket size determines how “bursty” the traffic can be (how many requests can be processed at once), while the refill rate defines the sustained throughput, the long-term average request rate the system can safely handle.

For instance, suppose the refill rate is 5 tokens per second and the bucket size is 20 tokens. This means the system can tolerate an initial burst of 20 requests instantly, but afterward, it will only process up to 5 new requests per second as tokens are replenished. If traffic slows down, tokens accumulate in the bucket. When a sudden surge happens later, the stored tokens allow the system to absorb it smoothly, up to the bucket’s capacity.

By enforcing an average rate over time, the Token Bucket algorithm helps prevent prolonged overload and creates a predictable, well-regulated flow of requests. It gracefully handles the natural burstiness of most applications and networks without overreacting to short spikes. Moreover, it’s computationally lightweight and efficient, making it an excellent choice for high-performance systems where every millisecond counts.

However, fine-tuning the parameters, `bucket_size` and `refill_rate`, is crucial. A bucket that’s too small may throttle legitimate bursts, while one that’s too large might overload your system. In distributed architectures, things get even trickier: since the service may run across multiple servers, you’ll need a centralized or synchronized mechanism for managing tokens (or use sticky sessions). Otherwise, if each server maintains its own independent bucket, clients could effectively bypass the intended global rate limit.


#### 2. Leaky Bucket Algorithm

The Leaky Bucket algorithm enforces a strict, constant rate of handling the requests, no matter how bursty or unpredictable the incoming traffic is. Its goal is to smooth incoming requests into a steady, predictable stream, much like water leaking from a bucket at a fixed rate.

As the name implies, the algorithm imagines a bucket with a small hole at the bottom. Water (requests) can be poured in at any speed, but it leaks out at a constant rate. If water is poured faster than it leaks, the bucket fills up. Once it’s full, any excess water overflows and is lost.

The algorithm is defined by two parameters: bucket size and the leak rate or the outflow rate.

The bucket is a FIFO queue with a fixed capacity, holding incoming requests that are waiting to be processed. When a request arrives, the system checks if there is space in the queue. The request is added to the end of the queue if there is space. If the queue is full, the request is discarded. Meanwhile, a separate “leak” process continuously pulls requests from the front of the queue at the constant outflow rate and processes them.

This setup ensures that the processing component always sees requests arriving at a uniform rate, even if they originally came in bursts. The leak rate determines the steady-state throughput of the system, while the bucket size controls how much burst traffic can be absorbed before requests start getting dropped.

Unlike the Token Bucket, where the bucket size defines the maximum number of requests that can be processed in a burst, in the Leaky Bucket, it defines the maximum burst that can be queued. A larger bucket allows more requests to wait, but it also increases overall latency since later requests in the queue must wait longer to be processed.

For example, suppose the outflow rate is 100 requests per second and the bucket size is 500. A sudden burst of 500 requests would completely fill the queue. The system would then process these requests at a steady 100 per second, taking 5 seconds to empty the queue. If another burst arrives while the bucket is still full, the new requests will be dropped.

In essence:
- The Leaky Bucket is ideal when you want a smooth, constant processing rate, such as streaming or batch-oriented systems.
- The Token Bucket, on the other hand, is better suited when you want to allow controlled bursts, for instance, in interactive or latency-sensitive APIs.

One key limitation of the Leaky Bucket is that it processes requests at the fixed outflow rate even when the system has spare capacity. The Token Bucket can “catch up” by processing bursts faster if resources are available, which can make better use of idle capacity. However, the Leaky Bucket’s predictability makes capacity planning and resource provisioning simpler as you always know exactly how much load the system will handle at any given time.

#### 3. Fixed Window Counter Algorithm

The Fixed Window Counter algorithm is the simplest and most intuitive rate-limiting strategy. Its straightforward design makes it a common starting point for many systems. In essence, it restricts the number of requests that can be made within discrete, fixed intervals of time, or windows.

The algorithm is configured using two parameters, window size and request threshold. For example, suppose we want to allow a maximum of 5 requests per minute for a particular API. In that case, the window size is 1 minute, and the threshold is 5 requests.

When a request arrives, the algorithm checks how many requests have already been made in the current window, the one covering the request’s timestamp. For instance, if the current time is `14:18:32` and our window size is 1 minute, the active window is `14:18:00 - 14:18:59`. Each incoming request increments the counter for this window until it reaches the defined threshold.

Once the counter hits the limit (in our case, 5 requests), any additional requests within that same window are rejected. When the next window begins, which happens when the clock ticks from `14:18:59` to `14:19:00`, the counter resets automatically. The new window starts fresh, independent of the previous one.

The algorithm is quite easy to implement and maintain. For each user, we typically only need to store a single counter and the start time of the window. This is very lightweight, especially in a distributed system using a centralized cache like Redis. It’s also user-friendly: clients know exactly when their quota will reset (e.g., at the top of each minute), which can help reduce frustration.

However, this simplicity comes with a notable drawback. Because the counter resets sharply at the window boundary, users can exploit timing to effectively double their allowed rate.

For instance, in the example above, if a user sends 5 requests at `14:32:58` and then immediately sends 5 more at `14:33:00`, they’ve effectively made 10 requests within a few seconds, even though the system is supposed to allow only 5 per minute.

This “edge burst” effect, where bursts occur near the boundary between two windows, is the main weakness of the Fixed Window algorithm.

#### 4. Sliding Window Log Algorithm

The Sliding Window Log algorithm improves upon the Fixed Window Counter by eliminating its major flaw, the “edge burst” problem. Instead of dividing time into rigid blocks, this algorithm uses a continuously moving time window to track requests more accurately.

For each entity being rate-limited (such as a user ID or IP address), the system maintains a log, typically implemented as a sorted set, that records the timestamps of all recent requests.

When a new request arrives, the system first purges the entity’s log, removing all timestamps older than the current window (for example, older than 60 seconds if the window size is one minute). It then counts the number of timestamps remaining in the log. If this count is below the threshold, the new request’s timestamp is added, and the request is processed. If the count is equal to or exceeds the threshold, the request is rejected.

This mechanism ensures that the log always reflects only the requests made within the most recent window, providing a perfectly accurate count of active requests within that time span.

The algorithm uses the same configuration parameters as the Fixed Window Counter, window size and request threshold, but it provides much more precise rate control. At any moment, it guarantees that no entity can exceed the defined limit within the configured window. This makes the Sliding Window Log algorithm ideal for systems where exact enforcement is critical, such as payment gateways, stock trading platforms, or APIs protecting sensitive resources where even short bursts could pose a security risk.

This algorithm solves the "boundary condition" problem of fixed window counter algorithm, but it is computationally expensive as for each request, the algorithm needs to purge a list, read it, and then update it. While these operations are efficient for small logs, they become increasingly expensive as request volumes grow.

The algorithm’s main drawback is its memory overhead: it must store a timestamp for every single request. For example, a rate limit of 10,000 requests per hour per user could require up to 10,000 timestamps for each active user. This makes the algorithm not very suitable for high throughput systems.

Because of this, while the Sliding Window Log algorithm provides excellent accuracy and fairness, it’s not ideal for large-scale systems with heavy traffic or long window durations. In such cases, it’s often replaced by more space-efficient variants like the Sliding Window Counter.

#### 5. Sliding Window Counter Algorithm

While the Fixed Window Counter algorithm is lightweight but imprecise, and the Sliding Window Log is accurate but memory-intensive, the Sliding Window Counter algorithm strikes a balance between the two. It blends the low memory overhead of the Fixed Window approach with the accuracy of the Sliding Window Log, offering a “best of both worlds” solution.

This hybrid algorithm tracks the number of requests made in two adjacent fixed windows, the current and the previous one, and then uses interpolation to estimate the true rate within the sliding window.

Let’s break it down with an example. Suppose we want to allow 100 requests per minute per user, and the current time is `13:47:45`. The two fixed windows being tracked are:
- Previous window: `13:46:00 - 13:46:59`
- Current window: `13:47:00 - 13:47:59`

Let’s say the user made:
- 80 requests in the previous window, and
- 100 requests so far in the current window.

Now, the “true” sliding window we care about spans the last 60 seconds, i.e., `13:46:45 - 13:47:45`. The algorithm estimates the number of requests in this real-time window by computing how much of it overlaps with each of the two fixed windows:
- The previous window overlaps for 15 seconds, `15/60 = 25%` of the total window.
- The current window overlaps for 45 seconds, `45/60 = 75%` of the total window.

Using this overlap, the estimated number of requests in the sliding window is:
`(0.25 × 80) + (0.75 × 100) = 95`

Since 95 is below the threshold of 100, the request is allowed.

This technique provides a close approximation of the actual request rate without the need to log individual timestamps. It only requires two counters per user, one for each window, making it both fast and highly scalable. The Sliding Window Counter is therefore a popular choice in high-performance distributed systems, where perfect accuracy isn’t mandatory but burst control and resource efficiency are critical.

Another advantage is that it mitigates the boundary burst problem of the Fixed Window algorithm. Because part of the previous window is factored into the new calculation, sudden spikes at the window edge are naturally smoothed out, preventing users from doubling their effective rate.

The main limitation, however, is that the algorithm assumes requests are evenly distributed within each fixed window. If all 80 requests in our example occurred during the last 15 seconds of the previous window, the actual rate would be higher than the estimate, meaning this method slightly underestimates burstiness.

Still, in most real-world scenarios, this trade-off is well worth it. The Sliding Window Counter offers a robust, efficient, and practical rate-limiting mechanism that balances precision with performance, making it a true workhorse for large-scale systems.

#### Which Algorithm to Use?

Rate limiting isn’t just about rejecting requests. It’s about shaping traffic into something predictable and fair. Each algorithm brings its own strengths:
- Token Bucket lets short bursts through while keeping a sustainable long-term rate.
- Leaky Bucket enforces a smooth, constant outflow making it perfect for evenly paced systems.
- Fixed Window Counter is simple and fast, but vulnerable to edge bursts.
- Sliding Window Log gives perfect accuracy at higher computational cost.
- Sliding Window Counter strikes the balance between precision and performance.

When designing your system, start with the question: Do I need precision, fairness, or scalability most? The answer will tell you which algorithm belongs in your toolbox.

In the next article, I'll focus more on the deployment of rate limiters.


