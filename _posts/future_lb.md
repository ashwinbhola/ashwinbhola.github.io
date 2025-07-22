## Global Server Load Balancing (GSLB): Distributing Traffic Across the Globe
Imagine your application serves users from all over the world. If all your servers are in one location (e.g., London), a user in Sydney will experience higher latency (delay) when accessing your application compared to a user in Paris. Worse, if that London data center goes offline due to a power outage or natural disaster, your app becomes unavailable to everyone, everywhere.

**Global Server Load Balancing (GSLB)** solves this by distributing user traffic to the **most appropriate data center** based on **location, latency, or availability**. This results in a **more resilient and responsive global infrastructure**.

> Note: The closest data center isn't always the best choice. GSLB may route a user to a data center with the lowest latency, best performance, or highest availability, even if it's not geographically nearest.

### How GSLB Works: DNS-Based Load Balancing
The most common and fundamental mechanism for GSLB is **DNS-based load balancing**. Here's how it works:
1. The GSLB service acts as the **authoritative DNS server** that manages the DNS records for your application's domain (`example.com`). 
2. When a client (or more accurately, its DNS resolver) asks for the IP address of `example.com`, the GSLB service makes a real-time decision.
3. Instead of returning a fixed IP, it chooses a data center based on several factors, like client location, current network latency, or data center health, and returns the IP of that selected data center.
4. The client’s device then connects directly to that IP.

DNS records have a **"Time-To-Live" (TTL)** value, which tells **DNS resolvers** how long they should cache (remember) a given IP address before requesting it again:
- **High TTL (e.g., several hours)**: Fewer DNS lookups, but slower reaction to outages. Clients may continue using an outdated IP even if a data center fails.
- **Low TTL (e.g., 60–300 seconds)**: More DNS traffic, but quicker rerouting in case of failure. This is typical in GSLB setups to improve responsiveness and availability.

### GSLB Routing Policies
GSLB systems can use different strategies to decide where to send each client:

##### 1. Latency-Based Routing
The GSLB system actively measures the network latency from various global locations (or from common DNS resolver locations) to each of your data centers. This can be done via **periodic "probes"** or using **real-user data**. When a DNS query comes in, the GSLB system uses its current latency measurements to determine which data center has the **lowest measured latency from the client's presumed location** (often inferred from the client's DNS resolver IP). It then returns the IP address of that lowest-latency data center.


> Note: "Common DNS resolver locations" refer to the local DNS resolvers provided by ISPs that end-users' devices (or home routers) are configured to use. It can also include popular public DNS resolvers like Google Public DNS (`8.8.8.8`) or Cloudflare DNS (`1.1.1.1`) because many users manually configure their devices to use them.

##### 2. Proximity-Based Routing
The GSLB system uses the **source IP address** of the client's DNS resolver to determine its **approximate geographical location**. It maintains a mapping of IP address ranges to geographical regions. Once the client's approximate location is determined, the GSLB system returns the **IP address of the data center that is geographically closest to that location**.


#### What Happens During a Data Center Failure?
If a data center becomes unavailable due to **hardware failure, software crash, or even a regional outage**, the GSLB service detects this using **active health checks**. These checks monitor the **availability and responsiveness of each data center**. The GSLB system immediately stops returning the IP addresses of the failed data center in response to new DNS queries. Instead, it directs all incoming new traffic to other healthy, active data centers. This shift happens **within the TTL window**, so **lower TTLs allow faster failover**. 

This capability makes GSLB essential for **disaster recovery and business continuity** in large-scale, global systems.

### Alternatives to DNS based GSLB
While **DNS-based GSLB** is the most prevalent and fundamental mechanism, especially for public internet-facing applications, there are a few other approaches or complementary techniques that can be considered for global traffic distribution.

#### 1. Anycast Routing
Instead of a unique IP address for each data center, multiple data centers advertise the same IP address to the internet. **Border Gateway Protocol (BGP)**, the routing protocol of the internet, then directs incoming traffic for that shared IP address to the **"closest" data center** based on network topology (**shortest path as determined by BGP**).

There's **no explicit DNS lookup** for different IPs for different users; everyone resolves to the same IP, but network routing handles the global distribution.

It's used very commonly for **Content Delivery Networks (CDNs)** and **highly distributed services** (like DNS root servers themselves) where content needs to be served from the closest point of presence.

#### 2. HTTP/Application-Layer Redirects
A **central web server or proxy** receives the initial client request (e.g., `www.example.com`). Based on its own logic (e.g., source IP geolocation, server health, load), it issues an **HTTP redirect** (a `302 Found` or `307 Temporary Redirect` response) to the client's browser, telling it to go to a specific URL or IP address in another data center. The client's browser then makes a new request to the redirected URL/IP.

This method introduces an **extra round trip**, increasing latency. It's also **only applicable to HTTP/HTTPS traffic**. 

This technique is often **used in conjunction with other routing methods as a fallback**.

#### 3. Client-Side GSLB
The application itself, particularly **thick clients** (like desktop software, mobile apps), can embed logic to determine the best data center. The client application might first query a **central API endpoint**, which provides a list of available data centers and their status/latency. The client then directly connects to the chosen data center. Or, the client might actively measure latency to predefined data centers and pick the best one.

This strategy requires **custom development** within the client application. It is **less suitable for pure web based applications where the browser has limited control over**.


While these techniques have their place, DNS-based GSLB remains the **most flexible and widely adopted approach** for global traffic distribution on the public internet. It integrates easily with existing DNS infrastructure, works across protocols, and requires no client modifications, making it a default choice for most large-scale systems.

