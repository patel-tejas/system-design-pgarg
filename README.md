
# 📘 System Design – Deep Dive (Beginner → Intermediate)

---

## 🚀 Introduction

System Design is about building systems that can handle massive traffic without crashing.

Examples:
- Amazon during sale
- Instagram reels traffic
- IRCTC tatkal booking

Goal:
- Scalable
- Reliable
- Fast

---

## 🧑‍💻 Basic Concept: Client & Server

### Client
User-side device:
- Browser
- Mobile app
- IoT

👉 Sends requests

---

### Server
A machine that:
- Runs 24/7
- Has public IP
- Processes requests

Examples:
- AWS EC2
- DigitalOcean

---

## 🌐 IP Address

Unique identity of a server on the internet.

Example:
142.251.42.110

---

## 🌍 DNS (Domain Name System)

Maps domain → IP

Example:
amazon.com → IP address

Flow:
1. Enter domain
2. DNS returns IP
3. Request goes to server

---

## ⚠️ Server Overload

Server has limited:
- CPU
- RAM

Too many users → crash

Example:
Exam result websites

---

## 📈 Scaling Techniques

---

### 🔼 Vertical Scaling (Scale Up)

Increase power of single server:
More CPU + More RAM

Pros:
- Simple

Cons:
- Expensive
- Downtime
- Limited

---

### 🔁 Horizontal Scaling (Scale Out)

Add multiple servers:
Server A  
Server B  
Server C  

Pros:
- No downtime
- Highly scalable

Cons:
- Needs load balancing

---

## ⚖️ Load Balancer

Distributes traffic across servers.

### Algorithms:
- Round Robin
- Least Connections
- IP Hash

Extra features:
- Health checks
- Failover
- SSL handling

---

## 🧩 Microservices Architecture

Break system into smaller services:

- Auth
- Orders
- Payment

Pros:
- Independent scaling
- Easier maintenance

Cons:
- More complexity

---

## 🚪 API Gateway

Entry point of system.

Handles:
- Routing
- Authentication
- Rate limiting

Flow:
Client → API Gateway → Service

---

## 🔄 Background Processing

Some tasks take time:
- Emails
- File processing

Use background workers instead of blocking user.

---

## 📬 Queue System (Async)

Tasks go into queue.

Flow:
Payment → Queue → Worker → Email

Benefits:
- Faster response
- Retry support

Tools:
- SQS
- Kafka
- RabbitMQ

---

## 📡 Pub/Sub (Event Driven)

One event → multiple services

Example:
Payment →
- Email
- SMS
- Notification

Difference:

Queue → One consumer  
Pub/Sub → Multiple consumers  

---

## 🧠 Fan-Out Architecture

Combines Pub/Sub + Queue

Flow:
Event → Multiple Queues → Workers

Advantage:
- Reliable + scalable

---

## 🚫 Rate Limiting

Limit requests to prevent abuse.

Example:
100 requests/min per user

Algorithms:
- Token Bucket
- Leaky Bucket

---

## 🗄️ Database Scaling

Single DB becomes bottleneck.

### Solution:

#### Read Replicas
- Reads → replicas
- Writes → primary

#### Sharding (advanced)
- Split data

---

## ⚡ Caching (Redis)

Store frequently used data in memory.

Flow:
Cache → If miss → DB → Save in cache

Benefits:
- Faster response
- Less DB load

---

## 🌍 CDN (Content Delivery Network)

Serve content from nearest location.

Flow:
User → Nearest CDN → Response

Examples:
- Cloudflare
- AWS CloudFront

---

## 🧱 Final Architecture

User
 ↓
DNS
 ↓
CDN
 ↓
Load Balancer
 ↓
API Gateway
 ↓
Microservices
 ↓
Queue / Workers
 ↓
Database + Cache

---

## 💡 Key Takeaways

- Prefer horizontal scaling
- Use async processing
- Cache frequently used data
- Protect with rate limiting
- Use microservices for large systems

---

## 🧠 Practical Advice

Try implementing:
- Redis caching
- Queue (BullMQ / Kafka)
- Load balancing (NGINX)
- Microservices with Docker

---

# 📘 System Design – Crash Course (Part 2)

---

## 🚀 Introduction

This part dives deeper into real-world system design concepts:

- Traffic patterns and scalability
- Serverless architecture (AWS Lambda)
- Virtualization vs Containers
- Kubernetes (container orchestration)

💡 System design is not fixed — it evolves based on use case and traffic.

---

## 🧠 Core Idea: Trade-offs

Every system balances:
- Scalability
- Reliability
- Cost

You cannot maximize all three at once.

---

## 📊 Traffic Patterns (Most Important)

Understanding traffic is key to system design.

---

### 🎥 Netflix (Predictable Traffic)

- Movies release on fixed dates
- Traffic spikes are predictable

Strategy:
- Pre-scale servers
- Cache content in CDN
- Prepare before release

---

### ▶️ YouTube (Unpredictable Traffic)

- Sudden spikes anytime
- Viral content, live streams

Strategy:
- Always keep extra capacity
- Handle unpredictable spikes

---

### 🏏 Hotstar (Mixed Traffic)

- Movies + Live streaming

Behavior:
- Match start → spike
- Player events → spike
- Users go back → spike on home API

Key insight:
Traffic in one service affects another.

---

## ⚡ Traditional Server Problems

- Manual scaling
- OS management
- Infrastructure complexity

---

## ☁️ Serverless (AWS Lambda)

### Idea:
Write code → Cloud handles everything

### How:
- Each request triggers a function
- Auto scaling per request

---

### ✅ Pros:
- No server management
- Auto scaling
- Pay per request

---

### ❌ Cons:

- Cold start latency
- Stateless (no memory)
- Execution time limits
- Vendor lock-in
- Hidden costs (API Gateway, S3, etc.)
- DB connection overload

---

## 🖥️ Virtualization (VMs)

### Idea:
Full virtual machines with OS

Pros:
- Consistent environment

Cons:
- Heavy (GBs)
- Slow
- Expensive

---

## 📦 Containerization (Docker)

### Idea:
Lightweight VMs without OS

- Share host OS
- Only include code + dependencies

---

### Benefits:
- Fast startup
- Lightweight
- Easy scaling
- Consistent environment

---

## ⚠️ New Problem

Many containers → hard to manage

---

## 🧠 Container Orchestration

Automating:
- Deployment
- Scaling
- Management

---

## ☸️ Kubernetes

### Built by Google

Inspired by Borg → open sourced

---

### What it does:

- Auto scaling
- Self-healing
- Rolling updates
- Load balancing

---

### Example:

Old containers replaced gradually → zero downtime

---

## 🧱 Modern Architecture

User  
↓  
CDN  
↓  
Load Balancer  
↓  
API Gateway  
↓  
Containers (Docker)  
↓  
Kubernetes  
↓  
Database  

---

## 🔥 Final Learning

- Traffic pattern defines architecture
- Serverless is simple but limited
- Containers are industry standard
- Kubernetes manages everything

---

## 🧠 Real World Practice

- Load testing before events
- Simulating traffic spikes
- Monitoring system limits

---



