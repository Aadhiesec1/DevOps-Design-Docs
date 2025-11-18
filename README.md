# AWS Architecture Design 

## 1. UI (Frontend) Architecture Design Overview
This document describes a frontend (UI) architecture hosted on AWS designed for high performance, security, and availability.  
The design uses a serverless CDN-backed approach (CloudFront + S3) with AWS WAF for protection.

---

## Architecture Diagram

**Embedded diagram (PNG):**  
![UI Architecture Diagram](./ui-arch.png)

**Downloadable diagram (PDF):**  
[ui-arch.pdf](./ui-arch.pdf)

> Diagram flow (high level):  
> End User → Route 53 → CloudFront → WAF → CloudFront Edge Locations (AZ-A / AZ-B) → S3 (origin)

---

## Components

### 1. End User
Client (browser or mobile app webview) requesting the UI over HTTPS.

### 2. Route 53
DNS service that resolves the website domain and points the user to the CloudFront distribution. Route 53 returns the nearest edge based on routing policies/latency.

### 3. Amazon CloudFront
Global Content Delivery Network (CDN) that:
- Terminates TLS for client requests,
- Caches static assets (HTML, JS, CSS, images),
- Auto-scales across its global edge network to handle traffic spikes,
- Forwards non-cached requests to the origin (S3).

### 4. AWS WAF
Web Application Firewall attached to the CloudFront distribution to:
- Filter and block malicious or suspicious requests,
- Enforce custom rules (rate limits, IP blocks, geo block, SQLi/XSS protections),
- Reduce unwanted traffic before it reaches origin.

### 5. CloudFront Edge Locations (AZ-A, AZ-B)
Edge POPs located in multiple Availability Zones and regions. They serve cached content closest to users (lowers latency) and provide fault tolerance through distribution.

### 6. Amazon S3 (Static Website Hosting)
S3 bucket configured as the origin for CloudFront. Stores built frontend artifacts (React/Angular/Vue production build). Highly durable, highly available, and automatically scales to handle origin requests.

---

## How the UI is Hosted
Primary hosting approach:
- **S3 + CloudFront** (recommended for modern SPAs)
  - S3 stores the static build artifacts.
  - CloudFront distributes and caches content globally.
  - Route 53 provides DNS (alias to CloudFront distribution).
  - AWS WAF is attached to CloudFront for security.

Alternate hosting approaches (not used here but valid depending on requirements):
- **ALB + ECS / EC2** (if server-side rendering or dynamic backend is needed)
- **Amplify** (for quick CI/CD for static apps)

---

## Scaling Approach for UI

**Horizontal scaling**
- **CloudFront (CDN)**: automatically scales horizontally across many edge locations. It can absorb large bursts of traffic without provisioning servers.
- **S3 (origin)**: storage and bandwidth scale virtually without user-managed instances.
- No application servers → eliminates the need for autoscaling groups for the UI layer.

**Caching strategy**
- Set long `Cache-Control` for immutable assets (e.g., hashed JS/CSS).
- Set reasonable TTLs for HTML (or use Cache-Control + stale-while-revalidate if using SSR).
- Use CloudFront behaviors to segregate cache rules per path.

**Traffic routing & availability**
- Route 53 routes users to the nearest edge location (latency-based).
- CloudFront edge locations exist across multiple AZs/regions to ensure availability.
- S3 cross-region replication (optional) can add redundancy for origin bucket.

---

## How the System Handles Load

1. **DNS resolution**  
   - User resolves the domain via Route 53 and is directed to the nearest CloudFront edge.

2. **Edge serve (cache hit) — the fast path**  
   - CloudFront serves the requested static asset directly from the edge POP cache.
   - Most requests (90%+) should be cache hits — minimal latency and no origin load.

3. **Edge miss (cache miss) — origin fetch**  
   - CloudFront forwards the request to the origin (S3).
   - S3 serves the file; CloudFront caches it for subsequent requests.

4. **Security & filtering**
   - AWS WAF inspects requests (attached to CloudFront). Malicious or anomalous requests are blocked at the CDN layer, preventing origin overload.

5. **Autoscaling and resilience**
   - CloudFront’s global design automatically scales to handle millions of requests per second.
   - S3 remains highly available and durable; rarely becomes a bottleneck due to caching.
   - Multi-AZ edge distribution ensures regional or AZ failures have minimal user impact.

6. **Monitoring & alarms (recommended)**
   - **CloudWatch** for metrics (requests, error rates, cache hit ratio).
   - **CloudFront logs** + **S3 logs** for access analytics.
   - **AWS Config / GuardDuty** for security posture monitoring.

----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
