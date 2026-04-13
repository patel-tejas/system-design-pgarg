# System Design for Beginners — Part 1
> **Channel:** Piyush Garg  
> **Series:** System Design Crash Course  

---

## Table of Contents

1. [The Basics: Client & Server](#the-basics-client--server)
2. [DNS Resolution](#dns-resolution)
3. [Vertical Scaling](#vertical-scaling)
4. [Horizontal Scaling & Load Balancer](#horizontal-scaling--load-balancer)
5. [Microservices Architecture](#microservices-architecture)
6. [API Gateway](#api-gateway)
7. [Queue Systems & Async Processing](#queue-systems--async-processing)
8. [Pub/Sub & Fan-Out Architecture](#pubsub--fan-out-architecture)
9. [Rate Limiting](#rate-limiting)
10. [Database Optimization](#database-optimization)
11. [Caching](#caching)
12. [CDN — Content Delivery Network](#cdn--content-delivery-network)
13. [Final Architecture Overview](#final-architecture-overview)

---

## The Basics: Client & Server

Every system starts with just two things:

```
┌─────────────────────────────────────────────────────┐
│                                                     │
│   CLIENT                        SERVER              │
│  ┌──────────┐    Request →    ┌──────────┐          │
│  │ 📱 Mobile│  ─────────────► │          │          │
│  │ 💻 Laptop│                 │ Public   │          │
│  │ 🔌 IoT   │  ◄─────────────  │ IP Addr  │          │
│  └──────────┘    Response ←   └──────────┘          │
│                                  IP: 10.2.3.4       │
│                                                     │
└─────────────────────────────────────────────────────┘
```

A **server** is simply a machine that:
- Runs 24/7
- Has a public IP address
- Can be accessed by anyone on the internet

> A cloud server (AWS EC2, DigitalOcean Droplet) is just someone else's machine with a public IP — nothing magical about it.

---

## DNS Resolution

IP addresses like `10.2.3.4` are hard to remember. DNS maps human-friendly names to IPs.

```
┌──────────────────────────────────────────────────────────┐
│                    DNS RESOLUTION FLOW                   │
│                                                          │
│  User types: amazon.com                                  │
│                    │                                     │
│                    ▼                                     │
│             ┌─────────────┐                             │
│             │  DNS Server │  ◄── "Global directory      │
│             │  (Global    │       of domain→IP maps"    │
│             │  Directory) │                             │
│             └──────┬──────┘                             │
│                    │  Returns: 10.2.3.4                  │
│                    ▼                                     │
│             ┌─────────────┐                             │
│             │   SERVER    │  ◄── Request sent directly   │
│             │ 10.2.3.4    │       to correct server      │
│             └─────────────┘                             │
│                                                          │
│  This process = DNS Resolution                           │
└──────────────────────────────────────────────────────────┘
```

When you buy a domain (e.g., `amazon.com`) and point it to an IP, you pay a fee to register it in DNS servers globally.

---

## Vertical Scaling

When your server gets overwhelmed, one approach is to add more resources to the **same machine**.

```
┌─────────────────────────────────────────────────────────┐
│                   VERTICAL SCALING                       │
│                                                         │
│   Before:                     After:                    │
│  ┌──────────────┐           ┌──────────────┐            │
│  │  2 CPU       │   ──────► │  64 CPU      │            │
│  │  4 GB RAM    │  Scale Up │  128 GB RAM  │            │
│  │              │           │              │            │
│  └──────────────┘           └──────────────┘            │
│                                                         │
│  ✅ Simple                  ❌ Has a physical limit      │
│  ✅ Single IP               ❌ Requires DOWNTIME         │
│  ✅ No Load Balancer needed ❌ Wasteful if low traffic   │
│                             ❌ Can't add CPU while live  │
└─────────────────────────────────────────────────────────┘
```

**The key problem:** You can't add CPU/RAM to a running machine. It must restart → **downtime**. Even 1 minute of downtime is unacceptable for Amazon during a sale.

> AWS solves this with auto-scaling policies: start with 2 CPU/4GB RAM, and automatically scale up during high load — paying only for extra hours used.

---

## Horizontal Scaling & Load Balancer

Instead of making one machine bigger, add more machines.

```
┌────────────────────────────────────────────────────────────────┐
│                   HORIZONTAL SCALING                           │
│                                                                │
│   All users → amazon.com                                       │
│                    │                                           │
│                    ▼                                           │
│            ┌───────────────┐                                   │
│            │ LOAD BALANCER │  IP: 10.2.3.7 (registered in DNS)│
│            │   (ELB)       │                                   │
│            └───────┬───────┘                                   │
│                    │  Round Robin Distribution                  │
│          ┌─────────┼─────────┐                                 │
│          ▼         ▼         ▼                                 │
│     ┌─────────┐ ┌─────────┐ ┌─────────┐                       │
│     │Server 1 │ │Server 2 │ │Server 3 │                       │
│     │IP: .5   │ │IP: .6   │ │IP: .7   │                       │
│     └─────────┘ └─────────┘ └─────────┘                       │
│                                                                │
│  Req 1 → Server 1                                              │
│  Req 2 → Server 2   (Round Robin Algorithm)                    │
│  Req 3 → Server 3                                              │
│  Req 4 → Server 1   (cycle repeats)                            │
│                                                                │
│  ✅ Zero downtime           ✅ Fault tolerant                  │
│  ✅ Scale in/out freely     ✅ Health checks built-in          │
└────────────────────────────────────────────────────────────────┘
```

**How the Load Balancer works:**
- Only the LB's IP is registered in DNS
- LB checks if each server is healthy before sending traffic
- If a server goes down, LB stops routing to it
- AWS calls this **ELB (Elastic Load Balancer)**

---

## Microservices Architecture

Real production apps split into independent services:

```
┌────────────────────────────────────────────────────────────────────┐
│                  MICROSERVICES ARCHITECTURE                        │
│                                                                    │
│   amazon.com/auth     → Auth Service    (4 servers)                │
│   amazon.com/orders   → Orders Service  (3 servers)                │
│   amazon.com/payments → Payment Service (2 servers)               │
│   amazon.com/         → API Service     (6 servers)               │
│                                                                    │
│  Each service has its own load balancer:                           │
│                                                                    │
│  ┌───────┐  ┌─────────────┐  ┌──────────────────────┐            │
│  │ Auth  │  │   Auth LB   │  │ S1  S2  S3  S4       │           │
│  │ /auth │─►│             │─►│ EC2 EC2 EC2 EC2       │           │
│  └───────┘  └─────────────┘  └──────────────────────┘            │
│                                                                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────────┐            │
│  │ Orders   │  │Orders LB │  │ S1  S2  S3            │           │
│  │ /orders  │─►│          │─►│ EC2 EC2 EC2            │           │
│  └──────────┘  └──────────┘  └──────────────────────┘            │
│                                                                    │
│  Why scale separately?                                             │
│  → People browse more than they buy                                │
│  → Auth is called on every request                                 │
│  → Payments are rare but critical                                  │
└────────────────────────────────────────────────────────────────────┘
```

---

## API Gateway

The entry point that routes requests to the correct microservice:

```
┌─────────────────────────────────────────────────────────────────────┐
│                         API GATEWAY                                 │
│                                                                     │
│  All users → DNS → API Gateway IP                                   │
│                                                                     │
│                   ┌─────────────────┐                               │
│    Any User ────► │   API GATEWAY   │                               │
│                   │                 │                               │
│                   │  Routing Rules: │                               │
│                   │  /auth    → LB1 │                               │
│                   │  /orders  → LB2 │                               │
│                   │  /payment → LB3 │                               │
│                   │  /        → LB4 │                               │
│                   └────────┬────────┘                               │
│                            │                                        │
│              ┌─────────────┼────────────┐                           │
│              ▼             ▼            ▼                           │
│         ┌──────────┐  ┌──────────┐  ┌──────────┐                   │
│         │ Auth LB  │  │Order LB  │  │  API LB  │                   │
│         └──────────┘  └──────────┘  └──────────┘                   │
│              │             │             │                           │
│         EC2 Servers   EC2 Servers   EC2 Servers                     │
│                                                                     │
│  Extra features API Gateway provides:                               │
│  ✅ Authentication/Authorization checks                             │
│  ✅ Route to Lambda, S3, external APIs                              │
│  ✅ SSL termination                                                  │
│  ✅ Rate limiting at the edge                                        │
└─────────────────────────────────────────────────────────────────────┘
```

> **Definition:** An API Gateway is a centralized entry point for API calls, acting as a reverse proxy that routes requests from clients to backend services.

---

## Queue Systems & Async Processing

When operations take time (e.g., sending emails), don't block the main flow.

### The Problem with Synchronous Processing

```
┌────────────────────────────────────────────────────────────────┐
│              SYNCHRONOUS — BAD FOR SCALE                       │
│                                                                │
│  User places order                                             │
│       │                                                        │
│       ▼                                                        │
│  Payment Service ──────────────────────────► Email Worker     │
│       │                                          │            │
│       │    WAITING...  (2-3 seconds)             │            │
│       │◄─────────────────────────────────────────┘            │
│       │                                                        │
│  Response to User                                              │
│                                                                │
│  ❌ Payment server waits for email                             │
│  ❌ Every payment blocks a thread                              │
│  ❌ Gmail rate limit can break payment flow                    │
└────────────────────────────────────────────────────────────────┘
```

### The Solution: Queue System (SQS)

```
┌────────────────────────────────────────────────────────────────┐
│              ASYNCHRONOUS WITH QUEUE — SCALABLE                │
│                                                                │
│  User places order                                             │
│       │                                                        │
│       ▼                                                        │
│  Payment Service ────► ┌───────────────────────┐              │
│       │                │      QUEUE (SQS)       │              │
│  Instant Response ◄─── │  [order1][order2][...] │             │
│                         └───────────┬───────────┘              │
│                                     │  pull events             │
│                         ┌───────────▼──────────┐               │
│                         │    Email Workers      │               │
│                         │  ┌───┐ ┌───┐ ┌───┐   │              │
│                         │  │W1 │ │W2 │ │W3 │   │              │
│                         │  └───┘ └───┘ └───┘   │              │
│                         └──────────────────────┘               │
│                                     │                           │
│                              Gmail API (10/sec limit)           │
│                                                                │
│  ✅ Payment doesn't wait       ✅ Workers scale independently   │
│  ✅ Rate limiting respected    ✅ Dead Letter Queue for failures │
│  ✅ Parallelism increases      ✅ Email arrives in 2-3s (async) │
└────────────────────────────────────────────────────────────────┘
```

**Pull vs Push Mechanisms:**

| Mechanism | How it works | Used in |
|-----------|-------------|---------|
| **Short Polling** | Ask queue every 1 second — "anything there?" | High frequency needs |
| **Long Polling** | Block for 10 seconds, collect all events at once | Cost-efficient, SQS default |
| **Push** | Queue invokes the worker directly | SNS notifications |

**Dead Letter Queue (DLQ):** If processing fails (e.g., Gmail is down), the message goes to a DLQ for retry after 5-10 minutes.

---

## Pub/Sub & Fan-Out Architecture

When one event needs to trigger **multiple** actions simultaneously.

```
┌─────────────────────────────────────────────────────────────────┐
│                    FAN-OUT ARCHITECTURE                         │
│                                                                 │
│  Payment happens                                                │
│       │                                                         │
│       ▼                                                         │
│  ┌───────────┐                                                  │
│  │    SNS    │  ← Simple Notification Service (Pub/Sub)         │
│  │ (Topic)   │                                                  │
│  └─────┬─────┘                                                  │
│        │  Broadcasts to all subscribers                         │
│   ┌────┴─────┬──────────┬──────────┐                            │
│   ▼          ▼          ▼          ▼                            │
│ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐                            │
│ │SQS   │ │SQS   │ │SQS   │ │SQS   │                            │
│ │Email │ │WhApp │ │ SMS  │ │Vendor│                            │
│ │Queue │ │Queue │ │Queue │ │Queue │                            │
│ └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘                            │
│    ▼        ▼        ▼        ▼                                 │
│  Email    WhApp     SMS    Vendor                               │
│ Worker   Worker   Worker    API                                 │
│                                                                 │
│  ✅ One event → many services notified                          │
│  ✅ Each queue has retry + DLQ guarantees                       │
│  ✅ Services are fully decoupled                                │
│                                                                 │
│  SQS alone = 1:1 (one consumer picks up message)               │
│  SNS + SQS  = 1:N (fan-out to many consumers)                  │
└─────────────────────────────────────────────────────────────────┘
```

**Real-world example — YouTube video upload:**
SNS fans out to: transcode to 4K → transcode to 480p → transcode to audio-only → generate thumbnail → update search index

---

## Rate Limiting

Protect your system from abuse, DDoS attacks, and overuse.

```
┌──────────────────────────────────────────────────────────────┐
│                      RATE LIMITING                           │
│                                                              │
│  Without Rate Limiting:                                      │
│  ┌──────┐                                                    │
│  │ Bot  │ ─── 10,000 requests/sec ──► Server 💥 CRASH        │
│  └──────┘                                                    │
│                                                              │
│  With Rate Limiting:                                         │
│  ┌──────┐                                                    │
│  │ Bot  │ ─── Request 6 ──► [429 Too Many Requests]          │
│  └──────┘                  (Rate limit: 5 req/sec/user)      │
│                                                              │
│  Common Rate Limiting Algorithms:                            │
│  ┌─────────────────────────────────────────────────────┐     │
│  │ TOKEN BUCKET                                        │     │
│  │  Bucket fills at fixed rate (e.g., 5 tokens/sec)   │     │
│  │  Each request consumes 1 token                      │     │
│  │  No token? Request rejected                         │     │
│  │  Allows burst traffic up to bucket size             │     │
│  └─────────────────────────────────────────────────────┘     │
│  ┌─────────────────────────────────────────────────────┐     │
│  │ LEAKY BUCKET                                        │     │
│  │  Requests queue up; processed at fixed rate         │     │
│  │  Smooths out bursty traffic                         │     │
│  │  Queue overflow = requests dropped                  │     │
│  └─────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────┘
```

Rate limiting is applied at multiple levels: per-user, per-IP, and even between internal services (e.g., respecting Gmail's 10 emails/sec limit).

---

## Database Optimization

A single database can't handle everything. Strategies to scale:

```
┌─────────────────────────────────────────────────────────────────┐
│                    DATABASE SCALING                             │
│                                                                 │
│  ┌──────────────┐                                               │
│  │   PRIMARY    │  ← All WRITES go here (source of truth)       │
│  │   DATABASE   │  ← Real-time READS also go here               │
│  └──────┬───────┘                                               │
│         │   Replication (slight delay)                          │
│    ┌────┼────┐                                                  │
│    ▼    ▼    ▼                                                  │
│  ┌────┐┌────┐┌────┐                                             │
│  │ RR ││ RR ││ RR │  ← READ REPLICAS                            │
│  └────┘└────┘└────┘                                             │
│    │                                                            │
│    └── Used for: Analytics, Reports, Non-real-time queries      │
│                                                                 │
│  Trade-off: Slight data delay in replicas (eventual consistency)│
│                                                                 │
│  Use Primary for: Live product prices, stock levels             │
│  Use Replica for: Analytics dashboards, reporting               │
└─────────────────────────────────────────────────────────────────┘
```

---

## Caching

Store frequently-accessed results in memory to reduce DB load.

```
┌──────────────────────────────────────────────────────────────┐
│                    CACHING WITH REDIS                        │
│                                                              │
│  Request: "Get product details for ID 123"                   │
│                    │                                         │
│                    ▼                                         │
│            ┌──────────────┐                                  │
│            │  Cache Check  │                                  │
│            │   (Redis)     │                                  │
│            └──────┬───────┘                                  │
│                   │                                          │
│         ┌─────────┴─────────┐                                │
│         ▼                   ▼                                │
│    CACHE HIT           CACHE MISS                            │
│    Return data         Query Database                        │
│    instantly ✅         │                                    │
│                         ▼                                    │
│                    Store in Redis                            │
│                         │                                    │
│                    Return data                               │
│                                                              │
│  Benefits:                                                   │
│  ✅ Reduces DB calls significantly                           │
│  ✅ Response time drops from 100ms → 1ms                     │
│  ✅ Redis = in-memory = extremely fast                       │
│                                                              │
│  Use cache for: product details, user profiles,             │
│                 homepage data, search results                │
└──────────────────────────────────────────────────────────────┘
```

---

## CDN — Content Delivery Network

Serve static content from servers physically close to users.

```
┌───────────────────────────────────────────────────────────────────┐
│              CONTENT DELIVERY NETWORK (CloudFront)               │
│                                                                   │
│   Origin Server (US)                                              │
│         │                                                         │
│         │ One-time load per region                                │
│    ┌────┴──────────────────────────────┐                          │
│    ▼                ▼                  ▼                          │
│ ┌──────────┐  ┌──────────┐  ┌──────────┐   Edge Servers          │
│ │  US Edge │  │  IN Edge │  │  CA Edge │   (CDN PoPs)            │
│ │  Server  │  │  Server  │  │  Server  │                         │
│ └──────────┘  └──────────┘  └──────────┘                         │
│      │               │             │                              │
│   US Users      IN Users       CA Users                          │
│                                                                   │
│  User in North India requests product photo:                      │
│  1. Request hits nearest CDN edge server (India)                  │
│  2. Cache HIT → Photo returned instantly ✅                       │
│  3. Cache MISS → Fetch from origin (US), cache it, return         │
│  4. Next India user → Cache HIT, no US trip needed               │
│                                                                   │
│  Anycast routing: One IP, user auto-routed to nearest edge server │
│                                                                   │
│  Benefits:                                                        │
│  ✅ Lower latency (physical proximity)                            │
│  ✅ Reduces load on origin server                                 │
│  ✅ Saves bandwidth costs                                         │
│  ✅ Can invalidate cache when content updates                     │
└───────────────────────────────────────────────────────────────────┘
```

---

## Final Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│              COMPLETE SCALABLE SYSTEM ARCHITECTURE                   │
│                                                                      │
│  Users (Global)                                                      │
│       │                                                              │
│       ▼                                                              │
│  ┌──────────────────────────────┐                                    │
│  │   CDN (CloudFront)           │ ← Cache static assets globally     │
│  └──────────────┬───────────────┘                                    │
│                 │ (cache miss)                                        │
│                 ▼                                                     │
│  ┌──────────────────────────────┐                                    │
│  │   API Gateway                │ ← Auth, routing, rate limiting     │
│  └───┬──────┬──────┬────────────┘                                    │
│      │      │      │                                                  │
│      ▼      ▼      ▼                                                  │
│   Auth LB Orders LB Payment LB   ← Elastic Load Balancers            │
│      │      │      │                                                  │
│    EC2s    EC2s   EC2s             ← Horizontally scaled servers      │
│      │      │      │                                                  │
│      └──────┴──────┘                                                  │
│              │                                                        │
│       ┌──────▼──────┐                                                 │
│       │ Redis Cache │ ← Check cache before DB query                  │
│       └──────┬──────┘                                                 │
│              │ (cache miss)                                           │
│       ┌──────▼──────┐                                                 │
│       │  Primary DB │ ← Writes + real-time reads                     │
│       └──────┬──────┘                                                 │
│              │                                                        │
│    ┌─────────┴─────────┐                                              │
│    ▼         ▼         ▼                                              │
│ Read Rep  Read Rep  Read Rep   ← Read Replicas (analytics)            │
│                                                                       │
│  Async Side:                                                          │
│  EC2s → SNS → SQS Email Queue → Email Workers → Gmail API            │
│             → SQS SMS Queue   → SMS Workers   → Twilio               │
│             → SQS WhApp Queue → WhApp Workers → Meta API             │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Key Concepts Summary

| Concept | What it does | AWS Service |
|---------|-------------|-------------|
| **DNS** | Resolves domain to IP address | Route 53 |
| **Load Balancer** | Distributes traffic among servers | ELB |
| **API Gateway** | Routes requests to correct microservice | API Gateway |
| **Vertical Scaling** | Add CPU/RAM to existing server | EC2 instance resize |
| **Horizontal Scaling** | Add more server replicas | Auto Scaling Group |
| **Queue System** | Async communication between services | SQS |
| **Pub/Sub** | One event → many subscribers | SNS |
| **Fan-Out** | SNS → multiple SQS queues | SNS + SQS |
| **Rate Limiting** | Throttle requests per user/IP | API Gateway throttling |
| **Caching** | Store results in memory | ElastiCache (Redis) |
| **Read Replica** | Offload reads from primary DB | RDS Read Replica |
| **CDN** | Serve content from nearest edge | CloudFront |

---

*Summary based on Piyush Garg's System Design for Beginners (Part 1)*

# System Design Crash Course — Part 2
> **Channel:** Piyush Garg  
> **Series:** System Design Crash Course  
> **Prerequisites:** Part 1 (Load Balancers, API Gateway, Queues, CDN)

---

## Table of Contents

1. [System Design Is Not One-Size-Fits-All](#system-design-is-not-one-size-fits-all)
2. [Traffic Patterns: Netflix vs YouTube vs Hotstar](#traffic-patterns-netflix-vs-youtube-vs-hotstar)
3. [Auto Scaling Policies](#auto-scaling-policies)
4. [Serverless Architecture (AWS Lambda)](#serverless-architecture-aws-lambda)
5. [The "Works on My Machine" Problem](#the-works-on-my-machine-problem)
6. [Virtualization (VMs)](#virtualization-vms)
7. [Containerization (Docker)](#containerization-docker)
8. [Container Orchestration (Kubernetes)](#container-orchestration-kubernetes)
9. [Deployment Strategies](#deployment-strategies)
10. [Summary: Evolution of Deployment](#summary-evolution-of-deployment)

---

## System Design Is Not One-Size-Fits-All

> *"System Design is something that evolves over time. You start, you monitor, your server crashes, you optimize."*

The goal of system design is a balance between two forces:

```
┌────────────────────────────────────────────────────────────┐
│                  THE SYSTEM DESIGN BALANCE                 │
│                                                            │
│   FAULT TOLERANT ◄────────────────────────► COST OPTIMAL  │
│                                                            │
│   "System never crashes"      "Don't over-provision"       │
│                                                            │
│         ↕                                                  │
│   UNDERSTAND YOUR TRAFFIC PATTERN                          │
│                                                            │
│  Each company has:                                         │
│  → Unique use case    → Unique traffic pattern             │
│  → Unique USP         → Unique scaling requirements        │
└────────────────────────────────────────────────────────────┘
```

---

## Traffic Patterns: Netflix vs YouTube vs Hotstar

All three are video streaming platforms — but their system designs are completely different because of different **traffic patterns**.

### Netflix: Predictable Spikes

```
┌──────────────────────────────────────────────────────────┐
│                  NETFLIX TRAFFIC PATTERN                 │
│                                                          │
│  Users                                                   │
│   ▲                        Movie                         │
│   │                       Launch  ┌─────────             │
│   │                          ↓    │         │            │
│   │                          ▲────┘         └────        │
│   │                                                      │
│   └────────────────────────────────────────► Time        │
│                                                          │
│  Traffic is PREDICTABLE because:                         │
│  ✅ Netflix controls what's released                     │
│  ✅ Trailers announce launches weeks in advance          │
│  ✅ Historical data shows spike on release day           │
│                                                          │
│  Strategy: PRE-SCALE before launch                       │
│  → Running 10 servers normally                           │
│  → 24 hrs before movie: spin up 30 servers               │
│  → Pre-cache first 10 mins of movie on all CDN edges     │
│  → If movie flops: scale back down to 10                 │
└──────────────────────────────────────────────────────────┘
```

### YouTube: Unpredictable Spikes

```
┌──────────────────────────────────────────────────────────┐
│                  YOUTUBE TRAFFIC PATTERN                 │
│                                                          │
│  Users  MrBeast    Breaking   Exam    Festival           │
│   ▲     goes live  News       Season  Holiday            │
│   │      ↑↑           ↑↑        ↑       ↓               │
│   │  ────╫──────────╫────────╫─────────╫──              │
│   │      │          │        │                           │
│   └────────────────────────────────────────► Time        │
│                                                          │
│  Traffic is UNPREDICTABLE because:                       │
│  ❌ Anyone can publish at any time                       │
│  ❌ Viral moments can't be forecast                      │
│  ❌ Breaking news triggers instant spikes                │
│                                                          │
│  Strategy: Always keep servers pre-warmed                │
│  → Pay the cost of extra capacity 24/7                   │
│  → ML can predict monthly trends, not minute-by-minute  │
│  → Auto-scaling often too slow for sudden spikes         │
└──────────────────────────────────────────────────────────┘
```

### Hotstar: Cricket — The Complex Case

```
┌──────────────────────────────────────────────────────────────┐
│              HOTSTAR TRAFFIC PATTERN (Cricket)               │
│                                                              │
│  Hotstar has TWO independent services:                       │
│                                                              │
│  ┌───────────────────┐    ┌───────────────────────────┐     │
│  │  Movies/Series    │    │  Live Streaming (Cricket) │     │
│  │  Service          │    │  Service                  │     │
│  └───────────────────┘    └───────────────────────────┘     │
│                                                              │
│  Match Day Traffic Pattern:                                  │
│                                                              │
│  Users ▲                                                     │
│        │  Toss ────────────► SPIKE                           │
│        │     ↓   Boring overs                               │
│        │     │   ↑ Wicket falls → SPIKE                     │
│        │     │   │ Virat bats  → BIG SPIKE                  │
│        │     │   │ Virat out   → SPIKE (everyone leaves)    │
│        └──────────────────────────────────────► Time        │
│                                                              │
│  Strategy:                                                   │
│  ✅ Scale DOWN movies service (nobody watching)             │
│  ✅ Scale UP live stream: 10 → 200 servers, 2hrs before     │
│  ❌ DISABLE auto-scaling during match (too unpredictable)   │
│  ✅ Keep 200 servers for full 4-hour match duration         │
│                                                              │
│  The hidden problem: BACK BUTTON SPIKE                       │
│  → 240 million watching live                                 │
│  → Boring over → everyone hits BACK                         │
│  → All 240M hit the Movies API at once!                     │
│  → Movies service must also be scaled during cricket        │
└──────────────────────────────────────────────────────────────┘
```

---

## Auto Scaling Policies

Scaling doesn't happen magically — it requires defined policies:

```
┌──────────────────────────────────────────────────────────┐
│               AUTO SCALING POLICY EXAMPLES               │
│                                                          │
│  GRADUAL TRAFFIC (works well):                           │
│  Policy: "If avg CPU > 70% for 15 min → add 1 server"   │
│                                                          │
│  Traffic ────────────────────────────────►               │
│  Servers: 3 → 4 → 5 → 6 → 7 → 8  (smooth scaling)      │
│                                                          │
│  SUDDEN SPIKE (auto-scaling fails):                      │
│  Policy: Same as above                                   │
│                                                          │
│  Traffic ──────────────────↑↑↑↑↑↑↑↑↑ (10x spike)        │
│  Servers: 3 → ... calculating avg ... 4  (TOO SLOW!)    │
│                          ↑                               │
│                     System crashes                       │
│                     before new server is ready           │
│                                                          │
│  Why? Scaling policies work on AVERAGES over TIME        │
│  → Sudden spikes happen faster than policies react       │
│  → Solution: Pre-warm servers before expected spikes     │
└──────────────────────────────────────────────────────────┘
```

---

## Serverless Architecture (AWS Lambda)

Skip managing servers entirely — just write code.

```
┌────────────────────────────────────────────────────────────────┐
│                  SERVERLESS (AWS LAMBDA)                       │
│                                                                │
│  Traditional:                       Serverless:               │
│  ┌─────────────────────┐            ┌─────────────────┐       │
│  │ 1. Provision EC2    │            │ 1. Write code   │       │
│  │ 2. Choose OS        │     VS     │ 2. Upload file  │       │
│  │ 3. Set CPU/RAM      │            │ 3. Get URL      │       │
│  │ 4. Install deps     │            │ Done ✅         │       │
│  │ 5. Setup scaling    │            └─────────────────┘       │
│  │ 6. Register in LB   │                                      │
│  └─────────────────────┘                                      │
│                                                                │
│  HOW LAMBDA WORKS:                                             │
│                                                                │
│  Request 1 ──►                                                 │
│  Request 2 ──►  ┌─────────────────────────────────────┐       │
│  Request 3 ──►  │        AWS Lambda Platform           │       │
│       ...       │                                      │       │
│  Request N ──►  │  λ1  λ2  λ3  λ4  λ5  ... λN         │       │
│                 │  (one function instance per request)  │       │
│                 └─────────────────────────────────────┘       │
│                                                                │
│  Traffic rises → more λs spin up automatically                │
│  Traffic falls → λs are destroyed                             │
│  Zero traffic  → Zero λs running → Zero cost                  │
│                                                                │
│  Pricing: First 1 million requests/month FREE                 │
│           Then $0.20 per million requests                      │
└────────────────────────────────────────────────────────────────┘
```

### Lambda: Pros vs Cons

```
┌──────────────────────────────┬──────────────────────────────┐
│            PROS              │            CONS              │
├──────────────────────────────┼──────────────────────────────┤
│ ✅ Pay per request (not/hr)  │ ❌ Cold start latency        │
│ ✅ Auto-scales infinitely    │ ❌ Max 15 sec timeout         │
│ ✅ No server management      │ ❌ Always stateless           │
│ ✅ 1M requests free/month    │ ❌ DDoS = infinite bill       │
│ ✅ No OS/dependencies worry  │ ❌ Vendor lock-in             │
│                              │ ❌ DB connection pool issues  │
└──────────────────────────────┴──────────────────────────────┘
```

### Cold Start Problem

```
┌──────────────────────────────────────────────────────────┐
│                    COLD START                            │
│                                                          │
│  Zero users for 8 hours (night)...                       │
│  → Zero lambdas running                                  │
│                                                          │
│  First request in the morning:                           │
│  → Lambda must be initialized                            │
│  → Code pulled from storage                              │
│  → ~2 second delay for first user                        │
│                                                          │
│  Subsequent requests:                                    │
│  → Lambda is "warm" and ready                            │
│  → ~0.5 second responses                                 │
│                                                          │
│  Not an issue if: you always have at least some traffic  │
└──────────────────────────────────────────────────────────┘
```

### Vendor Lock-In Warning

Once you use Lambda, you naturally start using: SQS, SNS, API Gateway, Route 53, S3, CloudWatch, Step Functions... You become deeply embedded in the AWS ecosystem. Switching later = rewriting everything.

---

## The "Works on My Machine" Problem

Traditional server deployments face a classic problem:

```
┌──────────────────────────────────────────────────────────────┐
│            "IT WORKS ON MY MACHINE" PROBLEM                  │
│                                                              │
│  Developer's Laptop          Production Server               │
│  ┌────────────────┐          ┌────────────────┐              │
│  │ Windows / Mac  │   →      │ Ubuntu Linux   │              │
│  │ FFmpeg v0.4.2  │  Deploy  │ FFmpeg v0.5.1  │              │
│  │ Node v18       │          │ Node v16       │              │
│  │ 30 packages    │          │ Different pkgs │              │
│  │ ✅ Works!      │          │ ❌ Crashes!    │              │
│  └────────────────┘          └────────────────┘              │
│                                                              │
│  Problem: Different OS, different library versions,          │
│  different binary compatibility                              │
│                                                              │
│  Plus: When you need a NEW server (scale-out), you must:     │
│  1. Run: sudo apt-get update                                 │
│  2. sudo apt-get install ffmpeg                              │
│  3. Install all 30 packages                                  │
│  4. Configure environment                                    │
│  5. Start your app                                           │
│  → Takes 5-10 minutes while users are waiting!               │
└──────────────────────────────────────────────────────────────┘
```

---

## Virtualization (VMs)

First attempt to solve the consistency problem:

```
┌──────────────────────────────────────────────────────────────┐
│                  VIRTUAL MACHINES                            │
│                                                              │
│  Physical Machine                                            │
│  ┌─────────────────────────────────────────┐                 │
│  │  Host OS (Ubuntu)                       │                 │
│  │  ┌─────────────────────────────────┐   │                 │
│  │  │    Virtual Machine              │   │                 │
│  │  │  ┌───────────────────────────┐  │   │                 │
│  │  │  │ Guest OS (Ubuntu)  4GB   │  │   │                 │
│  │  │  │ FFmpeg v0.4.2             │  │   │                 │
│  │  │  │ Node v18                  │  │   │                 │
│  │  │  │ Your App                  │  │   │                 │
│  │  │  └───────────────────────────┘  │   │                 │
│  │  └─────────────────────────────────┘   │                 │
│  └─────────────────────────────────────────┘                 │
│                                                              │
│  ✅ "Works on any machine" problem SOLVED                    │
│  ✅ Consistent environment everywhere                        │
│                                                              │
│  ❌ Very heavy: 2 full OS layers running                     │
│  ❌ VM image is 4+ GB                                        │
│  ❌ Boot time: minutes (OS must start)                       │
│  ❌ Wastes CPU/RAM on running two OS kernels                 │
└──────────────────────────────────────────────────────────────┘
```

---

## Containerization (Docker)

Containers = lightweight VMs that share the host OS kernel.

```
┌─────────────────────────────────────────────────────────────────┐
│                   VM vs CONTAINER                               │
│                                                                 │
│   VIRTUAL MACHINE                  CONTAINER                    │
│  ┌─────────────────┐              ┌──────────────────┐          │
│  │ Host OS         │              │ Host OS Kernel   │          │
│  │ ┌─────────────┐ │              │ ┌──────────────┐ │          │
│  │ │ Guest OS    │ │              │ │ App + Deps   │ │          │
│  │ │   4 GB      │ │     VS       │ │   250 MB     │ │          │
│  │ │ App + Deps  │ │              │ └──────────────┘ │          │
│  │ │   250 MB    │ │              │ ┌──────────────┐ │          │
│  │ └─────────────┘ │              │ │ App + Deps   │ │          │
│  └─────────────────┘              │   250 MB       │ │          │
│                                   │ └──────────────┘ │          │
│  Total: ~4.25 GB                  └──────────────────┘          │
│  Boot: minutes                    Total: 250 MB                 │
│  1-2 per physical machine         Boot: milliseconds            │
│                                   16+ per physical machine      │
│                                                                 │
│  KEY INSIGHT: Share the OS kernel, not duplicate it             │
│  → The OS kernel is removed from the container image            │
│  → Container uses host machine's kernel                         │
│  → Only your app + dependencies are packaged                    │
└─────────────────────────────────────────────────────────────────┘
```

### One Physical Machine Running Multiple Containers

```
┌──────────────────────────────────────────────────────────────┐
│         PHYSICAL MACHINE WITH CONTAINERS                     │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  EC2 Machine (e.g., 8GB RAM)                        │    │
│  │                                                     │    │
│  │  C1   C2   C3   C4   C5   C6   C7   C8             │    │
│  │ ┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐          │    │
│  │ │250││250││250││250││250││250││250││250│  MB        │    │
│  │ │MB ││MB ││MB ││MB ││MB ││MB ││MB ││MB │           │    │
│  │ └───┘└───┘└───┘└───┘└───┘└───┘└───┘└───┘          │    │
│  │                                                     │    │
│  │  C9   C10  C11  C12  C13  C14  C15  C16            │    │
│  │ ┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐          │    │
│  │ │250││250││250││250││250││250││250││250│  MB        │    │
│  │ │MB ││MB ││MB ││MB ││MB ││MB ││MB ││MB │           │    │
│  │ └───┘└───┘└───┘└───┘└───┘└───┘└───┘└───┘          │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  Pay for 1 machine → run 16+ isolated services               │
│  One container crashes → others unaffected                   │
│  Machine fills up → spin up 2nd machine + run more          │
└──────────────────────────────────────────────────────────────┘
```

---

## Container Orchestration (Kubernetes)

With 50+ containers across multiple machines, you need a "brain" to manage them.

```
┌────────────────────────────────────────────────────────────────────┐
│              CONTAINER ORCHESTRATION PROBLEM                       │
│                                                                    │
│  3 Machines × 16 containers each = 48 containers running          │
│                                                                    │
│  Questions the "brain" must answer:                                │
│  → Container C12 crashed — restart it automatically?              │
│  → Traffic spiked — create 20 more containers, which machine?     │
│  → Traffic dropped — destroy 15 containers to save cost?          │
│  → New code deployed — swap old containers for new ones?          │
│  → Machine 2 is overloaded — move some containers to Machine 3?   │
│                                                                    │
│  → This problem is called: CONTAINER ORCHESTRATION                 │
└────────────────────────────────────────────────────────────────────┘
```

### The Origin of Kubernetes

```
┌────────────────────────────────────────────────────────────┐
│              HOW KUBERNETES WAS BORN                       │
│                                                            │
│  Google ran containers at massive scale                    │
│  → Built internal tool: BORG (2003-2004)                  │
│  → Borg managed containers across data centers            │
│  → Never open-sourced                                      │
│                                                            │
│  2013: Docker makes containers mainstream                  │
│  → Everyone faces the orchestration problem                │
│                                                            │
│  Google's response:                                        │
│  → Same team that built Borg                               │
│  → Rewrote it from scratch (Go language)                  │
│  → Open-sourced as "Kubernetes" (k8s) in 2014             │
│  → Donated to CNCF (Cloud Native Computing Foundation)    │
│                                                            │
│  Kubernetes = Greek for "Helmsman/Captain"                 │
│  (the one who steers the ship of containers)               │
└────────────────────────────────────────────────────────────┘
```

### Kubernetes Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                  KUBERNETES CLUSTER                             │
│                                                                 │
│  ┌──────────────────────────────────────┐                       │
│  │         CONTROL PLANE (Brain)        │                       │
│  │                                      │                       │
│  │  ┌──────────────┐  ┌─────────────┐  │                       │
│  │  │  API Server  │  │  Scheduler  │  │                       │
│  │  │ (entry point)│  │(assigns pods│  │                       │
│  │  └──────────────┘  │ to nodes)   │  │                       │
│  │  ┌──────────────┐  └─────────────┘  │                       │
│  │  │    etcd      │  ┌─────────────┐  │                       │
│  │  │ (key-value   │  │  Controller │  │                       │
│  │  │  state store)│  │  Manager    │  │                       │
│  │  └──────────────┘  └─────────────┘  │                       │
│  └──────────────────────────────────────┘                       │
│                         │                                        │
│         ┌───────────────┼───────────────┐                       │
│         ▼               ▼               ▼                       │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐            │
│  │  Worker Node │ │  Worker Node │ │  Worker Node │            │
│  │  (Machine 1) │ │  (Machine 2) │ │  (Machine 3) │            │
│  │              │ │              │ │              │            │
│  │ [Pod][Pod]   │ │ [Pod][Pod]   │ │ [Pod][Pod]   │            │
│  │ [Pod][Pod]   │ │ [Pod][Pod]   │ │ [Pod][Pod]   │            │
│  │              │ │              │ │              │            │
│  │  Kubelet     │ │  Kubelet     │ │  Kubelet     │            │
│  │  Kube-proxy  │ │  Kube-proxy  │ │  Kube-proxy  │            │
│  └──────────────┘ └──────────────┘ └──────────────┘            │
│                                                                 │
│  Pod = smallest deployable unit in k8s (1 or more containers)  │
└─────────────────────────────────────────────────────────────────┘
```

### What Kubernetes Manages Automatically

| Feature | What it does |
|---------|-------------|
| **Auto Scaling** | Scales pods up/down based on CPU/memory/custom metrics |
| **Self Healing** | Restarts crashed containers; replaces unhealthy nodes |
| **Load Balancing** | Distributes traffic across pods automatically |
| **Rolling Updates** | Deploys new versions without downtime |
| **Rollback** | Instantly revert to previous version if something breaks |
| **Service Discovery** | Services find each other by name, not IP |
| **Config Management** | Secrets and config maps injected into containers |

---

## Deployment Strategies

```
┌──────────────────────────────────────────────────────────────────┐
│                   ROLLING UPDATE                                 │
│                                                                  │
│  Version 1: [C1][C2][C3][C4][C5][C6]   ← All old containers     │
│                                                                  │
│  Step 1:    [C1][C2][C3][C4][C5][NEW]  ← 1 new added            │
│  Step 2:    [C1][C2][C3][C4][NEW][NEW] ← Old removed            │
│  Step 3:    [C1][C2][C3][NEW][NEW][NEW]                          │
│  ...                                                             │
│  Final:     [NEW][NEW][NEW][NEW][NEW][NEW] ← All updated         │
│                                                                  │
│  → Zero downtime throughout the process                          │
│  → At any point, some old + some new containers serve traffic    │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                 BLUE-GREEN DEPLOYMENT                            │
│                                                                  │
│  BLUE (v1)  [C1][C2][C3]  ← Currently serving all traffic       │
│  GREEN (v2) [C1][C2][C3]  ← New version, fully warmed up        │
│                                                                  │
│  Switch:  Traffic ──────────────► GREEN                          │
│           BLUE stays on standby (instant rollback possible)      │
│                                                                  │
│  → Instant switch, zero downtime                                 │
│  → Instant rollback if something goes wrong                      │
│  → Requires double the infrastructure temporarily                │
└──────────────────────────────────────────────────────────────────┘
```

---

## Summary: Evolution of Deployment

```
┌────────────────────────────────────────────────────────────────────┐
│              DEPLOYMENT EVOLUTION TIMELINE                         │
│                                                                    │
│  TRADITIONAL                                                       │
│  ┌─────────┐                                                       │
│  │ Bare    │ → One app per physical machine                        │
│  │ Metal   │ → "Works on my machine" problem                       │
│  └────┬────┘ → Manual scaling, slow, costly                       │
│       │                                                            │
│       ▼                                                            │
│  VIRTUAL MACHINES                                                  │
│  ┌─────────┐                                                       │
│  │   VM    │ → Consistent environment (solved "my machine")        │
│  │         │ → Still heavy (full OS per VM = 4GB+)                │
│  └────┬────┘ → Slow to start and scale                            │
│       │                                                            │
│       ▼                                                            │
│  CONTAINERS (Docker)                                               │
│  ┌─────────┐                                                       │
│  │Container│ → Lightweight (250MB, shares OS kernel)              │
│  │         │ → Starts in milliseconds                              │
│  └────┬────┘ → Works everywhere consistently                      │
│       │ → Problem: how to manage 50+ containers?                   │
│       ▼                                                            │
│  CONTAINER ORCHESTRATION (Kubernetes)                              │
│  ┌─────────┐                                                       │
│  │   K8s   │ → Auto scale, auto heal, zero-downtime deploys       │
│  │         │ → Manages cluster of machines + containers           │
│  └─────────┘ → Open source, cloud-agnostic                        │
│                                                                    │
│  PARALLEL TRACK: Serverless (Lambda)                               │
│  ┌─────────┐                                                       │
│  │ Lambda  │ → No infra management at all                         │
│  │         │ → Pay per request (very cheap)                        │
│  └─────────┘ → Vendor lock-in risk, cold start, stateless         │
└────────────────────────────────────────────────────────────────────┘
```

---

## Key Takeaways

**On Traffic Patterns:**
- Netflix: predictable → pre-scale before launches
- YouTube: unpredictable → always keep servers pre-warmed
- Hotstar cricket: semi-predictable + hidden correlations (back button spike)
- System design evolves through crashes, monitoring, and optimization

**On Serverless:**
- Great for small to medium scale and event-driven workloads
- Beware vendor lock-in — you adopt more AWS services gradually
- Not ideal for long-running tasks, stateful workloads, or DDoS-prone endpoints

**On Containers:**
- Solves "works on my machine" — package app + dependencies (not OS)
- Much lighter than VMs — pack 16+ per machine vs 1-2 VMs
- The go-to for modern deployable units

**On Kubernetes:**
- Born from Google's internal Borg system
- Open-sourced and donated to CNCF
- The industry-standard brain for container orchestration
- Enables rolling updates, blue-green deploys, zero-downtime releases

---

*Summary based on Piyush Garg's System Design Crash Course — Part 2*

# 📚 System Design: Event Sourcing

> **Source:** [Piyush Garg - System Design: Event Sourcing](https://www.youtube.com/watch?v=JTmgi0vO5Ug)  
> **Topic:** Event Sourcing Pattern — What it is, how it works, and when to use it.

---

## Table of Contents

1. [What is an Event?](#1-what-is-an-event)
2. [Traditional CRUD Approach](#2-traditional-crud-approach)
3. [Problems with Traditional State Management](#3-problems-with-traditional-state-management)
4. [What is Event Sourcing?](#4-what-is-event-sourcing)
5. [Core Concepts](#5-core-concepts)
   - [Event Log (Append-Only Log)](#51-event-log-append-only-log)
   - [Hydration](#52-hydration)
   - [Replay & Audit Trail](#53-replay--audit-trail)
   - [Snapshots / Caching](#54-snapshots--caching)
6. [Event Sourcing in Action — Banking Example](#6-event-sourcing-in-action--banking-example)
7. [Event Sourcing in Action — Video Processing Example](#7-event-sourcing-in-action--video-processing-example)
8. [Out-of-Order Events Problem](#8-out-of-order-events-problem)
9. [Kafka: Consumer Groups & Topic Partitions](#9-kafka-consumer-groups--topic-partitions)
10. [CQRS Pattern (Brief Introduction)](#10-cqrs-pattern-brief-introduction)
11. [Real-World Usage (Uber, Netflix, Amazon)](#11-real-world-usage-uber-netflix-amazon)
12. [Trade-offs & When to Use](#12-trade-offs--when-to-use)
13. [Key Takeaways](#13-key-takeaways)

---

## 1. What is an Event?

> **Definition:** An **event** is a record of something that *happened* in your system. It is immutable — once emitted, it cannot be changed.

In any large-scale system, users perform **actions** on your platform. Each of those actions is an **event**.

### Examples (Amazon):

| Action | Event Name |
|---|---|
| User adds item to cart | `ADD_ITEM_TO_CART` |
| User places an order | `CHECKOUT_ITEM` |
| Seller updates product price | `PRODUCT_PRICE_UPDATED` |
| User uploads a product | `PRODUCT_UPLOADED` |

Events answer the question: **"What happened, when, and to what?"**

---

## 2. Traditional CRUD Approach

In a standard full-stack application, when a user triggers an action, you directly **mutate the database**.

### Flow Diagram:

```
User
 │
 ▼
API Gateway
 │
 ▼
Reverse Proxy (Nginx)
 │
 ▼
App Server
 │
 ▼
Database (PostgreSQL)
  └── UPDATE products SET price = 100 WHERE id = 1;
```

### Example:
A user sends a PATCH request to update Product ID 1's price to ₹100.

```
PATCH /products/1
{ "price": 100 }
```

Your server runs:
```sql
UPDATE products SET price = 100 WHERE id = 1;
```

Now, whenever any user queries this product, they get `price = 100`. ✅

This works fine at small scale. But problems arise at scale.

---

## 3. Problems with Traditional State Management

### Problem 1: Database Bottleneck (Lock Contention)

When you update a row frequently:
- The database **acquires a row-level lock** during each update.
- While the lock is held, **no one can read that row**.
- At high update frequency, this becomes a **bottleneck**.

```
Thread A: UPDATE price = 100  ← acquires lock 🔒
Thread B: SELECT price        ← BLOCKED, waiting...
Thread C: UPDATE price = 110  ← also BLOCKED...
```

This causes **race conditions** and **dirty reads** (reading stale data).

---

### Problem 2: State Sync Failures (The Real Problem)

Consider a **Video Processing Pipeline**:

```
User Uploads Video
       │
       ▼
  S3 (Raw Video Stored)
       │
       ▼
  DB: status = "UPLOADED"
       │
       ▼
  Worker picks up video
       │
       ▼
  DB: status = "PROCESSING"
       │
       ▼
  Worker finishes
       │
       ▼
  S3 (Processed Video Re-uploaded)
       │
       ▼
  DB: status = "SUCCESS" (or "FAILED")
```

**Pseudo-code for this:**
```
1. User uploads video    → DB.update(status = "UPLOADED")
2. Worker picks video    → DB.update(status = "PROCESSING")
3. Worker finishes       → DB.update(status = "SUCCESS")
                           (or DB.update(status = "FAILED"))
```

**What can go wrong?**

| Step | What failed? | Result |
|---|---|---|
| Step 1 succeeds, DB update fails | DB was busy / SQL error | Video uploaded but DB says nothing happened |
| Step 2 succeeds, DB update fails | DB was busy | Video processing but DB still shows "UPLOADED" |
| Step 3 (re-upload) succeeds, DB update fails | DB was busy | Video done but DB forever stuck at "PROCESSING" |

> **Core Issue:** The **real state** (what actually happened) and the **stored state** (what the DB says) go **out of sync**.  
> Once out of sync, **there is no way to track back** what went wrong.

---

## 4. What is Event Sourcing?

> **Definition:** Event Sourcing is a software design pattern where **changes to application state are stored as a sequence of events**, rather than just storing the current state.

Instead of:
```sql
UPDATE videos SET status = 'PROCESSING' WHERE id = 1;
```

You do:
```
Emit Event → { type: "VIDEO_PROCESSING_INIT", videoId: 1, timestamp: "10:05" }
```

**The events ARE your source of truth, not the database rows.**

---

### Analogy: Bank Ledger

A bank doesn't store just your "current balance." It stores **every transaction** (deposit, withdrawal, transfer). Your balance is *derived* by replaying all those transactions.

```
Initial Balance: ₹500
+ Deposit ₹200   → ₹700
+ Deposit ₹300   → ₹1000
- Withdraw ₹500  → ₹500

Current Balance = ₹500  ✅ (derived, not stored directly)
```

This is Event Sourcing.

---

## 5. Core Concepts

### 5.1 Event Log (Append-Only Log)

> **Definition:** An **append-only log** is a data store where you can only **add** new entries. You cannot modify or delete existing entries.

**Why append-only?**

Because events represent **things that already happened** — you cannot un-happen them. The sequence of events must be preserved in chronological order.

```
Event Log (Append-Only):
┌─────────────────────────────────────────────────┐
│ [1] VIDEO_UPLOADED        | videoId:1 | 10:00   │
│ [2] VIDEO_PROCESSING_INIT | videoId:1 | 10:05   │
│ [3] VIDEO_PROCESSING_20%  | videoId:1 | 10:08   │
│ [4] VIDEO_PROCESSING_60%  | videoId:1 | 10:11   │
│ [5] VIDEO_PROCESSING_DONE | videoId:1 | 10:15   │
└─────────────────────────────────────────────────┘
         ↑ Can ONLY add at the bottom. Never edit above.
```

You can store this in:
- **Apache Kafka** (most common for real-time)
- **Amazon S3** (for archival)
- **Event Store DB** (purpose-built)
- **PostgreSQL** (simple append-only table)

---

### 5.2 Hydration

> **Definition:** **Hydration** is the process of **reconstructing the current state** by replaying all events from the beginning (or from a snapshot).

```
Hydration Process:
─────────────────
Event 1: VIDEO_UPLOADED        → state = { status: "UPLOADED" }
Event 2: VIDEO_PROCESSING_INIT → state = { status: "PROCESSING" }
Event 3: VIDEO_PROCESSING_DONE → state = { status: "SUCCESS" }

Final State → { status: "SUCCESS" }  ✅
```

**How Hydration works in code (conceptual):**

```javascript
function hydrateVideoState(events) {
  let state = { status: null };

  for (const event of events) {
    switch (event.type) {
      case "VIDEO_UPLOADED":
        state.status = "UPLOADED";
        break;
      case "VIDEO_PROCESSING_INIT":
        state.status = "PROCESSING";
        break;
      case "VIDEO_PROCESSING_DONE":
        state.status = "SUCCESS";
        break;
      case "VIDEO_PROCESSING_FAILED":
        state.status = "FAILED";
        break;
    }
  }

  return state;
}
```

---

### 5.3 Replay & Audit Trail

**Replay** means re-running all events from the log to:
1. **Audit** — verify what actually happened step by step.
2. **Reconcile** — fix out-of-sync states.
3. **Debug** — find exactly where something went wrong.

**Time Machine feature:** Since you have all events with timestamps, you can answer questions like:

> "What was my bank balance exactly 1 month ago?"

You simply replay only the events that occurred **before** that date.

```javascript
function getBalanceAt(events, targetDate) {
  const relevantEvents = events.filter(e => e.timestamp <= targetDate);
  return hydrateBalance(relevantEvents);
}
```

---

### 5.4 Snapshots / Caching

**Problem:** Replaying thousands of events every time a user queries state is **slow**.

**Solution:** Periodically **snapshot** the derived state into a fast-read database (like PostgreSQL/Redis).

```
Hydration Process
      │
      ▼
Derived State Cached in DB ──→ User queries this DB directly (fast)
      ▲
      │
New Event arrives → Update the cached state incrementally
```

This way:
- **Normal reads** → hit the cache (fast ⚡)
- **Reconciliation / debugging** → replay from raw events (accurate 🎯)

```
┌──────────────────────────────────┐
│         Architecture             │
│                                  │
│  Events ──► Hydration ──► Cache  │
│              (async)       │     │
│                            ▼     │
│                       User Query │
└──────────────────────────────────┘
```

---

## 6. Event Sourcing in Action — Banking Example

### Traditional Approach (Mutable State):

```sql
-- Initial
INSERT INTO users (id, balance) VALUES (1, 500);

-- Deposit 200
UPDATE users SET balance = 700 WHERE id = 1;

-- Deposit 300
UPDATE users SET balance = 1000 WHERE id = 1;

-- Withdraw 500
UPDATE users SET balance = 500 WHERE id = 1;
```

❌ **Problem:** If a user complains "my balance is wrong," you have **no audit trail**. You only see the current value `500`.

---

### Event Sourcing Approach (Immutable Event Log):

```
Event Log:
──────────────────────────────────────────
{ type: "DEPOSIT",  amount: 200, ts: T1 }
{ type: "DEPOSIT",  amount: 300, ts: T2 }
{ type: "WITHDRAW", amount: 500, ts: T3 }
──────────────────────────────────────────
```

**Hydration:**
```
Initial Balance: 500
+ DEPOSIT 200  → 700
+ DEPOSIT 300  → 1000
- WITHDRAW 500 → 500

✅ Final Balance: 500
```

If user disputes, **replay the log** and show them exactly what happened and when. Full transparency.

---

## 7. Event Sourcing in Action — Video Processing Example

### Events Emitted:

```javascript
// Step 1: User uploads video
emitEvent({
  type: "VIDEO_UPLOADED",
  data: { videoId: "abc123", path: "s3://bucket/raw/abc123.mp4" },
  timestamp: new Date()
});

// Step 2: Worker picks up video
emitEvent({
  type: "VIDEO_PROCESSING_INIT",
  data: { videoId: "abc123", workerId: "worker-4" },
  timestamp: new Date()
});

// Step 3 (optional): Progress events
emitEvent({ type: "VIDEO_PROCESSING_PROGRESS", data: { videoId: "abc123", percent: 50 } });

// Step 4a: Success
emitEvent({
  type: "VIDEO_PROCESSING_SUCCESS",
  data: { videoId: "abc123", outputPath: "s3://bucket/processed/abc123" },
  timestamp: new Date()
});

// Step 4b: OR Failure
emitEvent({
  type: "VIDEO_PROCESSING_FAILED",
  data: { videoId: "abc123", error: "codec unsupported" },
  timestamp: new Date()
});
```

### State Machine View:

```
[UPLOADED] ──► [PROCESSING] ──► [SUCCESS]
                    │
                    └──────────► [FAILED]
```

All transitions are recorded as events. You **never lose** which state the video was in and when.

---

## 8. Out-of-Order Events Problem

### The Problem:

With multiple workers consuming from the same event stream, **events can be processed out of order**.

```
Event Stream:
[1] VIDEO_UPLOADED
[2] VIDEO_PROCESSING_INIT
[3] VIDEO_PROCESSING_SUCCESS

Workers:
Worker A (overloaded) → picks event [1]
Worker B              → picks event [2] → updates status to PROCESSING
Worker C              → picks event [3] → updates status to SUCCESS
Worker A finally runs → updates status to UPLOADED  ← ❌ Wrong!

Final State: UPLOADED ← INCORRECT!
```

The user sees "UPLOADED" even though the video is already processed. This is a **dirty read** caused by out-of-order processing.

---

## 9. Kafka: Consumer Groups & Topic Partitions

Kafka solves the out-of-order problem using **Partitions** and **Consumer Groups**.

### Key Idea:

> All events for a **specific object** (e.g., a specific video) must always go to the **same partition**, which is always consumed by the **same worker**.

### How it works:

```
Video A events → Partition 0 → Worker 1 (always)
Video B events → Partition 1 → Worker 2 (always)
Video C events → Partition 2 → Worker 3 (always)
```

**Partition key** = Video ID (or any unique object identifier)

```javascript
kafka.produce({
  topic: "video-events",
  partition: hashFunction(videoId) % totalPartitions,  // deterministic
  message: event
});
```

### Consumer Group Layout:

```
Consumer Group: "video-processors"
┌──────────────────────────────────────┐
│  Worker 1 → Partition 0              │
│  Worker 2 → Partition 1, 4           │
│  Worker 3 → Partition 2, 5           │
│  Worker 4 → Partition 3              │
└──────────────────────────────────────┘
```

**Guarantee:** All events for Video A flow through Worker 1 **in order**. Even if Worker 1 is slow, the events are delayed but **never reordered**.

---

## 10. CQRS Pattern (Brief Introduction)

Event Sourcing is often paired with **CQRS (Command Query Responsibility Segregation)**.

> **Definition:** CQRS separates **write operations (Commands)** from **read operations (Queries)** into different models/paths.

```
┌─────────────────────────────────────────────────┐
│                    CQRS + Event Sourcing         │
│                                                  │
│  Write Side (Command)        Read Side (Query)   │
│  ─────────────────────       ─────────────────   │
│  User Action                 User Query          │
│       │                           │              │
│       ▼                           ▼              │
│  Emit Event ──► Event Log ──► Hydration/Cache    │
│                                   │              │
│                              Return State        │
└─────────────────────────────────────────────────┘
```

- **Commands** → trigger events → stored in event log
- **Queries** → read from the hydrated/cached state

> ⚠️ A full deep-dive into CQRS is a separate topic. The architecture discussed in this video (events + cached state) is essentially CQRS in spirit.

---

## 11. Real-World Usage (Uber, Netflix, Amazon)

### Amazon Order Flow (Pure Event Sourcing):

Every step in an order's lifecycle is an **event**:

```
ORDER_PLACED
    │
    ▼
PAYMENT_CONFIRMED
    │
    ▼
ORDER_DISPATCHED
    │
    ▼
ORDER_SHIPPED
    │
    ▼
ORDER_OUT_FOR_DELIVERY
    │
    ▼
ORDER_DELIVERED  (or ORDER_RETURNED)
```

Amazon does **not** store `status = "DELIVERED"` in a single column. They store the full sequence of events. When you check your order status, the system **replays events** and shows you the latest one.

If you complain "my order wasn't delivered but it shows delivered," they can replay the exact event chain and investigate.

---

### Other Use Cases:

| Domain | Events |
|---|---|
| **Banking** | DEPOSIT, WITHDRAW, TRANSFER, INTEREST_CREDITED |
| **E-commerce** | CART_ITEM_ADDED, CHECKOUT, PAYMENT, SHIPPED |
| **Gaming** | PLAYER_MOVED, ITEM_PICKED, SCORE_UPDATED |
| **Healthcare** | PRESCRIPTION_ISSUED, MEDICATION_TAKEN, TEST_ORDERED |
| **Git (Version Control)** | Each commit is an event — you can replay history! |

---

## 12. Trade-offs & When to Use

### ✅ Advantages of Event Sourcing

| Advantage | Description |
|---|---|
| **Full Audit Trail** | Every state change is recorded with timestamp |
| **Time Travel** | Reconstruct state at any point in history |
| **Debugging** | Replay events to find where things went wrong |
| **Resilience** | Can reconcile state from events even after failures |
| **Decoupling** | Producers emit events; consumers react independently |
| **Analytics** | Rich event history enables powerful analytics |

### ❌ Disadvantages / Trade-offs

| Disadvantage | Description |
|---|---|
| **Complexity** | Much harder to implement than simple CRUD |
| **Hydration cost** | Reading state requires replaying events (mitigated by caching) |
| **Eventual consistency** | State cache may lag behind events briefly |
| **Storage growth** | Append-only means the log grows indefinitely |
| **Hard to reverse** | Migrating from traditional DB to Event Sourcing is painful |

### When to Use Event Sourcing

✅ Use it when:
- You need a **full audit trail** (banking, healthcare, legal)
- State changes are **complex** and come from multiple sources
- You need **time-travel / historical queries**
- You're building **event-driven microservices**
- **Reconciliation** and **debugging** are critical

❌ Avoid it when:
- Simple CRUD is sufficient (personal blog, basic admin panel)
- Your team isn't familiar with the pattern
- You need very **simple read models**

---

## 13. Key Takeaways

```
┌─────────────────────────────────────────────────────────┐
│                  Event Sourcing Summary                  │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  1. Events are the SOURCE OF TRUTH, not DB rows          │
│                                                          │
│  2. Event logs are APPEND-ONLY (immutable, ordered)      │
│                                                          │
│  3. HYDRATION reconstructs state from events             │
│                                                          │
│  4. REPLAY enables audit trails and time travel          │
│                                                          │
│  5. SNAPSHOT/CACHE the derived state for fast reads      │
│                                                          │
│  6. Use KAFKA partitions to guarantee event ordering     │
│     per object (same object → same partition → same      │
│     consumer)                                            │
│                                                          │
│  7. Often paired with CQRS to separate reads & writes    │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

---

### Comparison: Traditional vs Event Sourcing

| Aspect | Traditional CRUD | Event Sourcing |
|---|---|---|
| **Storage** | Current state only | Full event history |
| **Updates** | Overwrite row | Append new event |
| **State read** | Direct DB query | Hydrate from events |
| **Audit trail** | ❌ None (lost on update) | ✅ Full history |
| **Time travel** | ❌ Not possible | ✅ Replay up to any point |
| **Debugging** | ❌ Hard (no history) | ✅ Replay and investigate |
| **Complexity** | Low | High |
| **Performance** | Fast reads | Fast reads (with cache) |
| **Failure recovery** | Manual fix | Reconcile from events |

---

## Further Reading

- [Microsoft Azure — Event Sourcing Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing)
- [AWS — Event Sourcing Pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/modernization-data-persistence/service-per-team.html)
- [Martin Fowler — Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html)
- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)

---

> **Next Video (by Piyush Garg):** CQRS — Command Query Responsibility Segregation

---

*Notes compiled from: [System Design - Event Sourcing by Piyush Garg](https://www.youtube.com/watch?v=JTmgi0vO5Ug)*

# CQRS (Command Query Responsibility Segregation) — Video Summary

> **Video:** CQRS System Design Pattern  
> **Language:** Hindi  
> **Level:** Beginner-Friendly

---

## Table of Contents

1. [The Problem with Traditional Applications](#the-problem-with-traditional-applications)
2. [What is CQRS?](#what-is-cqrs)
3. [How CQRS Works](#how-cqrs-works)
4. [Separate Databases for Each Side](#separate-databases-for-each-side)
5. [Integration with Event Sourcing](#integration-with-event-sourcing)
6. [AWS Architecture Example](#aws-architecture-example)
7. [Key Takeaways](#key-takeaways)
8. [When to Use CQRS](#when-to-use-cqrs)
9. [Notable Quote](#notable-quote)

---

## The Problem with Traditional Applications

In conventional applications, a single database handles all **CRUD operations**:

| Operation | HTTP Method |
|-----------|-------------|
| Create    | POST        |
| Read      | GET         |
| Update    | PUT / PATCH |
| Delete    | DELETE      |

At scale, this becomes a **bottleneck**. For example, when many users are reading a product's price simultaneously while a seller tries to update it, the update acquires a **transaction-level lock** — slowing down all read queries.

> **Real-world analogy:** On Amazon, thousands of users may be reading an earphone's price ($50) at the same moment a seller pushes an update to $30. The write lock acquired during the update blocks all concurrent reads, degrading performance at scale.

**Scaling options in traditional apps:**

- **Vertical Scaling** — Add more CPU/RAM to the same server
- **Horizontal Scaling** — Clone the server and add more instances behind a load balancer

Neither solves the core issue: **reads and writes contend on the same database**.

---

## What is CQRS?

**CQRS = Command Query Responsibility Segregation**

CQRS splits the application into two distinct sides:

| Side        | Responsibility                        | Operations          |
|-------------|---------------------------------------|---------------------|
| **Command** | Mutates data (write side)             | Create, Update, Delete |
| **Query**   | Reads data only (read side)           | Read only           |

> Defined by AWS: *"The CQRS pattern separates data mutation (the Command part) from the Query part of the system."*

---

## How CQRS Works

```
User Request
     │
     ▼
API Gateway
     │
     ├── GET request  ──────────► Query Service  ──► Read DB
     │
     └── POST/PUT/PATCH/DELETE ─► Command Service ──► Write DB
```

An **API Gateway** routes traffic based on HTTP method:
- `GET` → **Query Service** (read path)
- `POST`, `PUT`, `PATCH`, `DELETE` → **Command Service** (write path)

### Command Side Flow

Instead of directly mutating the database, every action generates a **Command object**:

```
ProductUpdateCommand { id: "123", payload: { price: 40 } }
ProductCreateCommand { name: "Earphones", price: 50 }
```

A **Command Handler / Write Model** receives these commands, validates them (e.g., price cannot be negative), checks authorization, and executes the mutation on the Write DB.

### Query Side Flow

- A **Query Handler** receives all read requests
- A **Read Model** is optimized purely for reading
- Data is fetched from the dedicated **Read DB**

---

## Separate Databases for Each Side

CQRS recommends using **different databases** optimized for their respective workloads:

| Side     | Database Type     | Data Format       | Example        |
|----------|-------------------|-------------------|----------------|
| Write DB | SQL (Relational)  | Normalized        | PostgreSQL     |
| Read DB  | NoSQL             | Denormalized JSON | MongoDB        |

### Why Denormalize the Read DB?

In a normalized SQL database, reading a product's full details requires expensive `JOIN` operations across multiple tables (products, orders, inventory, etc.).

In a denormalized Read DB, all related data is pre-joined and stored as a **nested JSON document**. A single query with a `WHERE id = X` returns everything — no joins needed.

### Keeping Databases in Sync

The two databases stay in sync via an **event/message queue**:

```
Write DB ──► Event Emitted ──► Message Queue (Kafka/Kinesis) ──► Read DB Updated
```

This introduces **Eventual Consistency** — a deliberate trade-off:

> A small delay (1–2 seconds) is acceptable compared to the risk of database crashes under heavy load.

**The trade-off:**

| Option | Outcome |
|--------|---------|
| Single DB, fully consistent | Bottleneck → possible downtime at scale |
| Separate DBs, eventual consistency | Scalable, fault-tolerant, slight delay in sync |

---

## Integration with Event Sourcing

CQRS pairs naturally with **Event Sourcing**.

Instead of storing the *final state* in the Write DB, you store an **append-only log** of every event:

```
[ProductCreated]  { id: 1, price: 50, timestamp: T1 }
[PriceUpdated]    { id: 1, price: 40, timestamp: T2 }
[ProductDeleted]  { id: 1,            timestamp: T3 }
```

The Read DB is then a **materialized view** derived by replaying these events (hydration).

### Benefits of Event Sourcing inside CQRS

- Full **audit trail** of every change
- Read DB can be **regenerated** from event logs if corrupted or stale
- Easy **database migration** — just replay events into the new DB
- Enables **fan-out actions** — e.g., a price drop event can simultaneously:
  - Update the Read DB
  - Trigger a promotional email to interested users
  - Invalidate CDN cache

---

## AWS Architecture Example

```
User
 │
 ▼
API Gateway (routes by HTTP method)
 │                        │
 ▼                        ▼
Command ELB           Query ELB
 │                        │
 ▼                        ▼
EC2 Handlers          EC2 Query Handlers
(auth + validation)       │
 │                        ▼
 ▼                    DynamoDB (Read DB)
Kinesis Streams           ▲  ▲
(Kafka equivalent)        │  │
 │                    Lambda (Update Read DB)
 ▼                        │
ClickHouse DB             │
(Append-only Write DB)    │
 │                        │
 ▼                        │
SNS (fan-out)─────────────┘
 │
 ├──► SQS: Update Read DB Queue ──► Lambda ──► DynamoDB
 │
 └──► SQS: Email Queue ──► AWS SES (Simple Email Service)

Optional: CloudFront (CDN) for caching GET responses + cache invalidation Lambda
```

**Component Summary:**

| Component         | Purpose                                          |
|-------------------|--------------------------------------------------|
| API Gateway       | Routes requests by HTTP method                   |
| ELB               | Load balances across EC2 instances               |
| EC2 Handlers      | Authorization, validation, command generation    |
| Kinesis Streams   | Append-only event log (Kafka on AWS)             |
| ClickHouse DB     | Write DB — stores all event logs                 |
| SNS               | Fan-out notifications                            |
| SQS               | Decoupled queues for each downstream action      |
| Lambda            | Serverless processors (update Read DB, email)    |
| DynamoDB          | Read DB — denormalized, fast reads               |
| CloudFront        | CDN caching + cache invalidation                 |

---

## Key Takeaways

- CQRS is **only for complex, high-scale systems** — overkill for basic applications
- The core idea: reads and writes have **different scaling requirements**, so separate them
- **Eventual consistency** is the primary trade-off — not suitable for real-time systems like stock markets
- Event Sourcing fits **naturally** inside CQRS — commands become append-only logs, and the Read DB is a derived materialized view
- The Read DB can always be **regenerated from the event log** if corrupted or stale
- You have the **flexibility to choose different database technologies** for reads vs. writes

---

## When to Use CQRS

Use CQRS when:

- You implement a **database-per-service** pattern and need to join data across multiple microservices
- Your **read and write workloads have separate scaling requirements**
- **Eventual consistency is acceptable** (not for stock markets, financial transactions requiring strong consistency)
- You have **high read-to-write ratios** or vice versa and want to scale each independently

Do **not** use CQRS for:

- Simple CRUD applications
- Small-scale systems
- Systems requiring **strong, real-time consistency**

---

## Notable Quote

> *"If your Read DB ever gets corrupted or stale, you can always regenerate it from the Write DB's event logs — that's the power of combining CQRS with Event Sourcing."*

---

*Summary based on a Hindi tutorial video on CQRS System Design Pattern.*

![crqs image](cqrs.png)