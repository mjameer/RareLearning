# L4 vs. L7 Load Balancing

<img width="529" alt="image" src="https://github.com/user-attachments/assets/a2f611fa-991e-49fe-a30c-b46f178e2828" />


## Introduction
Load balancing is a critical component of distributed systems, helping distribute traffic across multiple servers to ensure availability, reliability, and performance. Load balancers operate at different layers of the OSI model, with Layer 4 (L4) and Layer 7 (L7) being the most common.

---
## Layer 4 (L4) Load Balancing
### Overview
L4 load balancing operates at the **transport layer** of the OSI model, primarily using **TCP/UDP** protocols. It makes forwarding decisions based on IP addresses and port numbers without inspecting the actual content of the traffic.

### How It Works
- Distributes traffic based on source and destination **IP addresses** and **ports**.
- Uses **Network Address Translation (NAT)** to route packets to backend servers.
- Does not inspect HTTP headers, payloads, or cookies.
- Works with both HTTP and non-HTTP traffic (e.g., FTP, DNS, SMTP).
- Supports both **TCP** (stateful) and **UDP** (stateless) load balancing.

### Advantages
✅ High performance due to minimal packet inspection.
✅ Works with any protocol, including non-HTTP services.
✅ Efficient for high-throughput applications.

### Disadvantages
❌ No deep packet inspection (cannot route based on HTTP headers or content).
❌ Limited control over traffic at the application layer.

---
## Layer 7 (L7) Load Balancing
### Overview
L7 load balancing operates at the **application layer**, making decisions based on HTTP headers, URLs, cookies, and content. It is commonly used for **web applications**.

### How It Works
- Inspects **HTTP(S)** traffic and routes requests based on URL paths, headers, cookies, or request body.
- Supports **content-based routing** (e.g., sending image requests to one set of servers and API requests to another).
- Enables **SSL termination** (decrypting SSL/TLS traffic before forwarding to backend servers).
- Can implement **advanced routing policies**, such as A/B testing and canary deployments.

### Advantages
✅ Granular control over traffic routing based on application-layer data.
✅ Supports **advanced features** like caching, compression, and authentication.
✅ Enables **security mechanisms**, such as blocking malicious requests.

### Disadvantages
❌ Higher latency due to deep packet inspection.
❌ More resource-intensive compared to L4 load balancing.

<img width="631" alt="image" src="https://github.com/user-attachments/assets/7e7abe9c-e5fb-421a-9ed8-b84dc413f775" />

---
## L4 vs. L7 Load Balancing - Quick Comparison
| Feature             | L4 Load Balancing  | L7 Load Balancing  |
|---------------------|-------------------|-------------------|
| OSI Layer          | Transport (Layer 4) | Application (Layer 7) |
| Protocols          | TCP/UDP             | HTTP/HTTPS, WebSockets |
| Routing Decisions  | IP & Port-based     | Content-based (headers, URL, cookies) |
| Performance       | Faster (low overhead) | Slightly slower (deep inspection) |
| Flexibility       | Limited              | Advanced routing & security features |
| SSL Termination   | Not supported        | Supported |
| Suitable for      | Non-HTTP traffic, high-throughput | Web applications, API gateways |

---
## When to Use Which?
- **Use L4 Load Balancing** when dealing with raw TCP/UDP traffic, databases, VoIP, or high-throughput scenarios.
- **Use L7 Load Balancing** when needing content-aware routing, HTTPS termination, API load balancing, or web application security.

### Example Technologies:
- **L4 Load Balancers**: Nginx (TCP/UDP mode), HAProxy (L4 mode), AWS ELB (Classic LB), IPVS.
- **L7 Load Balancers**: Nginx (HTTP mode), HAProxy (L7 mode), AWS ALB, Traefik, Envoy.

<img width="621" alt="image" src="https://github.com/user-attachments/assets/34c041ae-6840-4bb0-85ac-7d94a33cc585" />

---
## Conclusion
Both L4 and L7 load balancers have their strengths and are used in different scenarios. L4 is faster and protocol-agnostic, while L7 offers fine-grained control for web applications. The choice depends on **performance needs, flexibility, and traffic type**.
