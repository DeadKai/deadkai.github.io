+++
title = "System Design Fundamentals: Building Scalable Applications"
date = 2024-12-05T11:20:00Z
+++

System design is the art of building applications that can scale to millions of users while remaining reliable, maintainable, and cost-effective. Whether you're preparing for interviews or building production systems, understanding these fundamentals is essential.

## Key Principles

### 1. Scalability

The ability to handle increased load by adding resources. Two types exist:

**Vertical Scaling (Scale Up)**
- Add more power to existing machines (CPU, RAM, disk)
- Simpler to implement
- Limited by hardware constraints
- Single point of failure

**Horizontal Scaling (Scale Out)**
- Add more machines to the pool
- Better fault tolerance
- More complex architecture
- Nearly unlimited scaling potential

```
Vertical: 1 server with 32GB RAM
Horizontal: 8 servers with 4GB RAM each
```

### 2. Availability

The percentage of time a system is operational. Measured in "nines":

- 99% = 3.65 days downtime/year
- 99.9% = 8.76 hours downtime/year
- 99.99% = 52.6 minutes downtime/year
- 99.999% = 5.26 minutes downtime/year

Achieve high availability through:
- Redundancy (multiple instances)
- Failover mechanisms
- Health checks and monitoring
- Geographic distribution

### 3. Reliability

The probability that a system will produce correct outputs. Key strategies:

- **Replication**: Multiple copies of data
- **Checksums**: Verify data integrity
- **Monitoring**: Detect issues early
- **Graceful degradation**: Maintain core functionality during failures

## Core Components

### Load Balancers

Distribute traffic across multiple servers:

```
                    [Load Balancer]
                    /      |      \
            [Server 1] [Server 2] [Server 3]
```

Common algorithms:
- **Round Robin**: Cycle through servers sequentially
- **Least Connections**: Send to server with fewest active connections
- **IP Hash**: Route based on client IP
- **Weighted**: Assign capacity weights to servers

### Caching

Store frequently accessed data in faster storage:

```python
# Without cache: ~100ms database query
user = database.query("SELECT * FROM users WHERE id=?", user_id)

# With cache: ~1ms memory lookup
user = cache.get(f"user:{user_id}")
if not user:
    user = database.query("SELECT * FROM users WHERE id=?", user_id)
    cache.set(f"user:{user_id}", user, ttl=3600)
```

Caching strategies:
- **Cache-Aside**: Application manages cache
- **Write-Through**: Write to cache and database simultaneously
- **Write-Back**: Write to cache, async write to database
- **Refresh-Ahead**: Proactively refresh before expiration

### Databases

#### SQL vs NoSQL

**SQL (PostgreSQL, MySQL)**
- Structured schema
- ACID transactions
- Complex queries with JOINs
- Vertical scaling primarily
- Use for: Financial data, user accounts, structured relationships

**NoSQL (MongoDB, Cassandra, DynamoDB)**
- Flexible schema
- BASE model (eventual consistency)
- Simple query patterns
- Horizontal scaling
- Use for: Logs, sessions, real-time analytics, unstructured data

#### Database Sharding

Split data across multiple databases:

```
Users 1-1M    → Shard 1
Users 1M-2M   → Shard 2
Users 2M-3M   → Shard 3
```

Sharding strategies:
- **Range-based**: Shard by ID ranges
- **Hash-based**: Hash user ID to determine shard
- **Geographic**: Shard by user location
- **Directory-based**: Lookup table maps entities to shards

### Message Queues

Decouple services and handle asynchronous processing:

```
[API] → [Queue] → [Worker 1]
                → [Worker 2]
                → [Worker 3]
```

Benefits:
- Asynchronous processing
- Load smoothing
- Fault tolerance
- Retry mechanisms

Popular options: RabbitMQ, Apache Kafka, AWS SQS, Redis

### CDN (Content Delivery Network)

Distribute static content globally:

```
User in Tokyo → Tokyo CDN node (10ms)
Instead of:
User in Tokyo → US origin server (200ms)
```

Cache:
- Images, CSS, JavaScript
- Videos and audio
- Static HTML pages
- API responses (with proper headers)

## Design Patterns

### API Gateway

Single entry point for all client requests:

```
[Mobile App]  \
[Web App]     → [API Gateway] → [Microservices]
[3rd Party]   /
```

Responsibilities:
- Authentication/authorization
- Rate limiting
- Request routing
- Response transformation
- Monitoring and logging

### Microservices vs Monolith

**Monolith**
```
[Single Application]
- User Management
- Order Processing
- Payment
- Inventory
```

Pros: Simple deployment, easier debugging, no network overhead
Cons: Hard to scale specific features, technology lock-in

**Microservices**
```
[User Service] [Order Service] [Payment Service] [Inventory Service]
```

Pros: Independent scaling, technology flexibility, team autonomy
Cons: Complex deployment, network overhead, distributed debugging

### Event-Driven Architecture

Services communicate through events:

```
[Order Service] → Event: OrderCreated → [Email Service]
                                      → [Inventory Service]
                                      → [Analytics Service]
```

Benefits:
- Loose coupling
- Easy to add new consumers
- Natural async processing
- Event replay for debugging

## Real-World Example: Instagram-Like App

Let's design a simplified photo-sharing platform:

### Requirements
- Upload and view photos
- Follow users
- News feed
- Like and comment
- 100M users, 50M daily active

### High-Level Design

```
[CDN] ← Static content (images)
  ↑
[Load Balancer]
  ↓
[API Servers (stateless)]
  ↓
[Cache Layer (Redis)]
  ↓
[Database Cluster]
  - User DB (PostgreSQL - sharded by user_id)
  - Photo DB (PostgreSQL - sharded by photo_id)
  - Feed DB (Cassandra - timeline data)

[Object Storage (S3)]
  - Original photos
  - Processed photos (thumbnails, etc.)

[Message Queue (Kafka)]
  - Photo processing jobs
  - Feed fanout jobs
  - Notification jobs
```

### Key Decisions

**Photo Storage**
- Store metadata in database (photo_id, user_id, caption, timestamp)
- Store actual images in object storage (S3)
- Use CDN for fast global delivery
- Generate multiple sizes asynchronously

**News Feed**
- Pre-compute feeds for active users (fanout on write)
- On-demand generation for inactive users (fanout on read)
- Cache recent feed items in Redis
- Store feed in Cassandra for fast timeline queries

**Scaling Strategy**
- Horizontal scaling for API servers
- Database sharding by user_id
- Read replicas for read-heavy workloads
- Async processing for non-critical tasks

## Common Interview Questions

**"Design a URL shortener"**
- Hash function for URL → short code
- Database to store mappings
- Redirect service
- Analytics tracking
- Rate limiting

**"Design a rate limiter"**
- Token bucket algorithm
- Store counters in Redis
- Distributed rate limiting
- Different limits per user tier

**"Design a chat application"**
- WebSocket connections for real-time
- Message queue for reliability
- Database for message history
- Presence system for online status

## Monitoring and Observability

Essential metrics to track:

**System Metrics**
- CPU, memory, disk usage
- Network throughput
- Request latency (p50, p95, p99)
- Error rates

**Business Metrics**
- Active users
- Request volume
- Conversion rates
- Revenue metrics

**Tools**: Prometheus, Grafana, DataDog, New Relic

## Conclusion

Great system design requires understanding trade-offs:

- **Consistency vs Availability** (CAP theorem)
- **Latency vs Throughput**
- **Cost vs Performance**
- **Complexity vs Simplicity**

Start simple and scale as needed. Premature optimization is the root of all evil. Design for the current scale + one order of magnitude, then iterate based on real metrics.

The best system designers don't just know technologies—they understand business requirements, user needs, and how to make pragmatic trade-offs that deliver value.
