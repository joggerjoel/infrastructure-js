# Messaging & Search Infrastructure Strategy
## Where to Use Elasticsearch, Redis, Kafka, and RabbitMQ

---
**Status**: 💡 **GUIDANCE** - Strategic recommendations for messaging and search infrastructure  
**Priority**: 🟡 **P1** - High priority for scaling and operational excellence  
**Last Updated**: 2026-01-28  
**Owned By**: Infrastructure Team, Backend Team  
---

## Table of Contents

1. [Overview](#overview)
2. [Redis - Caching & Session Storage](#redis---caching--session-storage)
3. [Elasticsearch - Search & Log Aggregation](#elasticsearch---search--log-aggregation)
4. [Kafka - Event Streaming & Event Sourcing](#kafka---event-streaming--event-sourcing)
5. [RabbitMQ - Message Queue & Task Processing](#rabbitmq---message-queue--task-processing)
6. [Decision Matrix](#decision-matrix)
7. [Application/Service Selection Guide](#application-service-selection-guide)
8. [Integration Patterns](#integration-patterns)
9. [Implementation Roadmap](#implementation-roadmap)

---

## Overview

### Technology Stack Overview

```
┌─────────────────────────────────────────────────────────────┐
│                      Service Layer                           │
│  [API Services]  [Background Workers]  [Microservices]      │
└─────────────────────────────────────────────────────────────┘
         │              │              │              │
         ▼              ▼              ▼              ▼
    ┌────────┐    ┌──────────┐   ┌─────────┐   ┌──────────┐
    │ Redis  │    │  Kafka   │   │RabbitMQ │   │Elastic-  │
    │        │    │          │   │         │   │  search  │
    └────────┘    └──────────┘   └─────────┘   └──────────┘
     Cache &      Event Stream    Task Queue    Search &
     Sessions     & Logs          & Jobs        Analytics
```

### Quick Decision Guide

| Use Case | Technology | When to Use |
|----------|------------|-------------|
| **Cache user data, sessions** | Redis | ✅ Always - Fast key-value cache |
| **Search logs, full-text search** | Elasticsearch | ✅ When you need search/analytics |
| **Event streaming, audit logs** | Kafka | ✅ High-volume event streams |
| **Task queues, job processing** | RabbitMQ or Redis (Bull) | ✅ Background jobs, async tasks |

---

## Redis - Caching & Session Storage

### ✅ Best Use Cases

#### 1. **Cache Layer (L2 Cache)**

```typescript
// Multi-tier cache: Memory → Redis → Database
import Redis from 'ioredis';

const redis = new Redis({
  host: process.env.REDIS_HOST,
  port: parseInt(process.env.REDIS_PORT || '6379'),
  password: process.env.REDIS_PASSWORD,
  db: 0, // Cache database
  maxRetriesPerRequest: 3,
  retryStrategy: (times) => Math.min(times * 50, 2000),
});

// Cache user data
async function getUser(userId: string) {
  const cacheKey = `user:${userId}`;
  
  // Try cache first
  const cached = await redis.get(cacheKey);
  if (cached) {
    return JSON.parse(cached);
  }
  
  // Cache miss - fetch from DB
  const user = await db.query('SELECT * FROM users WHERE id = $1', [userId]);
  
  // Store in cache (TTL: 1 hour)
  await redis.setex(cacheKey, 3600, JSON.stringify(user));
  
  return user;
}
```

**Why Redis?**
- Sub-millisecond latency (1-5ms)
- Built-in TTL support
- Atomic operations (INCR, SETNX for distributed locks)
- Pub/Sub for cache invalidation

#### 2. **Session Storage**
```typescript
// Store user sessions
import RedisStore from 'connect-redis';
import session from 'express-session';

app.use(session({
  store: new RedisStore({ client: redis }),
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: {
    secure: process.env.NODE_ENV === 'production',
    maxAge: 24 * 60 * 60 * 1000, // 24 hours
  }
}));
```

**Why Redis?**
- Fast session lookups (critical for every request)
- Automatic expiration
- Shared across multiple app instances (horizontal scaling)

#### 3. **Rate Limiting**
```typescript
// Rate limiting middleware
async function rateLimit(req, res, next) {
  const key = `rate_limit:${req.ip}:${req.path}`;
  const current = await redis.incr(key);
  
  if (current === 1) {
    await redis.expire(key, 60); // 60 second window
  }
  
  if (current > 100) { // 100 requests per minute
    return res.status(429).json({ error: 'Rate limit exceeded' });
  }
  
  next();
}
```

#### 4. **Background Job Queues (Bull/BullMQ)**
**Current Status**: Already mentioned in `performance-scalability.md` and `graceful-shutdown.md`

```typescript
// Use Redis as backend for Bull queues
import Queue from 'bull';

const emailQueue = new Queue('email', {
  redis: {
    host: process.env.REDIS_HOST,
    port: 6379,
    db: 1, // Separate DB for queues
  }
});
```

**Why Redis for Queues?**
- Already in infrastructure (no new dependency)
- Bull/BullMQ are mature, well-tested libraries
- Good for moderate throughput (< 10k jobs/sec)

### ❌ When NOT to Use Redis

- **Large file storage** → Use S3/Object Storage
- **Complex queries** → Use PostgreSQL/Elasticsearch
- **Permanent data** → Use PostgreSQL (Redis is volatile)
- **Very high throughput queues** (> 50k msgs/sec) → Consider Kafka

---

## Elasticsearch - Search & Log Aggregation

### ✅ Best Use Cases

#### 1. **Log Aggregation & Search (ELK Stack)**

```typescript
// Send logs to Elasticsearch via Logstash or directly
import { Client } from '@elastic/elasticsearch';

const esClient = new Client({
  node: process.env.ELASTICSEARCH_URL,
  auth: {
    username: process.env.ES_USERNAME,
    password: process.env.ES_PASSWORD,
  }
});

// Index log entries
async function indexLog(logEntry: LogEntry) {
  await esClient.index({
    index: `logs-${new Date().toISOString().split('T')[0]}`, // Daily index
    body: {
      '@timestamp': new Date().toISOString(),
      level: logEntry.level,
      message: logEntry.message,
      traceId: logEntry.traceId,
      requestId: logEntry.requestId,
      userId: logEntry.userId,
      // ... other structured fields
    }
  });
}
```

**Why Elasticsearch for Logs?**
- **Full-text search**: Find logs by any field
- **Aggregations**: Count errors by endpoint, user, time range
- **Visualization**: Kibana dashboards for log analysis
- **Retention**: Easy to manage old indices (delete by date)

**Integration with Structured Logging**:
```typescript
// Extend structured logger to also send to Elasticsearch
import pino from 'pino';

const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  // ... existing config
});

// Add Elasticsearch transport
if (process.env.ELASTICSEARCH_ENABLED === 'true') {
  logger.addStream({
    level: 'info',
    stream: {
      write: async (log) => {
        await indexLog(JSON.parse(log));
      }
    }
  });
}
```

#### 2. **Full-Text Search in Domain Data**
```typescript
// Search domain entities with full-text capabilities
async function searchEntities(query: string, filters: SearchFilters) {
  const result = await esClient.search({
    index: 'domain-entities',
    body: {
      query: {
        bool: {
          must: [
            {
              multi_match: {
                query: query,
                fields: ['name^2', 'title', 'description'], // name field boosted
                fuzziness: 'AUTO',
              }
            }
          ],
          filter: [
            { term: { status: filters.status } },
            { range: { created_at: { gte: filters.startDate } } }
          ]
        }
      },
      highlight: {
        fields: {
          name: {},
          title: {}
        }
      }
    }
  });
  
  return result.body.hits.hits.map(hit => ({
    ...hit._source,
    highlights: hit.highlight
  }));
}
```

**Why Elasticsearch for Search?**
- **Relevance scoring**: Best matches first
- **Fuzzy matching**: Handles typos
- **Faceted search**: Filter by multiple dimensions
- **Fast**: Sub-100ms search even on millions of documents

#### 3. **Analytics & Aggregations**
```typescript
// Analyze error patterns
async function getErrorAnalytics(timeRange: string) {
  const result = await esClient.search({
    index: 'logs-*',
    body: {
      size: 0, // Don't return documents, just aggregations
      query: {
        bool: {
          must: [
            { term: { level: 'error' } },
            { range: { '@timestamp': { gte: `now-${timeRange}` } } }
          ]
        }
      },
      aggs: {
        errors_by_endpoint: {
          terms: { field: 'endpoint.keyword', size: 10 },
          aggs: {
            error_count: { value_count: { field: '_id' } }
          }
        },
        errors_over_time: {
          date_histogram: {
            field: '@timestamp',
            calendar_interval: '1h'
          }
        }
      }
    }
  });
  
  return result.body.aggregations;
}
```

### ⚠️ Elasticsearch for Historical Real-Time Data

**Question**: Is Elasticsearch the right choice for historical real-time data?

**Answer**: **Yes, with caveats** - Elasticsearch excels at this use case, but understand the trade-offs.

#### ✅ Why Elasticsearch Works for Historical Real-Time Data

1. **Near Real-Time Indexing**:
   - Default refresh interval: 1 second
   - Can be tuned to 100ms for critical data
   - Data searchable within 1-2 seconds of ingestion

2. **Time-Series Optimization**:
   - Daily/weekly indices for efficient retention
   - Hot/warm/cold tier architecture
   - Automatic index lifecycle management

3. **Search + Analytics**:
   - Full-text search on recent data
   - Aggregations on historical data
   - Single query can span real-time and historical

4. **Scalability**:
   - Handles high ingestion rates (100k+ docs/sec)
   - Horizontal scaling for both throughput and storage

#### ⚠️ Trade-Offs and Limitations

1. **Not Truly Real-Time**:
   - **1-second indexing delay** (default)
   - Can reduce to 100ms but impacts performance
   - For sub-100ms requirements → Use Redis or in-memory cache

2. **Cost at Scale**:
   - Storage costs grow with retention period
   - Need hot/warm/cold tiers for cost optimization
   - Consider archiving to S3 for very old data

3. **Complexity**:
   - Index management (rollover, retention policies)
   - Cluster management (sharding, replication)
   - Query optimization for time-series data

4. **Not a Database**:
   - No ACID transactions
   - Eventual consistency (within cluster)
   - Not source of truth for critical data

#### 🎯 When Elasticsearch is RIGHT for Historical Real-Time Data

✅ **Good Fit**:
- **Logs**: Real-time log ingestion + historical analysis
- **Metrics**: Time-series metrics with search capability
- **Events**: Event streams with search and aggregation needs
- **Analytics**: Real-time dashboards + historical reporting
- **Search**: Full-text search on both recent and historical data

**Example Use Cases**:
- Application logs (last 7 days hot, 90 days warm, archive older)
- User activity events (real-time search + historical analysis)
- Business metrics (real-time dashboards + trend analysis)
- Security events (real-time detection + historical forensics)

#### ❌ When to Consider Alternatives

**For Truly Real-Time (< 100ms)**:
- **Redis Streams**: Sub-millisecond latency
- **Apache Kafka**: Real-time streaming with consumers
- **In-Memory Cache**: For current state only

**For Pure Time-Series**:
- **InfluxDB**: Optimized for time-series metrics
- **TimescaleDB**: PostgreSQL extension for time-series
- **Prometheus**: Metrics-focused, not for general search

**For Simple Historical Storage**:
- **PostgreSQL**: If you don't need search/aggregations
- **S3 + Athena**: Cost-effective for very large historical datasets

#### 💡 Best Practices for Historical Real-Time Data

**1. Index Strategy**:
```yaml
# Daily indices for efficient retention
index_pattern: "events-%{+YYYY.MM.dd}"
retention:
  hot: 7 days      # Fast SSD, frequent queries
  warm: 30 days    # Slower storage, occasional queries
  cold: 90 days    # Very slow storage, rare queries
  delete: 365 days # Archive to S3, delete from ES
```

**2. Refresh Interval Tuning**:
```typescript
// For real-time data (100ms refresh)
await esClient.indices.putSettings({
  index: 'events-realtime',
  body: {
    refresh_interval: '100ms'  // More frequent refresh
  }
});

// For historical data (30s refresh)
await esClient.indices.putSettings({
  index: 'events-historical',
  body: {
    refresh_interval: '30s'  // Less frequent, better performance
  }
});
```

**3. Hybrid Architecture**:
```
Real-Time (< 1s):     Redis Streams → Application
Near Real-Time (1-5s): Elasticsearch → Dashboards
Historical (> 5s):    Elasticsearch → Analytics
Very Old (> 90 days): S3 → Athena (query on demand)
```

**4. Cost Optimization**:
- Use **index lifecycle management** (ILM) for automatic tiering
- **Compress old indices** (reduce storage by 70-80%)
- **Archive to S3** after retention period
- **Separate clusters** for hot vs cold data

#### 📊 Decision Framework

| Requirement | Elasticsearch | Alternative |
|-------------|---------------|-------------|
| **Search on recent data** | ✅ Excellent | Redis (no search) |
| **Search on historical data** | ✅ Excellent | PostgreSQL (slower) |
| **Sub-second latency** | ⚠️ 1s delay | Redis Streams |
| **Time-series aggregations** | ✅ Excellent | InfluxDB (better) |
| **Full-text search** | ✅ Excellent | PostgreSQL (limited) |
| **Cost at scale** | ⚠️ Expensive | S3 + Athena (cheaper) |
| **ACID transactions** | ❌ No | PostgreSQL |

**Verdict**: Elasticsearch is **excellent** for historical real-time data IF:
- You need search capabilities
- 1-second delay is acceptable
- You have budget for storage
- You need aggregations and analytics

**Consider alternatives** if:
- You need sub-100ms real-time
- You only need simple time-series metrics
- Cost is primary concern
- You don't need search functionality

### ❌ When NOT to Use Elasticsearch

- **Simple key-value lookups** → Use Redis
- **ACID transactions** → Use PostgreSQL
- **Low search volume** (< 1000 searches/day) → PostgreSQL full-text search might be enough
- **Sub-100ms real-time requirements** → Use Redis Streams or Kafka
- **Pure time-series metrics** (no search needed) → Consider InfluxDB or TimescaleDB

### Implementation Priority

**Phase 1**: Log aggregation (highest value)
- Replace file-based log search with Elasticsearch
- Enable Kibana dashboards for operations team
- **Value**: Dramatically faster incident investigation

**Phase 2**: Domain data search
- Add full-text search for domain entities
- **Value**: Better user experience, faster queries

---

## Kafka - Event Streaming & Event Sourcing

### ✅ Best Use Cases

#### 1. **Event Streaming Architecture**
```typescript
// Producer: Emit events
import { Kafka } from 'kafkajs';

const kafka = new Kafka({
  clientId: 'api-service',
  brokers: process.env.KAFKA_BROKERS.split(','),
});

const producer = kafka.producer();

// Emit domain events
async function createEntity(data: EntityData) {
  // Save to database
  const entity = await db.query(
    'INSERT INTO entities (...) VALUES (...) RETURNING *',
    [data]
  );
  
  // Emit event to Kafka
  await producer.send({
    topic: 'domain.events',
    messages: [{
      key: entity.id.toString(),
      value: JSON.stringify({
        eventType: 'entity.created',
        entityId: entity.id,
        timestamp: new Date().toISOString(),
        data: entity
      })
    }]
  });
  
  return entity;
}
```

**Why Kafka?**
- **High throughput**: Millions of events per second
- **Durability**: Events persisted to disk
- **Replay capability**: Replay events for debugging/recovery
- **Multiple consumers**: Same event stream consumed by multiple services

#### 2. **Event Sourcing for Audit Trail**
```typescript
// Store all state changes as events
const events = [
  { type: 'entity.created', data: {...} },
  { type: 'entity.updated', data: {...} },
  { type: 'entity.approved', data: {...} },
];

// Reconstruct current state from events
async function getEntityState(entityId: string) {
  const events = await kafkaConsumer.getEvents('domain.events', entityId);
  return events.reduce((state, event) => applyEvent(state, event), initialState);
}
```

#### 3. **Microservices Communication**
```typescript
// Service A: Emit event
await producer.send({
  topic: 'user.updated',
  messages: [{ value: JSON.stringify({ userId: '123', email: 'new@example.com' }) }]
});

// Service B: Consume event
const consumer = kafka.consumer({ groupId: 'email-service' });
await consumer.subscribe({ topic: 'user.updated' });

await consumer.run({
  eachMessage: async ({ topic, partition, message }) => {
    const event = JSON.parse(message.value.toString());
    // Update email service cache, send welcome email, etc.
  }
});
```

#### 4. **Log Streaming to Elasticsearch**
```typescript
// Stream logs through Kafka to Elasticsearch
// This decouples log producers from Elasticsearch

// Producer (in service)
await producer.send({
  topic: 'service.logs',
  messages: [{
    value: JSON.stringify(logEntry)
  }]
});

// Consumer (separate service)
// Consumes from Kafka and indexes to Elasticsearch
// This way Elasticsearch issues don't affect the service
```

**Why Kafka for Log Streaming?**
- **Buffering**: Handles Elasticsearch outages gracefully
- **Multiple sinks**: Same logs → Elasticsearch + S3 + monitoring
- **Backpressure**: Prevents overwhelming Elasticsearch

### ❌ When NOT to Use Kafka

- **Simple task queues** → Use RabbitMQ or Redis (Bull)
- **Low volume** (< 10k events/day) → Database + triggers might be enough
- **Request-response patterns** → Use HTTP/REST
- **Simple pub/sub** → Redis pub/sub might be sufficient

### Implementation Priority

**Phase 1**: Log streaming (if using Elasticsearch)
- Stream logs through Kafka → Elasticsearch
- **Value**: Resilient log pipeline

**Phase 2**: Event sourcing (if needed)
- Store domain events in Kafka
- **Value**: Complete audit trail, event replay

**Phase 3**: Microservices communication (when you have multiple services)
- Use Kafka for async service communication
- **Value**: Loose coupling, scalability

---

## RabbitMQ - Message Queue & Task Processing

### ✅ Best Use Cases

#### 1. **Reliable Task Queues**
```typescript
// More reliable than Redis for critical jobs
import amqp from 'amqplib';

const connection = await amqp.connect(process.env.RABBITMQ_URL);
const channel = await connection.createChannel();

// Declare queue with durability
await channel.assertQueue('email-send', {
  durable: true, // Survives broker restart
  arguments: {
    'x-message-ttl': 3600000, // 1 hour TTL
    'x-max-priority': 10 // Priority support
  }
});

// Producer: Send job
await channel.sendToQueue('email-send', Buffer.from(JSON.stringify({
  to: 'user@example.com',
  subject: 'Welcome',
  body: '...'
})), {
  persistent: true, // Persist to disk
  priority: 5
});

// Consumer: Process jobs
await channel.consume('email-send', async (msg) => {
  if (msg) {
    const job = JSON.parse(msg.content.toString());
    
    try {
      await sendEmail(job.to, job.subject, job.body);
      channel.ack(msg); // Acknowledge success
    } catch (error) {
      // Reject and requeue (or send to dead letter queue)
      channel.nack(msg, false, true);
    }
  }
});
```

**Why RabbitMQ?**
- **Message persistence**: Messages survive broker restarts
- **Guaranteed delivery**: At-least-once or exactly-once semantics
- **Dead letter queues**: Failed messages go to DLQ for inspection
- **Priority queues**: Process important jobs first
- **Better for long-running jobs**: More reliable than Redis

#### 2. **Work Queues with Acknowledgment**
```typescript
// Process heavy computation jobs
await channel.consume('image-processing', async (msg) => {
  if (msg) {
    const job = JSON.parse(msg.content.toString());
    
    // Long-running job (might take minutes)
    await processImage(job.imageId);
    
    // Only ack after successful completion
    channel.ack(msg);
  }
}, {
  noAck: false, // Require manual acknowledgment
  prefetch: 1 // Process one job at a time per worker
});
```

#### 3. **Pub/Sub with Exchanges**
```typescript
// Fanout: Broadcast to all subscribers
await channel.assertExchange('notifications', 'fanout', { durable: true });

// Topic: Route by pattern
await channel.assertExchange('events', 'topic', { durable: true });

// Send notification
await channel.publish('notifications', '', Buffer.from(JSON.stringify({
  type: 'system.maintenance',
  message: 'Scheduled maintenance in 1 hour'
})));

// Multiple consumers subscribe
// - Email service
// - SMS service  
// - Push notification service
// All receive the same message
```

#### 4. **Request/Response Pattern**
```typescript
// RPC-style communication between services
const correlationId = generateUUID();
const replyQueue = await channel.assertQueue('', { exclusive: true });

// Send request
channel.sendToQueue('rpc-queue', Buffer.from(JSON.stringify(request)), {
  correlationId,
  replyTo: replyQueue.queue
});

// Wait for response
channel.consume(replyQueue.queue, (msg) => {
  if (msg.properties.correlationId === correlationId) {
    const response = JSON.parse(msg.content.toString());
    // Handle response
  }
}, { noAck: true });
```

### ❌ When NOT to Use RabbitMQ

- **Simple caching** → Use Redis
- **High-throughput event streaming** (> 100k msgs/sec) → Use Kafka
- **Simple background jobs** → Redis (Bull) is simpler
- **Search/indexing** → Use Elasticsearch

### RabbitMQ vs Redis (Bull) for Queues

| Feature | RabbitMQ | Redis (Bull) |
|---------|----------|--------------|
| **Message Persistence** | ✅ Disk + Memory | ⚠️ Memory only (can lose messages) |
| **Guaranteed Delivery** | ✅ Yes | ⚠️ Best effort |
| **Dead Letter Queues** | ✅ Built-in | ✅ Built-in |
| **Priority Queues** | ✅ Yes | ✅ Yes |
| **Setup Complexity** | ⚠️ Medium | ✅ Simple |
| **Throughput** | ✅ High (50k+ msgs/sec) | ✅ Very High (100k+ msgs/sec) |
| **Best For** | Critical jobs, long-running tasks | High-throughput, simple jobs |

**Recommendation**: 
- **Use Redis (Bull)** for most background jobs (simpler, already in stack)
- **Use RabbitMQ** for critical jobs that must not be lost (payments, notifications)

---

## Decision Matrix

### Application/Service Type Matrix

Choose the best technology based on your application/service characteristics:

| Application/Service Type | Primary Technology | Secondary Options | Why |
|-------------------------|-------------------|-------------------|-----|
| **User-Facing Web App** | Redis (cache) + PostgreSQL (data) | Elasticsearch (if search needed) | Fast responses, ACID transactions |
| **API Gateway** | Redis (rate limiting, caching) | - | Sub-millisecond latency required |
| **Authentication Service** | Redis (sessions) + PostgreSQL (users) | - | Fast session lookups, persistent user data |
| **Search Service** | Elasticsearch | PostgreSQL (if simple) | Full-text search, relevance scoring |
| **Log Aggregation Service** | Elasticsearch | Kafka (if high volume) | Search + analytics on logs |
| **Analytics Service** | Elasticsearch | PostgreSQL (if simple) | Aggregations, time-series analysis |
| **Real-Time Dashboard** | Redis (current state) + Elasticsearch (historical) | Kafka (streaming) | <100ms for current, search for historical |
| **Event Processing Service** | Kafka | RabbitMQ (if low volume) | High throughput, multiple consumers |
| **Background Job Service** | Redis (Bull) | RabbitMQ (if critical) | Simple queues, high throughput |
| **Notification Service** | RabbitMQ | Redis (Bull) | Guaranteed delivery for critical alerts |
| **Data Pipeline Service** | Kafka | RabbitMQ (if simple) | High-volume data transformation |
| **Audit/Compliance Service** | Kafka + Elasticsearch | PostgreSQL (if simple) | Event sourcing + searchable audit trail |
| **Metrics Collection Service** | Elasticsearch | InfluxDB (if metrics-only) | Time-series + search capability |
| **Session Management** | Redis | PostgreSQL (if needed) | Fast lookups, TTL support |
| **Rate Limiting Service** | Redis | - | Atomic counters, TTL |
| **Feature Flag Service** | Redis | PostgreSQL (config) | Fast reads, distributed cache |

### Use Case Matrix

| Use Case | Redis | Elasticsearch | Kafka | RabbitMQ |
|----------|-------|---------------|-------|----------|
| **Cache user data** | ✅ Primary | ❌ | ❌ | ❌ |
| **Session storage** | ✅ Primary | ❌ | ❌ | ❌ |
| **Rate limiting** | ✅ Primary | ❌ | ❌ | ❌ |
| **Simple task queues** | ✅ Bull/BullMQ | ❌ | ❌ | ⚠️ Overkill |
| **Critical task queues** | ⚠️ Can lose msgs | ❌ | ❌ | ✅ Guaranteed |
| **Log search/aggregation** | ❌ | ✅ Primary | ⚠️ Can stream to ES | ❌ |
| **Full-text search** | ❌ | ✅ Primary | ❌ | ❌ |
| **Event streaming** | ❌ | ⚠️ Can index | ✅ Primary | ⚠️ Possible but not ideal |
| **Event sourcing** | ❌ | ⚠️ Can store | ✅ Primary | ⚠️ Possible but not ideal |
| **Microservices async comm** | ⚠️ Pub/sub only | ❌ | ✅ Primary | ⚠️ Possible |
| **Analytics/aggregations** | ❌ | ✅ Primary | ⚠️ Can stream to ES | ❌ |
| **Historical real-time data** | ❌ | ✅ Primary | ⚠️ Can stream to ES | ❌ |
| **Sub-100ms real-time** | ✅ Primary | ❌ | ⚠️ Possible | ❌ |

### Technology Overlap

**Redis vs RabbitMQ for Queues**:
- **Redis (Bull)**: Simpler, faster, good for most cases
- **RabbitMQ**: More reliable, better for critical jobs

**Kafka vs RabbitMQ**:
- **Kafka**: Event streaming, high throughput, replay capability
- **RabbitMQ**: Traditional message queue, better for request/response

**Elasticsearch vs PostgreSQL Full-Text Search**:
- **Elasticsearch**: Complex queries, aggregations, log search
- **PostgreSQL**: Simple full-text search, already in stack

---

## Integration Patterns

### Pattern 1: Log Pipeline
```
Service → Kafka → Elasticsearch → Kibana
         ↓
         S3 (long-term storage)
```

**Benefits**:
- Resilient (Kafka buffers during Elasticsearch outages)
- Multiple consumers (ES + S3 + monitoring)
- Scalable

### Pattern 2: Cache-Aside with Redis
```
Request → Check Redis → Cache Hit? → Return
                    ↓ Cache Miss
                    Check Database → Store in Redis → Return
```

**Benefits**:
- Simple to implement
- Works with any database
- Already documented in `intelligent-caching.md`

### Pattern 3: Event-Driven Architecture
```
Service A → Kafka → Service B
                  → Service C
                  → Elasticsearch (for search)
```

**Benefits**:
- Loose coupling
- Scalable (add consumers easily)
- Replay capability

### Pattern 4: Hybrid Queue Strategy
```
Critical Jobs → RabbitMQ (guaranteed delivery)
Simple Jobs → Redis Bull (high throughput)
```

**Benefits**:
- Right tool for each job
- Cost-effective (Redis already in stack)

---

## Implementation Roadmap

### Phase 1: Foundation (Month 1-2)
**Priority**: 🔴 P0 - Critical

1. **Redis** ✅ Already planned
   - [ ] Set up Redis cluster (ElastiCache or self-hosted)
   - [ ] Implement L2 cache layer (see `intelligent-caching.md`)
   - [ ] Add session storage
   - [ ] Implement rate limiting
   - **Value**: Immediate performance improvement

### Phase 2: Observability (Month 2-3)
**Priority**: 🟡 P1 - High Value

2. **Elasticsearch for Logs**
   - [ ] Set up Elasticsearch cluster
   - [ ] Configure Logstash or direct indexing
   - [ ] Set up Kibana dashboards
   - [ ] Migrate log search from files to Elasticsearch
   - **Value**: 10x faster incident investigation

### Phase 3: Search (Month 3-4)
**Priority**: 🟢 P2 - Nice to Have

3. **Elasticsearch for Domain Data Search**
   - [ ] Index domain entities
   - [ ] Implement search API
   - [ ] Add search UI
   - **Value**: Better user experience

### Phase 4: Event Streaming (Month 4-6)
**Priority**: 🟢 P2 - When Needed

4. **Kafka** (only if needed)
   - [ ] Set up Kafka cluster (if event volume justifies it)
   - [ ] Implement event producers
   - [ ] Set up log streaming through Kafka
   - **Value**: Scalable event architecture

### Phase 5: Reliable Queues (Month 6+)
**Priority**: 🟢 P2 - When Needed

5. **RabbitMQ** (only if Redis queues insufficient)
   - [ ] Set up RabbitMQ cluster
   - [ ] Migrate critical jobs from Redis to RabbitMQ
   - [ ] Set up dead letter queues
   - **Value**: Guaranteed job delivery

---

## Cost Considerations

### Estimated Monthly Costs (AWS)

| Service | Small (< 1M requests/day) | Medium (10M requests/day) | Large (100M+ requests/day) |
|---------|---------------------------|---------------------------|----------------------------|
| **Redis (ElastiCache)** | $15-30 | $100-200 | $500-1000 |
| **Elasticsearch (OpenSearch)** | $50-100 | $300-500 | $2000-5000 |
| **Kafka (MSK)** | $100-200 | $500-1000 | $3000-10000 |
| **RabbitMQ (MQ)** | $50-100 | $200-400 | $1000-2000 |

### Cost Optimization Tips

1. **Start with Redis**: Already needed for caching, can handle queues too
2. **Elasticsearch**: Use managed service (AWS OpenSearch) to reduce ops overhead
3. **Kafka**: Only if volume justifies cost (consider managed MSK)
4. **RabbitMQ**: Only if Redis queues cause data loss issues

---

## Summary Recommendations

### ✅ Implement Now

1. **Redis** - Already planned, implement caching and sessions
2. **Elasticsearch** - High value for log aggregation and search

### ⚠️ Consider Later

3. **Kafka** - Only if you need event streaming or high-volume event processing
4. **RabbitMQ** - Only if Redis queues lose critical messages

### Architecture Decision

**Recommended Stack**:
```
Services
    ↓
Redis (Cache, Sessions, Simple Queues)
    ↓
PostgreSQL (Primary Database)
    ↓
Elasticsearch (Logs, Search)
```

**Add Kafka** when:
- Event volume > 100k events/day
- Need event replay
- Multiple services consuming same events

**Add RabbitMQ** when:
- Redis queues lose critical messages
- Need guaranteed delivery for payments/notifications
- Long-running jobs (> 5 minutes)

---

## See Also

- [Intelligent Caching](intelligent-caching.md) - Redis caching strategies
- [Database Layer](database-layer.md) - PostgreSQL usage
- [Logging](logging.md) - Current logging setup
- [Performance & Scalability](performance-scalability.md) - Queue usage with Bull
- [Integration Guide](integration-guide.md) - How components work together

---

**Last Reviewed**: 2026-01-28  
**Next Review**: 2026-04-28 (Quarterly)
