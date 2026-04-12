
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

🔥 Learn by building, not just reading.
