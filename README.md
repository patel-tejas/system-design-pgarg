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


# 📚 System Design: Rate Limiting

> **Source:** [Piyush Garg - Master Rate Limiting](https://www.youtube.com/watch?v=CVItTb_jdkE)  
> **Topic:** Rate Limiting — What it is, algorithms to implement it, and trade-offs of each.

---

## Table of Contents

1. [What is Rate Limiting?](#1-what-is-rate-limiting)
2. [Why Rate Limiting is Necessary](#2-why-rate-limiting-is-necessary)
3. [High-Level Architecture](#3-high-level-architecture)
4. [Rate Limiting Algorithms](#4-rate-limiting-algorithms)
   - [Token Bucket](#41-token-bucket-algorithm)
   - [Leaky Bucket](#42-leaky-bucket-algorithm)
   - [Fixed Window Counter](#43-fixed-window-counter-algorithm)
   - [Sliding Window Log](#44-sliding-window-log-algorithm)
   - [Sliding Window Counter](#45-sliding-window-counter-algorithm-hybrid)
5. [Algorithm Comparison Table](#5-algorithm-comparison-table)
6. [Key Takeaways](#6-key-takeaways)

---

## 1. What is Rate Limiting?

> **Definition:** Rate limiting is a technique used to **control the rate of requests** a client can make to a server within a defined time window.

When a client exceeds the allowed limit, the server rejects excess requests with:

```
HTTP 429 Too Many Requests
```

### Real-World Analogy

Think of a highway toll booth. Only a fixed number of cars can pass per minute. If too many cars arrive at once, extras are turned away and told to wait. The toll booth is your **rate limiter**.

---

## 2. Why Rate Limiting is Necessary

Every server is just a machine with finite resources:

```
Server Specs Example:
┌──────────────────────────┐
│  vCPU:  2                │
│  RAM:   4 GB             │
│  Safe capacity: ~100 req/min │
└──────────────────────────┘
```

| Request Volume | Outcome |
|---|---|
| ≤ 100 req/min | ✅ Safe, server performs well |
| ~150 req/min | ⚠️ Performance degradation starts |
| ≥ 200 req/min | ❌ Server likely crashes |

**Problems without rate limiting:**
- A single bad actor (or buggy client) can send thousands of requests and **take down your server**
- No protection against **DDoS attacks**
- No fairness — one user can starve all others of resources

---

## 3. High-Level Architecture

```
User
 │
 ▼
┌─────────────────────┐
│     Rate Limiter    │  ← checks: are we within threshold?
└─────────────────────┘
       │          │
       │ YES      │ NO
       ▼          ▼
   Server      Drop Request
  (Process)   (Return 429)
       │
       ▼
   Response → User
```

The rate limiter sits **between** the user and your server. It acts as a gatekeeper, only allowing traffic through if the request count is within the configured threshold.

---

## 4. Rate Limiting Algorithms

There are **5 main algorithms** used to implement rate limiting:

1. Token Bucket
2. Leaky Bucket
3. Fixed Window Counter
4. Sliding Window Log
5. Sliding Window Counter (Hybrid)

---

### 4.1 Token Bucket Algorithm

> **Used by:** Amazon, Stripe, and most major API providers.

#### Concept

Imagine a physical bucket that holds tokens (coins). A refiller continuously adds tokens at a fixed rate. Every incoming request must consume one token to be processed. If the bucket is empty, the request is rejected.

#### Visual Diagram

```
            Refiller
          (3 tokens/sec)
               │
               ▼
    ┌─────────────────────┐
    │  🪙 🪙 🪙           │  ← Bucket (capacity: 5)
    │                     │
    └─────────────────────┘
               │
    ┌──────────┴──────────┐
    │                     │
  Token               No Token
 Available?          Available?
    │                     │
    ▼                     ▼
Process Request      Reject (429)
```

#### How It Works — Step by Step

```
Initial State: Bucket has 0 tokens
─────────────────────────────────────
Second 1: Refiller adds 3 tokens → Bucket: [🪙🪙🪙]

Request 1 arrives → consume token → Bucket: [🪙🪙] → ✅ Allowed
Request 2 arrives → consume token → Bucket: [🪙]   → ✅ Allowed
Request 3 arrives → consume token → Bucket: []      → ✅ Allowed
Request 4 arrives → no token!     → Bucket: []      → ❌ Rejected (429)

Second 2: Refiller adds 3 tokens → Bucket: [🪙🪙🪙]
(Repeat cycle...)

Note: If no requests come for 2 seconds:
  Bucket fills to max capacity (5), never exceeds it.
```

#### Parameters

| Parameter | Description | Example |
|---|---|---|
| `bucket_size` | Max tokens the bucket can hold | 5 |
| `refill_rate` | Tokens added per second | 3/sec |

#### Pros & Cons

| ✅ Pros | ❌ Cons |
|---|---|
| Simple to implement | Hard to tune two parameters in production |
| Memory efficient | Burst traffic at boundary moments possible |
| Allows short bursts of traffic | — |
| Used by major companies (Amazon, Stripe) | — |

---

### 4.2 Leaky Bucket Algorithm

#### Concept

Imagine a bucket with a **small hole at the bottom**. No matter how fast water (requests) pours in from the top, it only leaks out at a **fixed, steady rate**. This smooths out bursty traffic into a uniform flow.

#### Visual Diagram

```
Users → Requests pour in from top (any rate)
              │
              ▼
    ┌─────────────────────┐
    │  Req  Req  Req      │  ← Bucket / Queue (capacity: 10,000)
    │  Req  Req           │
    └──────────┬──────────┘
               │  (small hole)
               │  Fixed rate: 5 req/min
               ▼
           Processing
           (Server)
```

#### How It Works

```
Incoming: 20,000 requests/min  ← doesn't matter how fast
Outgoing: 5 requests/min       ← always fixed

If bucket is NOT full → queue the request
If bucket IS full     → reject with 429
```

#### Shower Analogy

> Your overhead tank has 1500 litres. If you open it directly, you'd be crushed by the pressure. Instead, a pipe + showerhead controls the *flow rate* — you get a comfortable, fixed stream. The leaky bucket works the same way.

#### Parameters

| Parameter | Description |
|---|---|
| `bucket_size` | Max queue capacity before rejecting |
| `leak_rate` | Fixed rate at which requests are processed |

#### Pros & Cons

| ✅ Pros | ❌ Cons |
|---|---|
| Smooths out traffic bursts | Burst traffic fills queue with old requests |
| Memory efficient | New (potentially important) requests may never get processed |
| Predictable, stable outflow rate | Two parameters are tricky to tune |

---

### 4.3 Fixed Window Counter Algorithm

#### Concept

Divide the timeline into **fixed-size time windows** (e.g., 1 second each). Count requests in each window. If count exceeds threshold → reject. When the window ends → reset the counter.

#### Visual Diagram

```
Timeline:
─────────────────────────────────────────────────────
|   Window 1  |   Window 2  |   Window 3  |
|   (0s–1s)   |   (1s–2s)   |   (2s–3s)  |
─────────────────────────────────────────────────────

Threshold = 3 req/window

Window 1: req1, req2, req3         → Counter: 3 → ✅ all allowed
          req4 arrives             → Counter: 4 → ❌ rejected

Window 2: RESET counter to 0
          req1, req2               → Counter: 2 → ✅ all allowed

Window 3: RESET counter to 0
          req1, req2, req3, req4   → Counter: 4 → ❌ req4 rejected
```

#### The Critical Bug 🐛

The Fixed Window algorithm has a fundamental flaw: **edge bursts**.

```
Window 1 (0s–1s)       Window 2 (1s–2s)
─────────────────────────────────────────
            │ req req req │ req req req │
            │             │             │
                  ↑
        0.5s ───────────── 1.5s
        This 1-second span also has 6 requests!
        But our limit is 3/sec.
        We just allowed 2× the intended limit.
```

A malicious user can intentionally time their requests at window boundaries to **double the allowed throughput**.

#### Pros & Cons

| ✅ Pros | ❌ Cons |
|---|---|
| Simple to implement | **Edge burst problem** — can exceed limit at window boundaries |
| Easy to understand | Not precise for strict rate enforcement |
| Works fine for lenient rate limiting | — |

---

### 4.4 Sliding Window Log Algorithm

> **Fixes the edge burst problem of Fixed Window Counter.**

#### Concept

Instead of fixed windows with counters, **maintain a log (list) of timestamps** for every request. When a new request arrives, remove all expired timestamps from the log, then check if the log size is within the threshold.

#### Visual Diagram

```
Threshold: 3 requests per 5 seconds
Log: [] (empty initially)

Timeline:
─────────────────────────────────────────────────────
Sec 1: Request arrives
  Log: [1] → size=1 < 3 → ✅ Allowed

Sec 3: Request arrives
  Log: [1, 3] → size=2 < 3 → ✅ Allowed

Sec 4.5: Request arrives
  Log: [1, 3, 4] → size=3 = 3 → ✅ Allowed (at limit)

Sec 6: Request arrives
  → Remove expired (sec 1 is outside 5-sec window from sec 6)
  Log after cleanup: [3, 4]
  Log after adding: [3, 4, 6] → size=3 = 3 → ✅ Allowed

Sec 7: Two requests arrive
  → Remove expired (sec 2 outside window — nothing here)
  Log: [3, 4, 6] → size=3 = 3 already → ❌ Both rejected

Sec 8: Request arrives
  → Remove expired (sec 3 is now outside 5-sec window from sec 8)
  Log after cleanup: [4, 6]
  Log: [4, 6, 8] → size=3 = 3 → ✅ Allowed
```

#### The "Sliding" Part

```
Sliding Window (size = 5 sec):

At t=6:    [1  2  3  4  5  |6]   → 1 slides out
At t=7:    [   2  3  4  5  6  |7] → 2 slides out
At t=8:    [      3  4  5  6  7  |8] → 3 slides out

The window always covers the most recent 5 seconds.
```

#### Pros & Cons

| ✅ Pros | ❌ Cons |
|---|---|
| **Fixes edge burst problem** | Higher memory usage (storing all timestamps) |
| Very accurate rate limiting | Expensive for high-traffic systems |
| No abrupt resets | — |

---

### 4.5 Sliding Window Counter Algorithm (Hybrid)

> **Best of both worlds: Fixed Window Counter + Sliding Window Log**

#### Concept

Keep the **fixed window structure** (for efficiency), but **approximate a sliding window** using the previous window's count weighted by how much of the current window has elapsed.

#### Formula

```
Effective Count = (Previous Window Count × Overlap %) + Current Window Count
```

Where **Overlap %** = how much of the previous window overlaps with the current sliding window.

#### Visual Example

```
Threshold = 3 req/window

Previous Window (2s–3s): 2 requests
Current Window  (3s–4s): starts, new request arrives at t=3.1

Overlap calculation:
  t=3.1 is 10% into the current window
  → Previous window's overlap = 90%
  → Weighted count from previous = 2 × 0.9 = 1.8

  Effective Count = 1.8 + 1 (current) = 2.8 → rounds to 2 or 3
  → Below threshold of 3 → ✅ Allowed
```

#### Another Example (Rejection)

```
Previous Window: 3 requests
Current Window: request arrives at t = 20% into current window

  Overlap = 80%
  Weighted count from previous = 3 × 0.8 = 2.4
  + Current window count = 1
  Effective Count = 3.4 → ❌ Rejected (exceeds threshold of 3)
```

#### Diagram

```
─────────────────────────────────────────────────────
|    Previous Window     |     Current Window        |
|  count = 3             |   new request here        |
|                        |   ← 30% into window →     |
─────────────────────────────────────────────────────
                    ↑
        Previous 70% is still "active"
        Effective = (3 × 0.70) + current_count
```

#### Pros & Cons

| ✅ Pros | ❌ Cons |
|---|---|
| Fixes edge burst problem | Approximate, not 100% precise |
| Memory efficient (just two counters) | Slightly complex logic |
| Better than Fixed Window alone | — |
| Scales well in production | — |

---

## 5. Algorithm Comparison Table

| Algorithm | Memory Usage | Handles Bursts | Fixes Edge Bug | Complexity | Best For |
|---|---|---|---|---|---|
| **Token Bucket** | Low | ✅ Yes | N/A | Low | APIs with burst tolerance (AWS, Stripe) |
| **Leaky Bucket** | Low | ❌ Smooths them | N/A | Low | Stable outflow systems |
| **Fixed Window Counter** | Very Low | ✅ Yes | ❌ No | Very Low | Simple, lenient rate limiting |
| **Sliding Window Log** | High | ❌ Strict | ✅ Yes | Medium | Strict, precise rate limiting |
| **Sliding Window Counter** | Low | ✅ Yes | ✅ Approx | Medium | Production systems needing accuracy + efficiency |

---

## 6. Key Takeaways

```
┌──────────────────────────────────────────────────────────────┐
│                   Rate Limiting Summary                       │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  1. Rate limiting protects servers from overload and abuse    │
│                                                               │
│  2. HTTP 429 = Too Many Requests (the rate limit error code)  │
│                                                               │
│  3. Token Bucket — refillable tokens; allows bursts           │
│                                                               │
│  4. Leaky Bucket — fixed outflow via queue; smooths traffic   │
│                                                               │
│  5. Fixed Window Counter — simple but has edge burst bug      │
│                                                               │
│  6. Sliding Window Log — accurate, high memory cost           │
│                                                               │
│  7. Sliding Window Counter — hybrid, best for production      │
│                                                               │
│  8. Both Token Bucket and Leaky Bucket have 2 parameters      │
│     to tune: capacity & rate. Hard to get right in prod.      │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### Decision Guide

```
Do you need burst tolerance?
        │
       YES → Token Bucket
        │
       NO
        │
Do you need a stable, uniform output rate?
        │
       YES → Leaky Bucket
        │
       NO
        │
Do you need simplicity and approximate accuracy?
        │
       YES → Fixed Window Counter (watch for edge bursts)
        │
       NO
        │
Do you need strict precision (memory not a concern)?
        │
       YES → Sliding Window Log
        │
       NO
        │
       → Sliding Window Counter (best balance for production)
```

---

## Further Reading

- [ByteByteGo — System Design Interview (Chapter 3: Rate Limiting)](https://bytebytego.com)
- [Cloudflare — Rate Limiting Docs](https://developers.cloudflare.com/waf/rate-limiting-rules/)
- [AWS API Gateway — Throttling](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-request-throttling.html)
- [Stripe API Rate Limits](https://stripe.com/docs/rate-limits)

---

> **Previous Video (by Piyush Garg):** Event Sourcing  
> **Next Topic to explore:** CQRS — Command Query Responsibility Segregation

---

*Notes compiled from: [Master Rate Limiting - System Design by Piyush Garg](https://www.youtube.com/watch?v=CVItTb_jdkE)*


# 📚 System Design: Consistent Hashing

> **Source:** [Piyush Garg - Consistent Hashing](https://www.youtube.com/watch?v=IC5Y1EE-aj4)  
> **Topic:** Hashing → Simple Hashing Problems → Consistent Hashing with Ring Mechanism → Virtual Nodes

---

## Table of Contents

1. [The Problem: Horizontal Scaling](#1-the-problem-horizontal-scaling)
2. [What is a Hash Function?](#2-what-is-a-hash-function)
3. [Simple Hashing: How It Works](#3-simple-hashing-how-it-works)
4. [The Critical Problem with Simple Hashing](#4-the-critical-problem-with-simple-hashing)
5. [What is Consistent Hashing?](#5-what-is-consistent-hashing)
6. [The Hash Ring (Ring Mechanism)](#6-the-hash-ring-ring-mechanism)
7. [Adding & Removing Servers in Consistent Hashing](#7-adding--removing-servers-in-consistent-hashing)
8. [The Hotspot Problem & Virtual Nodes](#8-the-hotspot-problem--virtual-nodes)
9. [Benefits Summary](#9-benefits-summary)
10. [Real-World Usage](#10-real-world-usage)
11. [Key Takeaways](#11-key-takeaways)

---

## 1. The Problem: Horizontal Scaling

When your application grows and a single database can't handle the load, you **horizontally scale** — adding more database instances.

```
Before (Single DB):
┌──────────┐
│  DB (1)  │ ← handles 100% of load → BOTTLENECK
└──────────┘

After (Horizontal Scaling):
┌──────────┐  ┌──────────┐  ┌──────────┐
│  DB [0]  │  │  DB [1]  │  │  DB [2]  │
└──────────┘  └──────────┘  └──────────┘
  ~33% load     ~33% load     ~33% load
```

**The new challenge:** With multiple databases, how do you decide *which database* stores or retrieves a particular piece of data? You need a **consistent, deterministic** way to route data.

---

## 2. What is a Hash Function?

> **Definition:** A **hash function** is a deterministic function that takes an input (e.g., user ID) and always returns the same output. It determines which partition/server a piece of data belongs to.

### Key Properties:

| Property | Description |
|---|---|
| **Deterministic** | Same input → always same output |
| **Fixed-size output** | Output is always within a defined range |
| **One-way** | Cannot reverse the output back to the input |
| **Load distribution** | Ideally spreads data evenly across servers |

### Bad Hash Function (Random):
```javascript
function f(userId) {
  return Math.random(); // WRONG — random output every time
}
// Problem: userId=1 might map to DB[2] on insert,
//          but DB[0] on lookup → data never found!
```

### Good Hash Function (Deterministic):
```javascript
function f(userId) {
  return userId % numberOfServers; // CORRECT — same output always
}

// Example with 3 servers:
f(1) = 1 % 3 = 1  → DB[1]
f(2) = 2 % 3 = 2  → DB[2]
f(3) = 3 % 3 = 0  → DB[0]
f(4) = 4 % 3 = 1  → DB[1]
f(5) = 5 % 3 = 2  → DB[2]
```

This hash function also acts as a **load balancer** — distributing data roughly equally across servers.

---

## 3. Simple Hashing: How It Works

With the formula `userId % numberOfServers`:

```
Users:  Piyush(1), John(2), Jane(3), Alex(4), Tiger(5)
Servers: DB[0], DB[1], DB[2]

1 % 3 = 1 → Piyush → DB[1]
2 % 3 = 2 → John   → DB[2]
3 % 3 = 0 → Jane   → DB[0]
4 % 3 = 1 → Alex   → DB[1]
5 % 3 = 2 → Tiger  → DB[2]
```

This works perfectly — as long as the **number of servers never changes**.

---

## 4. The Critical Problem with Simple Hashing

### Scenario: You Add a 4th Server

Your traffic grows. You spin up a new database: `DB[3]`. Now `numberOfServers = 4`.

```javascript
// Hash function changes from:
userId % 3
// to:
userId % 4
```

**Re-calculating where data should live:**

```
1 % 4 = 1 → Piyush → DB[1]  ✅ (was DB[1], OK)
2 % 4 = 2 → John   → DB[2]  ✅ (was DB[2], OK)
3 % 4 = 3 → Jane   → DB[3]  ❌ (was DB[0], WRONG!)
4 % 4 = 0 → Alex   → DB[0]  ❌ (was DB[1], WRONG!)
5 % 4 = 1 → Tiger  → DB[1]  ❌ (was DB[2], WRONG!)
```

**3 out of 5 keys are now mapped to the WRONG server!**

When Jane tries to retrieve her data, the hash function returns `3` (DB[3]), but her data actually lives in `DB[0]` → **data not found!**

### The Scale Problem

```
Real world scenario:
  5 million keys stored across 3 servers
  You add 1 new server

  Result: ~3 million keys need to be physically moved
          between servers → MASSIVE data migration!
```

> **Root Cause:** Changing `numberOfServers` by even 1 reshuffles the majority of keys. This makes adding or removing servers extremely expensive.

---

## 5. What is Consistent Hashing?

> **Definition:** Consistent Hashing is a technique that minimizes the number of keys that need to be remapped when servers are added or removed. Instead of `key % N`, it uses a **ring (circular) structure** where both data and servers are placed on the ring using a hash function.

**The core promise:** When a server is added or removed, only a **small fraction** of keys need to be moved — not the majority.

---

## 6. The Hash Ring (Ring Mechanism)

### Step 1: Create the Ring

Imagine a number line from 0 to some max value (e.g., 0–12), bent into a circle (like a clock face). The hash function maps any input to a number within this range.

```
        12
    11      1
  10          2
  9     ⊙     3      ← Hash Ring (0–12)
  10          4
    7       5
        6
```

### Step 2: Place Servers on the Ring

Each server has a unique identifier (e.g., IP address). Hash the server's identifier → place it at that position on the ring.

```
Server A → hash(A) = 2  → placed at position 2
Server B → hash(B) = 5  → placed at position 5
Server C → hash(C) = 9  → placed at position 9

        12
    11      1
  10          [A=2]
  [C=9]  ⊙   3
  8           4
    7    [B=5]
        6
```

### Step 3: Place Data on the Ring

Hash each data key → place it on the ring → **go clockwise** until you hit the first server → store data there.

```
Data key K → hash(K) = position → go clockwise → first server found

Example:
  hash(K1) = 1  → clockwise → hits A(2)  → stored in Server A
  hash(K2) = 3  → clockwise → hits B(5)  → stored in Server B
  hash(K3) = 6  → clockwise → hits C(9)  → stored in Server C
  hash(K4) = 10 → clockwise → hits A(2 via 12→1→2) → stored in Server A

        12
    11   K1  1
  10          [A=2] ← K1, K2 stored here... wait
  [C=9]  ⊙   K2=3
  K4=10       4
    7    [B=5]
        K3=6
```

**Rule:** Every data key is stored in the **first server found when travelling clockwise** from the key's position.

---

## 7. Adding & Removing Servers in Consistent Hashing

### Adding a New Server

Say you add Server D → `hash(D) = 12`.

```
Before (D added):
  K4 (at 10) → clockwise → Server A (at 2, wrapping around)

After (D added at 12):
  K4 (at 10) → clockwise → Server D (at 12) ← NEW!
```

**Only keys between the previous server (C at 9) and the new server (D at 12) need to move.**

```
Keys that need remapping = keys in range (9 → 12]
Keys NOT affected = everything else ✅
```

```
Finding affected range:
  New server D is at position 12
  Previous server (anti-clockwise) is C at position 9
  Affected keys = those in range (9, 12] → only K4

  All other keys: NO CHANGE ✅
```

### Removing a Server

If Server B (at position 5) goes down:

```
Keys affected = keys that were pointing to B
These keys now travel clockwise to the NEXT server (C at 9)

Only keys in range (2, 5] need remapping.
Everything else: NO CHANGE ✅
```

### Comparison: Simple vs Consistent Hashing

| Operation | Simple Hashing | Consistent Hashing |
|---|---|---|
| Add 1 server (3→4) | ~75% of keys move | Only keys in 1 arc move |
| Remove 1 server (3→2) | ~50% of keys move | Only keys from that arc move |
| Calculate affected keys | Must rehash everything | Simple range: prev_server → new_server |

---

## 8. The Hotspot Problem & Virtual Nodes

### The Problem

Even with consistent hashing, if servers land unevenly on the ring, some servers handle far more data than others.

```
Bad placement (servers too close together):

        12
    11      1
  10          2
  [C=3]       [A=4]    ← A and C are very close!
  8       [B=5]
    7       
        6

Result:
  Arc B→C = tiny (3 to 4) → very few keys → B underloaded
  Arc C→A? No, arc A→B = large (5 to 3 going clockwise) → MANY keys → B overloaded!
```

One server ends up being a **hotspot**, handling the majority of requests. This defeats the purpose of load balancing.

### Solution: Virtual Nodes

> **Definition:** Virtual Nodes (VNodes) are multiple "fake" positions on the ring that all point to the same physical server. Each real server is represented by several positions on the ring, distributing its load more evenly.

```
Physical servers: A, B, C

Virtual node placement:
  A → hash(A-1) = 2,  hash(A-2) = 7,  hash(A-3) = 11
  B → hash(B-1) = 4,  hash(B-2) = 9
  C → hash(C-1) = 1,  hash(C-2) = 6,  hash(C-3) = 10

        12
    [A=11]   [C=1]
  [C=10]       [A=2]
  [B=9]  ⊙   3
  8      [B=4] 4
    [A=7] [C=6]
        5
```

Now data is distributed much more evenly across all three physical servers, even if their primary hash positions would have been uneven.

**How it works at lookup:**
- Data key hashes to position X on the ring
- Go clockwise, hit the first virtual node
- That virtual node is a pointer to the actual physical server
- Route to that physical server

---

## 9. Benefits Summary

```
┌─────────────────────────────────────────────────────────────┐
│              Consistent Hashing Benefits                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. MINIMAL KEY REDISTRIBUTION                               │
│     Only keys in the affected arc move when servers          │
│     are added or removed (not the majority)                  │
│                                                              │
│  2. EASY TO CALCULATE AFFECTED KEYS                          │
│     Affected range = (previous_server, new_server]           │
│     Simple clockwise scan, no full rehash needed             │
│                                                              │
│  3. EASY HORIZONTAL SCALING                                  │
│     Add/remove servers without massive data migrations        │
│                                                              │
│  4. HOTSPOT MITIGATION (with Virtual Nodes)                  │
│     VNodes spread load evenly even with uneven placement      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 10. Real-World Usage

| System | How Consistent Hashing is Used |
|---|---|
| **Amazon DynamoDB** | Partition data across nodes |
| **Apache Cassandra** | Distribute data across cluster |
| **Discord** | Route chat data to servers |
| **CDNs (Akamai, Cloudflare)** | Route requests to edge servers |
| **Redis Cluster** | Shard keys across nodes |

---

## 11. Key Takeaways

```
Simple Hashing:     key % N  (breaks when N changes)
Consistent Hashing: hash ring + clockwise lookup (N-change safe)

Adding a server  → only keys in (prev_node, new_node] move
Removing server  → only keys from that arc move to next server
Uneven ring      → solve with Virtual Nodes
```

### Decision Flow

```
Need to distribute data across multiple servers?
          │
          ▼
    Use a Hash Function
          │
  Will the number of servers change (scale up/down)?
          │
         YES
          │
          ▼
    Use Consistent Hashing (Hash Ring)
          │
    Are some servers getting more load than others?
          │
         YES
          │
          ▼
    Add Virtual Nodes
```

---

## Further Reading

- [ByteByteGo — Consistent Hashing](https://bytebytego.com)
- [Amazon DynamoDB — Partitioning](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.Partitions.html)
- [Apache Cassandra — Data Distribution](https://cassandra.apache.org/doc/latest/cassandra/architecture/dynamo.html)

---

*Notes compiled from: [Consistent Hashing - System Design by Piyush Garg](https://www.youtube.com/watch?v=IC5Y1EE-aj4)*

# 📚 System Design: How Video Streaming Works at Scale

> **Source:** [Piyush Garg - How Video Streaming Works on Scale](https://www.youtube.com/watch?v=-JtjQ-OA7XE)  
> **Topic:** What is a video → Progressive Download → RTMP/RTSP → Adaptive Bitrate Streaming → HLS & MPEG-DASH

---

## Table of Contents

1. [What is a Video?](#1-what-is-a-video)
2. [The Problem: Streaming at Scale](#2-the-problem-streaming-at-scale)
3. [Era 1: Progressive Download](#3-era-1-progressive-download)
4. [Era 2: Real-Time Streaming Protocols (RTMP/RTSP)](#4-era-2-real-time-streaming-protocols-rtmprtsp)
5. [Era 3: Adaptive Bitrate Streaming (ABR)](#5-era-3-adaptive-bitrate-streaming-abr)
6. [HLS: HTTP Live Streaming](#6-hls-http-live-streaming)
7. [How Segments & the M3U8 Index File Work](#7-how-segments--the-m3u8-index-file-work)
8. [Video Processing Pipeline](#8-video-processing-pipeline)
9. [MPEG-DASH vs HLS](#9-mpeg-dash-vs-hls)
10. [Evolution Timeline](#10-evolution-timeline)
11. [Key Takeaways](#11-key-takeaways)

---

## 1. What is a Video?

> **Definition:** A video is a **sequence of images (frames)** played in rapid succession. Your brain perceives this as continuous motion.

```
Frame 1 → Frame 2 → Frame 3 → Frame 4 → Frame 5 ...
  📷          📷         📷         📷         📷
              (played rapidly = motion perceived)
```

### Frame Rate (FPS — Frames Per Second)

| FPS | Experience |
|---|---|
| 1 fps | Very choppy (slideshow-like) |
| 24 fps | Cinema standard |
| 30 fps | Smooth TV standard |
| 60 fps | Very smooth (gaming/sports) |
| 120+ fps | Ultra smooth |

The higher the FPS, the more images per second → smoother experience → but also **larger file size**.

### Why Video Files Are Large

A 1-hour video at 60 fps = 60 × 3600 = **216,000 individual frames**.  
Even a "small" modern video can easily be **4–5 GB** after compression.

Formats like **MP4** store these frames in a compressed sequence that computers can decode and play back.

---

## 2. The Problem: Streaming at Scale

The central engineering challenge:

> How do you deliver a 4–5 GB video file to millions of users across different devices, screen sizes, and network speeds — smoothly, without buffering?

### Devices Today

```
4K LED TV         → needs high resolution, fast connection
Laptop            → medium screen, decent connection
Mobile Phone      → small screen, varies from WiFi to 3G
Smartwatch        → tiny screen, low bandwidth
Smart Fridge      → ???
```

**One video quality does NOT fit all devices.** A 4K stream is wasteful on a smartwatch. A tiny bitrate looks terrible on a 65" TV.

---

## 3. Era 1: Progressive Download

The earliest approach to video delivery on the internet.

### How it worked

```
Client                     Server
  │                           │
  │── GET /video.mp4 ─────────►│
  │                           │
  │◄── (download entire file)─│
  │                           │
  │  [wait... wait... wait]   │
  │  100MB downloading...     │
  │                           │
  │  [finally] play video     │
```

**You had to download the entire video file before you could watch it.**

### Why it was bad

```
Modern video: 4–5 GB
Internet speed: 200 Mbps

Download time before playback starts = (4 GB × 8) / 200 Mbps
                                     ≈ 160 seconds ≈ 2.7 minutes of waiting

Plus: What if you watch 10 seconds and close the tab?
      You wasted bandwidth downloading GBs you never watched.
```

This is where **buffering** became a pain point — you literally had to wait for the buffer to fill before you could watch.

---

## 4. Era 2: Real-Time Streaming Protocols (RTMP/RTSP)

### What changed

Two protocols emerged to solve progressive download's problems:

| Protocol | Full Name | Creator |
|---|---|---|
| **RTMP** | Real-Time Messaging Protocol | Adobe |
| **RTSP** | Real-Time Streaming Protocol | RealNetworks |

### What they introduced

```
Before (Progressive Download):
  Server → [Send entire file] → Client (play only after full download)

After (RTMP/RTSP):
  Server → [Send chunk 1] → Client plays chunk 1
           [Send chunk 2] → Client plays chunk 2
           [Send chunk 3] → Client plays chunk 3
           ...and so on
```

### Benefits

| Feature | Description |
|---|---|
| **Low latency** | Start playing almost immediately |
| **Live streaming** | Stream in real time, not just pre-recorded |
| **Efficient bandwidth** | Only download what you're actually watching |

### The Remaining Problem

Even though streaming in chunks was possible, the video was still encoded in **one single quality** (e.g., 4K).

```
Problem scenario:
  Video: 4K (one quality)
  User A: 65" TV, 300 Mbps → perfect ✅
  User B: Mobile, 10 Mbps  → each 4K chunk is too large, buffering ❌
  User C: Smartwatch        → why is this even 4K?? ❌

There was no way to adapt quality based on the client's capabilities.
```

---

## 5. Era 3: Adaptive Bitrate Streaming (ABR)

> **Definition:** Adaptive Bitrate Streaming (ABR) is a technique where the video is encoded at **multiple quality levels**, and the client **dynamically selects** the best quality based on its current network speed, device screen size, and available bandwidth.

### The Core Idea

```
Instead of:
  One 4K video → served to everyone

ABR does:
  Same source video encoded into multiple qualities:
    480p  (low quality, small file)
    720p  (medium quality)
    1080p (HD)
    4K    (ultra HD, large file)

Client decides which quality to use based on:
  - Current network speed
  - Screen resolution
  - Device capabilities
```

### Why "Adaptive"?

The quality **adapts in real time**. If you're watching 1080p and suddenly move to a weak network area:
- The player detects lower bandwidth
- Switches to 480p automatically
- No buffering — just slightly reduced quality
- When you return to strong network → switches back to 1080p

```
Network:  ████████░░░░████████
Quality:  1080p → 480p → 1080p
Experience: smooth → slightly blurry → smooth
```

This is dramatically better than the old alternative: high quality that freezes and buffers.

---

## 6. HLS: HTTP Live Streaming

> **Full Name:** HTTP Live Streaming  
> **Created by:** Apple  
> **File extension:** `.m3u8`

HLS is the most widely used ABR protocol today. It works over regular HTTP — meaning it uses the same infrastructure as websites, no special servers needed.

### How HLS Works

```
                    ┌──────────────────────────────────────┐
                    │         Video Processing Server       │
                    │                                      │
Source Video (4K) → │  Encode into:                        │
                    │    480p  segments (seg1, seg2, seg3...) │
                    │    720p  segments                    │
                    │    1080p segments                    │
                    │    4K    segments                    │
                    │                                      │
                    │  + Generate M3U8 index file          │
                    └──────────────────────────────────────┘
                                      │
                                      ▼
                            Stored on CDN / Server
```

---

## 7. How Segments & the M3U8 Index File Work

### What are Segments?

The source video is **cut into small time-based chunks** called segments (typically 2–10 seconds each).

```
Full video (60 min) at 1080p:
  → seg_1080_001.ts  (seconds 0–6)
  → seg_1080_002.ts  (seconds 6–12)
  → seg_1080_003.ts  (seconds 12–18)
  ...and so on for every quality level
```

### The M3U8 Index File (Manifest / Playlist)

This is a plain text file that acts like a **table of contents** for all the segments.

```
master.m3u8 (Master Playlist):
─────────────────────────────────────────────
#EXTM3U
#EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=640x480
/streams/480p/playlist.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=1400000,RESOLUTION=1280x720
/streams/720p/playlist.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=2800000,RESOLUTION=1920x1080
/streams/1080p/playlist.m3u8
─────────────────────────────────────────────
```

The client reads this master file first, picks the best quality, then reads that quality's individual playlist.

```
480p/playlist.m3u8 (Quality-specific playlist):
─────────────────────────────────────────────
#EXTM3U
#EXT-X-TARGETDURATION:6
#EXTINF:6.0
seg_480_001.ts
#EXTINF:6.0
seg_480_002.ts
#EXTINF:6.0
seg_480_003.ts
...
─────────────────────────────────────────────
```

### The Full Client Flow

```
Client Request Flow:
─────────────────────────────────────────────
Step 1: Client fetches master.m3u8
          → sees available qualities: 480p, 720p, 1080p, 4K

Step 2: Client evaluates itself:
          - Screen resolution: 1920×1080
          - Network speed: 280 Mbps
          → Decision: I'll use 1080p

Step 3: Client fetches 1080p/playlist.m3u8
          → Gets list of segment URLs

Step 4: Client downloads and plays segments one by one:
          GET seg_1080_001.ts → play → GET seg_1080_002.ts → play...

Step 5: Client continuously monitors network:
          - Speed drops → switch to 720p/playlist.m3u8
          - Speed recovers → switch back to 1080p/playlist.m3u8
─────────────────────────────────────────────
```

---

## 8. Video Processing Pipeline

When a creator uploads a video (e.g., to YouTube), it doesn't appear instantly. It goes through a **processing pipeline**.

```
Creator uploads raw video (4K MP4)
              │
              ▼
┌─────────────────────────────────┐
│     Video Processing Pipeline   │
│                                 │
│  ffmpeg (or similar tool):      │
│  ┌────────────────────────────┐ │
│  │ Encode → 480p segments     │ │
│  │ Encode → 720p segments     │ │
│  │ Encode → 1080p segments    │ │
│  │ Encode → 4K segments       │ │
│  │ Generate M3U8 files        │ │
│  └────────────────────────────┘ │
└─────────────────────────────────┘
              │
              ▼
    Store on CDN / Object Storage
              │
              ▼
         Video is LIVE ✅
```

This is why YouTube videos take **20–30 minutes to process** after upload before being publicly available.

### Tools for Encoding

- **FFmpeg** — most common open-source video encoder
- **AWS MediaConvert** — managed cloud encoding
- **ImageKit** — managed platform handling encoding + HLS delivery
- **Mux, Cloudflare Stream** — managed video hosting with ABR out of the box

### Infrastructure Requirements

Building this yourself requires:

```
1. Storage (S3, GCS) — store all segments (can be 3–4× the original size)
2. Processing servers — CPU/GPU intensive encoding jobs
3. CDN — deliver segments fast, close to the user
4. Pipeline orchestration — queue, retry, monitor jobs
5. Maintenance — significant engineering ongoing effort
```

> Most startups use managed solutions like **ImageKit, Mux, or Cloudflare Stream** rather than building this from scratch.

---

## 9. MPEG-DASH vs HLS

Both are ABR protocols. They work similarly but have different origins and file formats.

| Feature | HLS | MPEG-DASH |
|---|---|---|
| **Full Name** | HTTP Live Streaming | Dynamic Adaptive Streaming over HTTP |
| **Created by** | Apple | MPEG consortium (open standard) |
| **Index file** | `.m3u8` | `.mpd` (Media Presentation Description) |
| **Segment format** | `.ts` (MPEG-2 TS) | `.mp4` fragments |
| **Device support** | Excellent on Apple, wide support | Strong on Android/web |
| **Usage today** | Most common on iOS/Safari | Common on Android/Chrome |

```
URL difference:
  HLS:  https://cdn.example.com/video/master.m3u8
  DASH: https://cdn.example.com/video/manifest.mpd
```

Both achieve the same goal. Today, most platforms support both.

---

## 10. Evolution Timeline

```
Video Delivery Evolution:
──────────────────────────────────────────────────────
Early 2000s: Progressive Download
  → Download full file first, then play
  → Massive buffering issues
  → Bandwidth wasteful

Mid 2000s: RTMP / RTSP (Specialized Streaming Protocols)
  → Stream in chunks
  → Low latency
  → Live streaming possible
  → BUT: single quality, no adaptation

2009+: Adaptive Bitrate Streaming (HLS by Apple)
  → Multiple quality levels pre-encoded
  → Client selects quality dynamically
  → Adapts to network conditions in real-time
  → Works over standard HTTP (no special infra)

Today: HLS + MPEG-DASH
  → Standard for all major platforms
  → CDN-accelerated delivery
  → Client-side quality algorithms (very sophisticated)
  → 4K, HDR, Dolby Atmos all delivered via ABR
──────────────────────────────────────────────────────
```

---

## 11. Key Takeaways

```
┌──────────────────────────────────────────────────────────────┐
│                  Video Streaming Summary                      │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  1. Video = sequence of frames (images) played rapidly        │
│                                                               │
│  2. Progressive Download = full file before play (old, bad)  │
│                                                               │
│  3. RTMP/RTSP = chunked streaming, but one quality only       │
│                                                               │
│  4. ABR = multiple quality encodings + client selects best    │
│                                                               │
│  5. HLS = Apple's ABR protocol using .m3u8 index files        │
│                                                               │
│  6. Segments = small time chunks of video at each quality     │
│                                                               │
│  7. M3U8 = index/manifest file pointing to all segments       │
│                                                               │
│  8. Client dynamically switches quality based on network      │
│                                                               │
│  9. Processing pipeline converts raw upload → multi-quality   │
│     segments (this is why YouTube takes time to process)      │
│                                                               │
│  10. Building your own pipeline is costly → use managed       │
│      services (ImageKit, Mux, Cloudflare Stream)              │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## Further Reading

- [Apple HLS Documentation](https://developer.apple.com/streaming/)
- [MPEG-DASH Industry Forum](https://dashif.org/)
- [ImageKit — Adaptive Bitrate Streaming Guide](https://imagekit.io/blog/adaptive-bitrate-streaming/)
- [FFmpeg Documentation](https://ffmpeg.org/documentation.html)

---

*Notes compiled from: [How Video Streaming Works on Scale by Piyush Garg](https://www.youtube.com/watch?v=-JtjQ-OA7XE)*


# 📚 System Design: How UPI Payments Work

> **Source:** [Piyush Garg - System Design of UPI Payments](https://www.youtube.com/watch?v=fqySz1Me2pI)  
> **Topic:** Traditional Banking → IMPS/NEFT/RTGS → UPI Architecture → NPCI → Transaction Flow

---

## Table of Contents

1. [Traditional Banking & Money Transfer](#1-traditional-banking--money-transfer)
2. [Legacy Transfer Protocols](#2-legacy-transfer-protocols)
3. [What is UPI?](#3-what-is-upi)
4. [Core Components of UPI](#4-core-components-of-upi)
   - [NPCI](#41-npci---the-backbone)
   - [VPA (Virtual Payment Address)](#42-vpa---virtual-payment-address)
   - [PSP (Payment Service Provider)](#43-psp---payment-service-provider)
5. [Complete UPI Transaction Flow](#5-complete-upi-transaction-flow)
6. [Push vs Pull Mechanism](#6-push-vs-pull-mechanism)
7. [What Happens When a Transaction Fails?](#7-what-happens-when-a-transaction-fails)
8. [Participant Map](#8-participant-map)
9. [Key Takeaways](#9-key-takeaways)

---

## 1. Traditional Banking & Money Transfer

Before UPI, to transfer money digitally you needed all of these details about the recipient:

```
Required Information (Traditional Bank Transfer):
┌─────────────────────────────────────────────┐
│  Account Number  : 123456789012             │
│  Bank Name       : ICICI Bank               │
│  Branch Code     : 001                      │
│  IFSC Code       : ICIC0000001              │
│  Amount          : ₹1,000                   │
└─────────────────────────────────────────────┘
```

**IFSC Code** (Indian Financial System Code) uniquely identifies:
- Which bank the account belongs to
- Which specific branch of that bank

Without any one of these, the transaction would fail or be reversed.

---

## 2. Legacy Transfer Protocols

India had multiple transfer systems before UPI, each for different use cases:

| Protocol | Full Name | Speed | Best For | Limit |
|---|---|---|---|---|
| **IMPS** | Immediate Payment Service | Instant (seconds) | Small transfers | Up to ₹5 lakh |
| **NEFT** | National Electronic Funds Transfer | 2–3 hours | Medium transfers | No fixed limit |
| **RTGS** | Real Time Gross Settlement | Instant | Large transfers | Min ₹2 lakh |

```
Timeline of a NEFT transfer:
  12:00 PM - You initiate transfer
  12:00 PM - Bank processes in next batch
   2:30 PM - Recipient receives the money

Contrast with UPI:
  12:00 PM - You initiate transfer
  12:00 PM - Money received ← INSTANT
```

**Problems with these systems:**
- Required complex account details (IFSC, account number)
- Not always real-time
- Poor UX for everyday consumers
- No unified app experience across banks

---

## 3. What is UPI?

> **UPI** = **Unified Payment Interface**

UPI was built to make digital payments **as simple as knowing someone's ID** — no account numbers, no IFSC codes, no branch codes.

```
Traditional Transfer:
  Account No: 123456789012
  IFSC: ICIC0000001
  Branch: Connaught Place, Delhi
  → Complex, error-prone, hard to remember

UPI Transfer:
  piyush@icici
  → Simple, memorable, shareable
```

UPI was built and is operated by **NPCI** (National Payments Corporation of India) under the regulation of **RBI** (Reserve Bank of India).

---

## 4. Core Components of UPI

### 4.1 NPCI - The Backbone

> **NPCI** = National Payments Corporation of India

NPCI is the **infrastructure** that powers UPI. It is not a payment gateway or a consumer app — it's the backend network that all banks and payment apps plug into.

```
NPCI Architecture:
┌─────────────────────────────────────────────┐
│                  NPCI                        │
│         (Closed, Trusted Network)           │
│                                              │
│  Only trusted banks can connect:            │
│    ✅ ICICI Bank                             │
│    ✅ HDFC Bank                              │
│    ✅ SBI                                    │
│    ✅ Yes Bank                               │
│    ✅ Axis Bank                              │
│    ✅ ...all major banks                     │
│                                              │
│  These CANNOT connect directly:             │
│    ❌ PhonePe                                │
│    ❌ Google Pay                             │
│    ❌ Any public app or user                 │
└─────────────────────────────────────────────┘
```

**Key facts about NPCI:**
- APIs are **not publicly available**
- Only RBI-regulated banks can talk to NPCI
- It operates on a **secure, closed network**
- Consumer apps (PhonePe, GPay) must **partner with a bank** to enter this network

---

### 4.2 VPA - Virtual Payment Address

> **VPA** = Virtual Payment Address

The VPA is the **human-readable ID** that replaces all the complex bank details.

```
Format:   username@handle

Examples:
  piyush@icici
  john123@okhdfcbank
  myshop@paytm
  9876543210@ybl   (phone number as username, Yes Bank handle)
```

**When you scan a QR code**, you're not scanning anything fancy — the QR code simply encodes a VPA. It's just a lazy, error-free way to enter a VPA without typing.

```
QR Code = encoded VPA string
Scanning QR code = automatically filling in the VPA field
```

### Common UPI Handles by Bank/App:

| Handle | Linked To |
|---|---|
| `@icici` | ICICI Bank |
| `@okhdfcbank` | HDFC via Google Pay |
| `@ybl` | Yes Bank (via PhonePe) |
| `@paytm` | Paytm Payments Bank |
| `@okaxis` | Axis Bank via Google Pay |

---

### 4.3 PSP - Payment Service Provider

> **PSP** = Payment Service Provider (also called Customer PSP or Third-Party App)

PSPs are the **consumer-facing apps** like PhonePe, Google Pay, and Paytm. They:
1. Provide the **UI/UX** for users to initiate payments
2. **Partner with a bank** (their PSP bank) to get access to NPCI's network
3. Forward payment requests from users to their partner bank

```
Consumer App → PSP Bank → NPCI Network

PhonePe     → Yes Bank  → NPCI
Google Pay  → ICICI / Axis / HDFC / SBI → NPCI
Paytm       → Paytm Payments Bank → NPCI
Amazon Pay  → Axis Bank → NPCI
```

```
Architecture Layer Diagram:
┌──────────────────────────────────────────────────────┐
│  Layer 3: Consumer Apps (PSPs)                       │
│  PhonePe | Google Pay | Paytm | Amazon Pay           │
│  (UI + UX layer, handles user interaction)           │
└────────────────────┬─────────────────────────────────┘
                     │ (partners with)
┌────────────────────▼─────────────────────────────────┐
│  Layer 2: PSP Banks                                  │
│  Yes Bank | ICICI | Axis | HDFC | SBI                │
│  (trusted members of NPCI network)                   │
└────────────────────┬─────────────────────────────────┘
                     │ (connects to)
┌────────────────────▼─────────────────────────────────┐
│  Layer 1: NPCI                                       │
│  Core UPI Infrastructure                            │
│  Routes transactions between banks                  │
└──────────────────────────────────────────────────────┘
```

---

## 5. Complete UPI Transaction Flow

### Scenario
**User A** (PhonePe, Yes Bank) wants to send ₹1,000 to **Piyush** (Google Pay, ICICI Bank).

```
User A's VPA: userA@ybl      (Yes Bank handle)
Piyush's VPA: piyush@icici   (ICICI handle)
Amount: ₹1,000
```

### Step-by-Step Flow

```
Step 1: Create Payment Intent
─────────────────────────────
User A opens PhonePe
Enters/scans Piyush's VPA: piyush@icici
Enters amount: ₹1,000

PhonePe creates a payment intent:
{
  from: "userA@ybl",
  to:   "piyush@icici",
  amount: 1000
}

Step 2: PSP → PSP Bank
─────────────────────────────
PhonePe (User A's PSP) cannot talk to NPCI directly.
PhonePe sends this intent to its partner bank: Yes Bank

Step 3: PSP Bank → NPCI
─────────────────────────────
Yes Bank (trusted NPCI member) puts the request
into the NPCI network.

NPCI accepts it because: it came from a trusted bank ✅

Step 4: NPCI Verifies Balance
─────────────────────────────
NPCI sees: "userA@ybl wants to send ₹1,000"
NPCI asks Yes Bank: "Does userA have ₹1,000?"
Yes Bank: "Yes, balance is available" ✅

Step 5: Authentication Request
─────────────────────────────
Yes Bank sends back to PhonePe:
"Transaction looks good. Need UPI PIN authentication."

PhonePe shows the UPI PIN screen to User A.
User A enters UPI PIN.
PhonePe sends the PIN back to Yes Bank.

Step 6: Debit the Sender
─────────────────────────────
NPCI instructs Yes Bank:
"Debit ₹1,000 from userA's account"

Yes Bank debits ₹1,000 from User A's account ✅
Yes Bank acknowledges to NPCI: "Debit successful"

Step 7: Credit the Receiver
─────────────────────────────
NPCI looks up "piyush@icici"
NPCI identifies: Piyush's bank is ICICI

NPCI instructs ICICI Bank:
"Credit ₹1,000 to Piyush's account"

ICICI Bank credits ₹1,000 to Piyush's account ✅
ICICI Bank acknowledges to NPCI: "Credit successful"

Step 8: Notification
─────────────────────────────
Both acknowledgements received by NPCI → 
Transaction marked as COMPLETE ✅

ICICI Bank pushes notification to Google Pay (Piyush's PSP):
"₹1,000 received from userA@ybl"

Piyush sees on Google Pay: "Received ₹1,000 from User A" 🎉
```

### Visual Flow Diagram

```
User A                PhonePe       Yes Bank      NPCI       ICICI      Piyush
(Sender)              (PSP)         (PSP Bank)             (Bank)     (Google Pay)
   │                     │               │           │          │           │
   │──intent──────────►  │               │           │          │           │
   │                     │──request────► │           │          │           │
   │                     │               │──put──────►│          │           │
   │                     │               │           │──balance?►│           │
   │                     │               │◄──ok───── │          │           │
   │◄──PIN screen──────  │               │           │          │           │
   │──PIN──────────────► │               │           │          │           │
   │                     │──PIN───────►  │           │          │           │
   │                     │               │──debit────►│          │           │
   │                     │               │◄──ack───── │          │           │
   │                     │               │           │──credit──►│           │
   │                     │               │           │          │──notify──►│
   │                     │               │           │          │           │
   │                     ◄── COMPLETE ───────────────────────── │           │
```

---

## 6. Push vs Pull Mechanism

UPI supports two types of transactions:

### Push Transaction (Most Common)
> You **send** money to someone.

```
You → initiate → transfer → Recipient gets money
Example: Paying a shopkeeper
```

### Pull Transaction
> You **request** money from someone.

```
Recipient → sends payment request → You approve → Money sent
Example:
  - "Request Money" feature in apps
  - Online shopping checkout (merchant pulls payment from you)
  - Splitting bills
```

```
Pull Flow:
NPCI receives pull request from merchant
NPCI notifies your UPI app: "Merchant X wants ₹500"
You approve → same debit/credit flow as push
```

---

## 7. What Happens When a Transaction Fails?

### Scenario: ICICI's server was busy, couldn't acknowledge

```
NPCI sends credit request to ICICI ─────────────────────►
ICICI server was busy ── cannot acknowledge ─────────────

NPCI waits... timeout...
NPCI: "No acknowledgement received from ICICI"
NPCI: "Transaction must be rolled back"

NPCI instructs Yes Bank: "Reverse the debit — credit ₹1,000 back to User A"
Yes Bank credits ₹1,000 back to User A ✅

User A sees: "Transaction failed, ₹1,000 refunded"
```

**This is why money sometimes shows as "debited" but then returns automatically** — the credit acknowledgement was never received, so the transaction was rolled back.

```
Two acknowledgements required for a COMPLETE transaction:
  1. Debit ACK from sender's bank  ✅
  2. Credit ACK from receiver's bank ✅

If EITHER fails → transaction is reversed → money returned
```

---

## 8. Participant Map

```
Full UPI Ecosystem:

┌──────────────────────────────────────────────────────────┐
│                    RBI (Regulator)                       │
│               Regulates all participants                 │
└───────────────────────────┬──────────────────────────────┘
                            │
┌───────────────────────────▼──────────────────────────────┐
│                      NPCI                                │
│              Core UPI Infrastructure                     │
│    Routes payments, validates, settles transactions      │
└──┬────────────────────────────────────────────────────┬──┘
   │                                                    │
   ▼                                                    ▼
Issuing Bank                                    Acquiring Bank
(Sender's Bank)                                 (Receiver's Bank)
e.g. Yes Bank                                   e.g. ICICI Bank
   │                                                    │
   ▼                                                    ▼
PSP Bank for                                    PSP Bank for
Sender's App                                    Receiver's App
   │                                                    │
   ▼                                                    ▼
Payer PSP                                       Payee PSP
(PhonePe)                                       (Google Pay)
   │                                                    │
   ▼                                                    ▼
Payer (User A)                                  Payee (Piyush)
```

### Google Pay's Partner Banks:
`Axis Bank` | `ICICI Bank` | `HDFC Bank` | `SBI`

### PhonePe's Partner Banks:
`Yes Bank` | `ICICI Bank` | `Axis Bank`

### Amazon Pay's Partner Banks:
`Axis Bank`

---

## 9. Key Takeaways

```
┌──────────────────────────────────────────────────────────────┐
│                    UPI Summary                               │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  1. VPA replaces complex bank details                         │
│     (username@handle instead of account + IFSC)              │
│                                                               │
│  2. NPCI is the closed backbone — only banks can access it    │
│                                                               │
│  3. Consumer apps (PhonePe/GPay) partner with banks to        │
│     enter the NPCI network                                    │
│                                                               │
│  4. QR code = just a VPA encoded for easy scanning           │
│                                                               │
│  5. Transaction requires TWO acks: debit + credit            │
│     If either fails → automatic rollback/refund              │
│                                                               │
│  6. UPI supports Push (send) and Pull (request) flows         │
│                                                               │
│  7. NPCI handles billions of transactions — incredibly        │
│     fault-tolerant, real-time engineering                     │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### Comparison: Old vs UPI

| Feature | Traditional | UPI |
|---|---|---|
| **Recipient ID needed** | Account no + IFSC + branch | Just a VPA (e.g., name@bank) |
| **Speed** | Hours (NEFT) or instant (IMPS) | Always instant |
| **UX** | Complex, bank-specific | Unified, any app |
| **Interoperability** | Limited | Works across all banks |
| **Live transfers** | IMPS only | All UPI transfers |

---

## Further Reading

- [RazorPay — What is UPI and How it Works](https://razorpay.com/learn/what-is-upi/)
- [NPCI Official UPI Page](https://www.npci.org.in/what-we-do/upi/product-overview)
- [Google Pay — UPI Transaction Responsibilities](https://pay.google.com/intl/en_in/about/policy/)

---

*Notes compiled from: [System Design of UPI Payments by Piyush Garg](https://www.youtube.com/watch?v=fqySz1Me2pI)*


# System Design Patterns You Should Master
> **Channel:** Piyush Garg  
> **Series:** System Design Crash Course

---

## Table of Contents

1. [Microservices vs Monolith](#microservices-vs-monolith)
2. [Database per Service](#database-per-service)
3. [Circuit Breaker Pattern](#circuit-breaker-pattern)
4. [Event Sourcing](#event-sourcing)
5. [CQRS (Command Query Responsibility Segregation)](#cqrs)
6. [Pattern Relationships & When to Use What](#pattern-relationships--when-to-use-what)

---

## Microservices vs Monolith

The very first architectural decision in any system design.

### Monolith Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    MONOLITH SERVER                          │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                Single Codebase                      │   │
│  │                                                     │   │
│  │  /auth routes         /notifications handlers       │   │
│  │  /posts routes        /comments routes              │   │
│  │  /payments routes     /users routes                 │   │
│  │                                                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                          │                                  │
│                   Single Database                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Scaling a monolith:** You can still run multiple instances, but you scale the entire server — not individual features.

```
┌──────────┐    ┌──────────┐    ┌──────────┐
│ Instance │    │ Instance │    │ Instance │
│    1     │    │    2     │    │    3     │
│ (full    │    │ (full    │    │ (full    │
│  app)    │    │  app)    │    │  app)    │
└──────────┘    └──────────┘    └──────────┘
```

### Microservices Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                  MICROSERVICES ARCHITECTURE                     │
│                                                                 │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐        │
│  │  Auth        │   │ Notification │   │   Post       │        │
│  │  Service     │   │  Service     │   │   Service    │        │
│  │  (Node.js)   │   │  (Python)    │   │   (Rust)     │        │
│  │  AWS         │   │  GCP         │   │   Azure      │        │
│  └──────────────┘   └──────────────┘   └──────────────┘        │
│         ↕                   ↕                  ↕               │
│  Each service independently deployed, scaled, and maintained   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Side-by-Side Comparison

| Aspect | Monolith | Microservices |
|--------|----------|---------------|
| **Deployment** | Deploy everything together | Deploy each service independently |
| **Scaling** | Scale the whole app | Scale only what needs it |
| **Fault isolation** | One bug can crash everything | Failed service doesn't bring down others |
| **Tech flexibility** | One language/framework | Each service picks its own stack |
| **Management** | Simple — one server | Complex — N servers to manage |
| **Communication** | In-process function calls | Network calls (HTTP, gRPC, events) |
| **Best for** | Small teams, early-stage products | Large teams, complex at-scale systems |

> **Rule of thumb:** Start with a monolith. Move to microservices when you have clear domain boundaries, separate team ownership, or wildly different scaling needs per feature.

---

## Database per Service

Once you choose microservices, the next question: **do each service get their own database?**

### Shared Database (Anti-pattern at scale)

```
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│    Auth      │   │  File Upload │   │ Notification │
│   Service    │   │   Service    │   │   Service    │
└──────┬───────┘   └──────┬───────┘   └──────┬───────┘
       │                  │                  │
       └──────────────────┼──────────────────┘
                          │
                   ┌──────▼──────┐
                   │  SHARED DB  │
                   │ (Postgres)  │
                   │             │
                   │ users       │
                   │ uploads     │
                   │ notifictions│
                   └─────────────┘

Problems:
❌ Notification service can accidentally DELETE users
❌ File service bug can corrupt user data
❌ Not true isolation — shared bottleneck
❌ Schema changes in one table affect all services
```

### Database per Service (Recommended)

```
┌──────────────────────────────────────────────────────────────────┐
│                  DATABASE PER SERVICE                            │
│                                                                  │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐         │
│  │    Auth      │   │  Notification│   │    Post      │         │
│  │   Service    │   │   Service    │   │   Service    │         │
│  └──────┬───────┘   └──────┬───────┘   └──────┬───────┘         │
│         │                  │                  │                  │
│         ▼                  ▼                  ▼                  │
│  ┌─────────────┐  ┌─────────────┐   ┌─────────────┐             │
│  │  MongoDB    │  │  ClickHouse │   │  PostgreSQL  │             │
│  │             │  │             │   │              │             │
│  │ (schema     │  │ (analytics: │   │ (relational  │             │
│  │  flexible,  │  │  open rates,│   │  posts,      │             │
│  │  users)     │  │  click data)│   │  comments)   │             │
│  └─────────────┘  └─────────────┘   └─────────────┘             │
│                                                                  │
│  ✅ True isolation         ✅ Choose best DB per use case        │
│  ✅ Independent scaling    ✅ Independent indexing strategy      │
└──────────────────────────────────────────────────────────────────┘
```

### The Trade-off: Application-Level Joins

```
┌──────────────────────────────────────────────────────────────────┐
│              THE JOIN PROBLEM                                    │
│                                                                  │
│  Old way (shared DB):                                            │
│  SELECT posts.*, users.name                                      │
│  FROM posts JOIN users ON posts.user_id = users.id               │
│  ← Single SQL query, done.                                       │
│                                                                  │
│  New way (DB per service):                                       │
│                                                                  │
│  1. Post Service: SELECT * FROM posts WHERE user_id = 42         │
│          │                                                       │
│          ▼                                                       │
│  2. HTTP call to Auth Service: GET /users/42                     │
│          │                                                       │
│          ▼                                                       │
│  3. Application code merges the two results                      │
│                                                                  │
│  ❌ Extra network hop       ❌ More code complexity              │
│  ✅ True service isolation  ✅ Worth it at large scale           │
└──────────────────────────────────────────────────────────────────┘
```

---

## Circuit Breaker Pattern

Prevents a failed service from cascading failures across your entire system.

### The Problem: Cascading Failures

```
┌──────────────────────────────────────────────────────────────────┐
│                  CASCADE FAILURE (No Circuit Breaker)            │
│                                                                  │
│  Service A ──► Service B ──► Service C ──► Service D             │
│                                              (DOWN 💀)           │
│                                                                  │
│  Step 1: D is down                                               │
│  Step 2: C keeps retrying D → C gets overwhelmed → C crashes    │
│  Step 3: B keeps retrying C → B gets overwhelmed → B crashes    │
│  Step 4: A keeps retrying B → A gets overwhelmed → A crashes    │
│                                                                  │
│  Result: ONE service failure brings down ENTIRE system           │
└──────────────────────────────────────────────────────────────────┘
```

### The Solution: Circuit Breaker States

The circuit breaker sits as a **proxy** between services and has 3 states:

```
┌──────────────────────────────────────────────────────────────────┐
│               CIRCUIT BREAKER STATE MACHINE                     │
│                                                                  │
│                    failures < threshold                          │
│              ┌─────────────────────────────────┐                │
│              │                                 │                │
│              ▼                                 │                │
│    ┌─────────────────┐    failures             │                │
│    │    CLOSED       │    exceed threshold     │                │
│    │  (Normal flow)  │ ──────────────────────► │                │
│    │  Requests pass  │                         │                │
│    └─────────────────┘                         │                │
│                                                │                │
│                                      ┌─────────▼───────┐        │
│                                      │      OPEN       │        │
│                                      │  (Fail fast)    │        │
│                                      │  Block requests │        │
│                                      │  Return error   │        │
│                                      │  immediately    │        │
│                                      └─────────┬───────┘        │
│                                                │                │
│                                  timeout passes│                │
│                                                │                │
│    ┌─────────────────┐                         │                │
│    │   HALF-OPEN     │ ◄───────────────────────┘                │
│    │  (Testing)      │                                          │
│    │ Allow 1 request │                                          │
│    │ to test service │                                          │
│    └────────┬────────┘                                          │
│             │           │                                       │
│        success          failure                                  │
│             │           │                                       │
│             ▼           ▼                                       │
│          CLOSED        OPEN                                      │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Circuit Breaker in Action

```
┌──────────────────────────────────────────────────────────────────┐
│              ARCHITECTURE WITH CIRCUIT BREAKER                  │
│                                                                  │
│  Service A                                                       │
│      │                                                           │
│      ▼                                                           │
│  ┌─────────────────┐     CLOSED: requests flow through          │
│  │ Circuit Breaker │ ──► Service B (healthy) ──► Response       │
│  │    Proxy        │                                            │
│  └─────────────────┘                                            │
│      │                                                           │
│      │  Service B starts failing...                             │
│      │                                                           │
│  ┌─────────────────┐     OPEN: requests blocked                 │
│  │ Circuit Breaker │ ──► Returns error immediately              │
│  │    Proxy        │     (no retry to Service B)                │
│  └─────────────────┘                                            │
│      │                                                           │
│      │  After timeout interval...                               │
│      │                                                           │
│  ┌─────────────────┐     HALF-OPEN: test 1 request              │
│  │ Circuit Breaker │ ──► Service B (recovering?) ──► Success?   │
│  │    Proxy        │                                            │
│  └─────────────────┘     If yes → CLOSED again ✅               │
│                          If no  → OPEN again  ❌               │
│                                                                  │
│  Benefit: Service B can recover without being bombarded         │
│  by retries from all upstream services                           │
└──────────────────────────────────────────────────────────────────┘
```

**Libraries that implement Circuit Breaker:**
- Node.js: `opossum`
- Java: `Resilience4j`, `Hystrix`
- Go: `gobreaker`
- Python: `circuitbreaker`

---

## Event Sourcing

Instead of storing the **current state**, store every **event** that led to that state.

### Traditional Approach (Mutable State)

```
┌──────────────────────────────────────────────────────────────────┐
│              TRADITIONAL — MUTABLE STATE                        │
│                                                                  │
│  Orders Table:                                                   │
│  ┌────────┬──────────────────────┐                              │
│  │ ID     │ State                │                              │
│  ├────────┼──────────────────────┤                              │
│  │ #1234  │ PLACED               │  ← gets overwritten          │
│  │ #1234  │ READY_TO_SHIP        │  ← overwritten               │
│  │ #1234  │ SHIPPING             │  ← overwritten               │
│  │ #1234  │ DELIVERED            │  ← current state             │
│  └────────┴──────────────────────┘                              │
│                                                                  │
│  At any moment only ONE row exists per order                     │
│                                                                  │
│  Problems at scale:                                              │
│  ❌ Requires locking rows during updates                         │
│  ❌ No history of what happened                                  │
│  ❌ Concurrent updates cause conflicts                           │
│  ❌ Hard to audit or debug past behavior                         │
└──────────────────────────────────────────────────────────────────┘
```

### Event Sourcing Approach (Immutable Log)

```
┌──────────────────────────────────────────────────────────────────┐
│              EVENT SOURCING — APPEND-ONLY LOG                   │
│                                                                  │
│  Event Stream (Immutable):                                       │
│  ┌────────────┬──────────────────────┬──────────────────────┐   │
│  │ Timestamp  │ Event                │ Payload              │   │
│  ├────────────┼──────────────────────┼──────────────────────┤   │
│  │ 12:00:00   │ ORDER_PLACED         │ { item: "shoes" }    │   │
│  │ 12:10:00   │ READY_TO_SHIP        │ { warehouse: "MH1" } │   │
│  │ 12:30:00   │ SHIPPING_STARTED     │ { carrier: "FedEx" } │   │
│  │ 15:45:00   │ OUT_FOR_DELIVERY     │ { agent: "Bob" }     │   │
│  │ 17:00:00   │ DELIVERED            │ { lat: 19.2, lng: 73}│   │
│  └────────────┴──────────────────────┴──────────────────────┘   │
│              ↑ Events are NEVER deleted or modified              │
│                                                                  │
│  "What is the current state of order #1234?"                     │
│  → Replay all events → derive current state                      │
│                                                                  │
│  ✅ No locking needed      ✅ Full audit trail                   │
│  ✅ Debug any point in time ✅ Replay to rebuild state           │
│  ✅ Append-only = fast writes                                    │
└──────────────────────────────────────────────────────────────────┘
```

### Real-world Example: Banking

```
┌──────────────────────────────────────────────────────────────────┐
│           BANKING — CLASSIC EVENT SOURCING                      │
│                                                                  │
│  NOT stored:     balance = $300   ← never a single mutable value │
│                                                                  │
│  ACTUALLY stored: list of transactions                           │
│  ┌────────────────────────────────────────┐                     │
│  │ +$500  (salary credited)               │                     │
│  │ -$150  (rent deducted)                 │                     │
│  │ -$50   (groceries)                     │                     │
│  │ +$100  (refund received)               │                     │
│  │ -$100  (transfer to savings)           │                     │
│  └────────────────────────────────────────┘                     │
│                                                                  │
│  "What is my current balance?"                                   │
│  → Sum all transactions → $300                                   │
│                                                                  │
│  Why? Any dispute → full trail exists                            │
│  Regulatory audit → every transaction verifiable                 │
└──────────────────────────────────────────────────────────────────┘
```

---

## CQRS

**Command Query Responsibility Segregation** — naturally pairs with Event Sourcing.

### The Problem It Solves

At Amazon-scale, a single database handling both reads and writes becomes the bottleneck:
- Writes need strong consistency and transaction locks
- Reads need speed, denormalized data, and complex aggregations
- Mixing both on one DB creates contention

### CQRS Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                         CQRS PATTERN                            │
│                                                                  │
│  User Action                                                     │
│      │                                                           │
│      ▼                                                           │
│  API Gateway                                                     │
│      │                                                           │
│  ┌───┴────────────────────────────────────────────────┐         │
│  │ Route by HTTP Method                               │         │
│  └───┬─────────────────────────────────────────┬──────┘         │
│      │ POST/PUT/PATCH/DELETE                   │ GET             │
│      ▼                                         ▼                │
│ ┌──────────────┐                        ┌──────────────┐        │
│ │   COMMAND    │                        │    QUERY     │        │
│ │    SIDE      │                        │    SIDE      │        │
│ │              │                        │              │        │
│ │ Validate     │                        │ Read Model   │        │
│ │ Authorize    │                        │ (optimized   │        │
│ │ Generate     │                        │  for reads)  │        │
│ │ Event        │                        │              │        │
│ └──────┬───────┘                        └──────┬───────┘        │
│        │                                       │                │
│        ▼                                       ▼                │
│ ┌─────────────┐    Event    ┌─────────────────────────┐        │
│ │  Write DB   │ ──────────► │       Read DB           │        │
│ │ (normalized │   Stream    │  (denormalized,         │        │
│ │  SQL)       │             │   pre-joined JSON)      │        │
│ │             │             │   MongoDB/DynamoDB       │        │
│ └─────────────┘             └─────────────────────────┘        │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Why Two Databases?

```
┌──────────────────────────────┬──────────────────────────────────┐
│         WRITE DB             │           READ DB                │
├──────────────────────────────┼──────────────────────────────────┤
│ Normalized SQL               │ Denormalized NoSQL               │
│ Multiple tables + joins      │ Pre-joined as flat JSON          │
│ ACID transactions            │ Optimized for fast lookups       │
│ Source of truth              │ Derived / materialized view      │
│                              │                                  │
│ Example query:               │ Example query:                   │
│ SELECT p.*, u.name,          │ db.products.findOne({_id: 123})  │
│   s.stock, r.avg_rating      │ → returns full product doc       │
│ FROM products p              │   with all nested details        │
│ JOIN users u ON ...          │   NO JOINS NEEDED                │
│ JOIN stock s ON ...          │                                  │
│ JOIN reviews r ON ...        │                                  │
│ (slow at scale)              │ (fast at scale)                  │
└──────────────────────────────┴──────────────────────────────────┘
```

### CQRS with Event Sourcing Combined

```
┌──────────────────────────────────────────────────────────────────┐
│          CQRS + EVENT SOURCING TOGETHER                         │
│                                                                  │
│  1. User places order                                            │
│          │                                                       │
│          ▼                                                       │
│  2. Command handler validates + generates event:                 │
│     { type: "ORDER_PLACED", orderId: 123, ... }                  │
│          │                                                       │
│          ▼                                                       │
│  3. Event appended to append-only log (Kafka/Kinesis)           │
│          │                                                       │
│     ┌────┴─────────────────────────────┐                        │
│     ▼                                  ▼                        │
│  4a. Write DB updated               4b. Read DB projection      │
│     (normalized state)                  built/updated           │
│                                         (denormalized)          │
│          │                                                       │
│          ▼                                                       │
│  5. Read DB corrupted/stale?                                    │
│     → Replay all events from log → Rebuild Read DB ✅           │
│                                                                  │
│  Trade-off: Eventual consistency between Write DB & Read DB     │
│  (usually 1-2 second delay — acceptable for most systems)       │
└──────────────────────────────────────────────────────────────────┘
```

**When is eventual consistency NOT acceptable?**
Stock market trades, financial settlements, medical records — anywhere where stale data causes real harm.

---

## Pattern Relationships & When to Use What

These patterns don't exist in isolation — they form a natural progression:

```
┌──────────────────────────────────────────────────────────────────┐
│            PATTERN DECISION FLOW                                 │
│                                                                  │
│  Are you building a new system?                                  │
│       │                                                          │
│       ▼                                                          │
│  Small team / early stage?                                       │
│  → Use MONOLITH. Simpler, faster to build.                       │
│                                                                  │
│  Large team / clear domain boundaries?                           │
│  → Use MICROSERVICES.                                            │
│       │                                                          │
│       ▼                                                          │
│  Need true service isolation?                                    │
│  → Use DATABASE PER SERVICE.                                     │
│  → Accept: joins become application-level                        │
│       │                                                          │
│       ▼                                                          │
│  Services calling each other?                                    │
│  → Add CIRCUIT BREAKERS to prevent cascade failures.             │
│       │                                                          │
│       ▼                                                          │
│  High throughput? Many writes? Need audit trail?                 │
│  → Use EVENT SOURCING (append-only log as truth).                │
│       │                                                          │
│       ▼                                                          │
│  Read and write loads have different scaling requirements?       │
│  → Use CQRS to separate read and write models.                   │
│  → Works naturally on top of Event Sourcing.                     │
└──────────────────────────────────────────────────────────────────┘
```

### Quick Reference Card

| Pattern | Problem Solved | Key Trade-off |
|---------|---------------|---------------|
| **Microservices** | Independent deployment & scaling | Inter-service communication complexity |
| **Monolith** | Simplicity & cohesion | Cannot scale individual features |
| **DB per Service** | True data isolation | Joins become application-level |
| **Circuit Breaker** | Cascade failure prevention | Added proxy layer latency |
| **Event Sourcing** | Immutable audit trail, high-throughput writes | More storage, eventual state derivation |
| **CQRS** | Separate read/write optimization | Eventual consistency between DBs |

---
