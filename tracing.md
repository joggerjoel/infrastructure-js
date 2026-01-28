# Distributed Tracing & Request Correlation Guide

---
**Status**: 💡 **GUIDANCE** - Best practices for distributed tracing implementation  
**Priority**: 🟢 **P2** - Nice to have for monoliths, critical for microservices  
**Last Updated**: 2026-01-28  
**Owned By**: Backend Team, DevOps  

---

## Overview

Distributed tracing tracks requests as they flow through multiple services, databases, and external APIs. It answers the critical question: "Why is this request slow?" by showing exactly where time is spent across your entire system.

---

## The Problem: Lost in the Logs

### Without Tracing

**User reports**: "The page is slow"

**You check**:
```bash
# API Server logs
[14:23:45] GET /users - 2500ms

# Database logs
[14:23:45] SELECT * FROM users - ???ms

# Redis logs
[14:23:46] GET user:cache - ???ms

# External API logs
[14:23:46] POST /validate - ???ms
```

**Questions you can't answer**:
- Which part took 2500ms?
- Was it database? Redis? External API? Our code?
- Did we make multiple database calls?
- Which query was slow?
- What happened in what order?

### With Tracing

**You see**:
```
Request: GET /users (2500ms total)
├─ API Handler (50ms)
├─ Database: SELECT users (200ms)
├─ Loop: Process each user (1800ms)
│  ├─ Redis: GET user:1:profile (100ms)
│  ├─ External API: POST /validate (500ms) ← SLOW!
│  ├─ Redis: GET user:2:profile (100ms)
│  └─ External API: POST /validate (500ms) ← SLOW!
└─ Response serialization (450ms)

Problem identified: External API is slow (500ms each)
Solution: Batch validate requests or cache results
```

---

## Concepts: Traces, Spans, and Context

### Trace

A **trace** represents a single request journey through your entire system.

```
Trace ID: abc123-def456-ghi789

This trace includes everything that happened
for request abc123 across all services.
```

### Span

A **span** represents a single operation within a trace.

```
Span: "Database Query"
├─ Span ID: span-001
├─ Parent Span ID: span-000 (API Handler)
├─ Start Time: 14:23:45.100
├─ End Time: 14:23:45.300
├─ Duration: 200ms
├─ Attributes:
│  ├─ db.system: postgresql
│  ├─ db.statement: SELECT * FROM users
│  └─ db.rows: 150
└─ Status: OK
```

### Span Hierarchy

```
Root Span: HTTP Request (2500ms)
├─ Child Span: Validate Input (10ms)
├─ Child Span: Database Query (200ms)
├─ Child Span: Process Results (1800ms)
│  ├─ Grandchild Span: External API Call 1 (500ms)
│  ├─ Grandchild Span: External API Call 2 (500ms)
│  └─ Grandchild Span: External API Call 3 (500ms)
└─ Child Span: Serialize Response (450ms)
```

### Context Propagation

**The magic**: Passing trace context between services

```
Service A → Service B → Service C

Service A:
  Trace-ID: abc123
  Span-ID: span-a
  ↓
  HTTP Header: traceparent: 00-abc123-span-a-01
  ↓
Service B:
  Trace-ID: abc123 (same!)
  Parent-Span-ID: span-a
  Span-ID: span-b
  ↓
  HTTP Header: traceparent: 00-abc123-span-b-01
  ↓
Service C:
  Trace-ID: abc123 (same!)
  Parent-Span-ID: span-b
  Span-ID: span-c
```

---

## Implementation Level 1: Request Correlation (Simple)

### Step 1: Generate Request ID

```typescript
// src/middleware/request-id.ts
import { randomUUID } from 'crypto';
import { Request, Response, NextFunction } from 'express';

export function requestIdMiddleware(req: Request, res: Response, next: NextFunction) {
  // Get request ID from header or generate new one
  const requestId = req.headers['x-request-id'] as string || randomUUID();
  
  // Store on request object
  req.id = requestId;
  
  // Send back in response header
  res.setHeader('X-Request-ID', requestId);
  
  next();
}

// Apply to all routes
app.use(requestIdMiddleware);
```

### Step 2: Log with Request ID

```typescript
// src/middleware/logging.ts
import { logger } from '../logger';

export function loggingMiddleware(req: Request, res: Response, next: NextFunction) {
  const startTime = Date.now();
  
  // Create child logger with request ID
  req.log = logger.child({ requestId: req.id });
  
  req.log.info('Request started', {
    method: req.method,
    url: req.url,
    ip: req.ip
  });
  
  res.on('finish', () => {
    req.log.info('Request completed', {
      statusCode: res.statusCode,
      duration: Date.now() - startTime
    });
  });
  
  next();
}
```

### Step 3: Use Request ID in Code

```typescript
// src/routes/users.ts
app.get('/users', async (req, res) => {
  req.log.info('Fetching users');
  
  try {
    const users = await db('users').select('*');
    req.log.info('Users fetched', { count: users.length });
    
    res.json({ users });
  } catch (error) {
    req.log.error('Failed to fetch users', { error: error.message });
    res.status(500).json({ error: 'Internal server error' });
  }
});

// All logs for this request will have the same requestId
```

### Step 4: Propagate to Downstream Services

```typescript
// src/services/external-api.ts
import axios from 'axios';

export async function callExternalAPI(data: any, context: { requestId: string }) {
  const response = await axios.post('https://api.external.com/validate', data, {
    headers: {
      'X-Request-ID': context.requestId,  // Pass along request ID
      'Authorization': `Bearer ${process.env.API_KEY}`
    }
  });
  
  return response.data;
}

// Usage
app.post('/validate', async (req, res) => {
  const result = await callExternalAPI(req.body, {
    requestId: req.id  // Pass request ID
  });
  
  res.json(result);
});
```

### Searching Logs by Request ID

```bash
# Find all logs for a specific request
grep "abc-123-def-456" logs/app.log

# Or in production (ELK, Loki, CloudWatch)
# Query: requestId:"abc-123-def-456"

# Results:
# [14:23:45.000] INFO Request started (requestId: abc-123)
# [14:23:45.010] INFO Fetching users (requestId: abc-123)
# [14:23:45.200] INFO Users fetched (requestId: abc-123, count: 150)
# [14:23:45.210] INFO Calling external API (requestId: abc-123)
# [14:23:45.710] INFO External API response (requestId: abc-123)
# [14:23:45.720] INFO Request completed (requestId: abc-123, duration: 720ms)
```

**✅ This is good enough for most monoliths!**

---

## Implementation Level 2: OpenTelemetry (Full Tracing)

### What is OpenTelemetry?

OpenTelemetry (OTel) is the industry standard for distributed tracing, metrics, and logs.

**Benefits**:
- **Vendor-neutral**: Works with Jaeger, Zipkin, DataDog, New Relic, etc.
- **Auto-instrumentation**: Automatically traces HTTP, database, Redis, etc.
- **Standard**: W3C Trace Context specification
- **Complete**: Traces, metrics, and logs in one library

### Install OpenTelemetry

```bash
npm install @opentelemetry/sdk-node \
            @opentelemetry/auto-instrumentations-node \
            @opentelemetry/exporter-trace-otlp-http \
            @opentelemetry/resources \
            @opentelemetry/semantic-conventions
```

### Basic Setup

```typescript
// src/tracing.ts
import { NodeSDK } from '@opentelemetry/sdk-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { Resource } from '@opentelemetry/resources';
import { SemanticResourceAttributes } from '@opentelemetry/semantic-conventions';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';

const sdk = new NodeSDK({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: 'myapp-api',
    [SemanticResourceAttributes.SERVICE_VERSION]: process.env.npm_package_version,
    [SemanticResourceAttributes.DEPLOYMENT_ENVIRONMENT]: process.env.NODE_ENV || 'development'
  }),
  
  traceExporter: new OTLPTraceExporter({
    url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT || 'http://localhost:4318/v1/traces',
  }),
  
  instrumentations: [
    getNodeAutoInstrumentations({
      // Auto-instrument HTTP, Express, PostgreSQL, Redis, etc.
      '@opentelemetry/instrumentation-http': {},
      '@opentelemetry/instrumentation-express': {},
      '@opentelemetry/instrumentation-pg': {},
      '@opentelemetry/instrumentation-redis': {},
    })
  ]
});

sdk.start();

// Graceful shutdown
process.on('SIGTERM', async () => {
  await sdk.shutdown();
  process.exit(0);
});

export default sdk;
```

### Initialize in Server

```typescript
// src/server.ts
import './tracing';  // Import FIRST, before other imports!
import express from 'express';
import { db } from './database';
// ... rest of imports

const app = express();

// Your routes work exactly the same
app.get('/users', async (req, res) => {
  const users = await db('users').select('*');
  res.json({ users });
});

// OpenTelemetry automatically traces everything!
```

### Auto-Instrumentation Magic

```typescript
// No code changes needed! OpenTelemetry automatically traces:

app.get('/users', async (req, res) => {
  // ✅ HTTP request automatically traced
  
  const users = await db('users').select('*');
  // ✅ Database query automatically traced
  
  const cached = await redis.get('users:count');
  // ✅ Redis call automatically traced
  
  const external = await axios.get('https://api.external.com/data');
  // ✅ HTTP call automatically traced
  
  res.json({ users });
});

// Result: Complete trace showing all operations!
```

---

## Trace Backends: Where to Send Traces

### Option 1: Jaeger (Self-Hosted, Free, Easiest)

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    environment:
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://jaeger:4318/v1/traces
    ports:
      - "3000:3000"

  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"  # Jaeger UI
      - "4318:4318"    # OTLP HTTP receiver
    environment:
      - COLLECTOR_OTLP_ENABLED=true

# Visit: http://localhost:16686
# Select service, search traces, view flame graphs
```

### Option 2: Tempo + Grafana (Production-Ready)

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    environment:
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://tempo:4318/v1/traces
    ports:
      - "3000:3000"

  tempo:
    image: grafana/tempo:latest
    command: [ "-config.file=/etc/tempo.yaml" ]
    volumes:
      - ./tempo.yaml:/etc/tempo.yaml
      - tempo-data:/tmp/tempo
    ports:
      - "4318:4318"  # OTLP HTTP

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3001:3000"
    environment:
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin
    volumes:
      - ./grafana-datasources.yaml:/etc/grafana/provisioning/datasources/datasources.yaml

volumes:
  tempo-data:

# tempo.yaml
server:
  http_listen_port: 3200

distributor:
  receivers:
    otlp:
      protocols:
        http:

storage:
  trace:
    backend: local
    local:
      path: /tmp/tempo/traces

# grafana-datasources.yaml
apiVersion: 1
datasources:
  - name: Tempo
    type: tempo
    access: proxy
    url: http://tempo:3200
```

### Option 3: Cloud Providers (Managed, Paid)

**DataDog**:
```typescript
traceExporter: new OTLPTraceExporter({
  url: 'https://api.datadoghq.com/v1/traces',
  headers: {
    'DD-API-KEY': process.env.DD_API_KEY
  }
})
```

**New Relic**:
```typescript
traceExporter: new OTLPTraceExporter({
  url: 'https://otlp.nr-data.net/v1/traces',
  headers: {
    'api-key': process.env.NEW_RELIC_LICENSE_KEY
  }
})
```

**Honeycomb**:
```typescript
traceExporter: new OTLPTraceExporter({
  url: 'https://api.honeycomb.io/v1/traces',
  headers: {
    'x-honeycomb-team': process.env.HONEYCOMB_API_KEY
  }
})
```

---

## Manual Instrumentation (Custom Spans)

### Adding Custom Spans

```typescript
// src/tracing.ts (add to previous setup)
import { trace } from '@opentelemetry/api';

export const tracer = trace.getTracer('myapp-api', process.env.npm_package_version);

// src/services/user-service.ts
import { tracer } from '../tracing';

export async function processUsers(users: User[]) {
  // Create custom span
  return tracer.startActiveSpan('processUsers', async (span) => {
    try {
      span.setAttribute('user.count', users.length);
      
      const results = [];
      
      for (const user of users) {
        // Nested span for each user
        await tracer.startActiveSpan('processUser', async (userSpan) => {
          userSpan.setAttribute('user.id', user.id);
          userSpan.setAttribute('user.email', user.email);
          
          const result = await validateUser(user);  // Automatically traced
          results.push(result);
          
          userSpan.end();
        });
      }
      
      span.setStatus({ code: 1 }); // OK
      span.end();
      
      return results;
    } catch (error) {
      span.recordException(error);
      span.setStatus({ code: 2, message: error.message }); // ERROR
      span.end();
      throw error;
    }
  });
}
```

### Adding Custom Attributes

```typescript
import { trace } from '@opentelemetry/api';

app.post('/orders', async (req, res) => {
  const span = trace.getActiveSpan();
  
  if (span) {
    // Add custom attributes to current span
    span.setAttribute('user.id', req.user.id);
    span.setAttribute('order.total', req.body.total);
    span.setAttribute('order.items', req.body.items.length);
    span.setAttribute('payment.method', req.body.paymentMethod);
  }
  
  const order = await createOrder(req.body);
  
  res.json(order);
});
```

### Recording Errors

```typescript
import { trace } from '@opentelemetry/api';

app.get('/users/:id', async (req, res) => {
  const span = trace.getActiveSpan();
  
  try {
    const user = await db('users').where({ id: req.params.id }).first();
    
    if (!user) {
      // Record error in span
      span?.setStatus({
        code: 2,  // ERROR
        message: 'User not found'
      });
      
      return res.status(404).json({ error: 'User not found' });
    }
    
    span?.setAttribute('user.email', user.email);
    res.json(user);
    
  } catch (error) {
    // Record exception in span
    span?.recordException(error);
    span?.setStatus({
      code: 2,
      message: error.message
    });
    
    res.status(500).json({ error: 'Internal server error' });
  }
});
```

---

## Microservices: Cross-Service Tracing

### Service A (API Gateway)

```typescript
// src/server.ts
import './tracing';  // Initialize OpenTelemetry
import axios from 'axios';

app.get('/orders/:id', async (req, res) => {
  // Trace automatically started by OpenTelemetry
  
  // Call User Service
  const user = await axios.get(`http://user-service/users/${req.user.id}`);
  // ✅ Trace context automatically propagated via HTTP headers!
  
  // Call Order Service
  const order = await axios.get(`http://order-service/orders/${req.params.id}`);
  // ✅ Trace context automatically propagated!
  
  // Call Payment Service
  const payment = await axios.get(`http://payment-service/payments/${order.data.paymentId}`);
  // ✅ Trace context automatically propagated!
  
  res.json({
    order: order.data,
    user: user.data,
    payment: payment.data
  });
});

// Result: Single trace showing all service calls!
```

### Service B (User Service)

```typescript
// src/server.ts
import './tracing';  // Initialize OpenTelemetry

app.get('/users/:id', async (req, res) => {
  // ✅ Receives trace context from Service A automatically!
  // This span becomes a child of Service A's span
  
  const user = await db('users').where({ id: req.params.id }).first();
  
  res.json(user);
});
```

### Resulting Trace

```
Trace ID: abc-123-def-456

API Gateway: GET /orders/42 (850ms)
├─ User Service: GET /users/123 (50ms)
│  └─ Database: SELECT users (45ms)
├─ Order Service: GET /orders/42 (600ms)
│  ├─ Database: SELECT orders (30ms)
│  └─ Inventory Service: GET /items/789 (550ms)
│     └─ Database: SELECT items (520ms) ← SLOW!
└─ Payment Service: GET /payments/999 (180ms)
   └─ Database: SELECT payments (150ms)

Problem: Inventory Service database query is slow (520ms)
```

---

## Background Jobs: Tracing Async Work

### Propagating Context to Jobs

```typescript
// src/jobs/email-job.ts
import { tracer } from '../tracing';
import { context, propagation } from '@opentelemetry/api';

// When enqueueing job, inject trace context
app.post('/signup', async (req, res) => {
  const user = await createUser(req.body);
  
  // Get current trace context
  const traceContext = {};
  propagation.inject(context.active(), traceContext);
  
  // Enqueue job with trace context
  await emailQueue.add('welcome', {
    userId: user.id,
    email: user.email,
    traceContext  // Include trace context
  });
  
  res.json(user);
});

// When processing job, extract trace context
emailQueue.process('welcome', async (job) => {
  // Extract trace context from job data
  const ctx = propagation.extract(context.active(), job.data.traceContext || {});
  
  // Start new span linked to parent trace
  return context.with(ctx, async () => {
    return tracer.startActiveSpan('send-welcome-email', async (span) => {
      span.setAttribute('user.id', job.data.userId);
      span.setAttribute('job.id', job.id);
      
      try {
        await sendEmail({
          to: job.data.email,
          subject: 'Welcome!',
          template: 'welcome'
        });
        
        span.setStatus({ code: 1 }); // OK
        span.end();
      } catch (error) {
        span.recordException(error);
        span.setStatus({ code: 2 }); // ERROR
        span.end();
        throw error;
      }
    });
  });
});
```

---

## Integration with Logging

### Linking Traces and Logs

```typescript
// src/logger.ts
import pino from 'pino';
import { trace, context } from '@opentelemetry/api';

export function createLogger() {
  return pino({
    // Add trace context to every log
    mixin() {
      const span = trace.getSpan(context.active());
      if (!span) return {};
      
      const spanContext = span.spanContext();
      return {
        traceId: spanContext.traceId,
        spanId: spanContext.spanId,
        traceFlags: spanContext.traceFlags
      };
    }
  });
}

export const logger = createLogger();

// Usage
app.get('/users', async (req, res) => {
  logger.info('Fetching users');
  // Log includes traceId and spanId automatically!
  
  const users = await db('users').select('*');
  
  logger.info('Users fetched', { count: users.length });
  // Same traceId, different spanId
  
  res.json(users);
});

// Result:
// {"level":"info","msg":"Fetching users","traceId":"abc123","spanId":"def456"}
// {"level":"info","msg":"Users fetched","count":150,"traceId":"abc123","spanId":"ghi789"}
```

### Viewing in Grafana

```
Grafana Dashboard:
1. Logs panel shows logs with traceId
2. Click traceId → Opens Tempo trace viewer
3. See full trace with all spans
4. See all logs for that trace

Full picture: Metrics + Logs + Traces in one view!
```

---

## Debugging with Traces

### Use Case 1: Find Slow Requests

**In Jaeger UI**:
```
Search:
- Service: myapp-api
- Min Duration: 1s
- Limit: 50

Results: All requests taking >1s
Click trace → See flame graph
Identify slowest span
```

### Use Case 2: Debug Specific Error

**In Jaeger UI**:
```
Search:
- Service: myapp-api
- Tags: error=true
- Time range: Last hour

Results: All failed requests
Click trace → See error details
See exact span where error occurred
See error message, stack trace
```

### Use Case 3: Find N+1 Queries

**In Trace View**:
```
Look for: Many sequential database queries

Example:
├─ GET /users (500ms)
   ├─ SELECT * FROM users (50ms)
   ├─ SELECT * FROM posts WHERE user_id=1 (10ms)
   ├─ SELECT * FROM posts WHERE user_id=2 (10ms)
   ├─ SELECT * FROM posts WHERE user_id=3 (10ms)
   ... 100 more queries

Problem: N+1 query pattern
Solution: Use JOIN or batch load
```

---

## Sampling Strategies

### Why Sample?

**Problem**: Tracing every request generates massive data
- 1000 req/sec × 60 × 60 × 24 = 86.4M traces/day
- Storage costs explode

**Solution**: Sample a percentage of requests

### Simple Sampling (10%)

```typescript
import { TraceIdRatioBasedSampler } from '@opentelemetry/sdk-trace-base';

const sdk = new NodeSDK({
  // Sample 10% of requests
  sampler: new TraceIdRatioBasedSampler(0.1),
  
  // ... other config
});

// 10% of traces are collected
// 90% are discarded
```

### Smart Sampling (Errors + Slow + Random)

```typescript
import { Sampler, SamplingDecision, SamplingResult } from '@opentelemetry/sdk-trace-base';

class SmartSampler implements Sampler {
  shouldSample(context, traceId, spanName, spanKind, attributes): SamplingResult {
    // Always sample errors
    if (attributes['http.status_code'] >= 400) {
      return { decision: SamplingDecision.RECORD_AND_SAMPLED };
    }
    
    // Always sample slow requests (>1s)
    if (attributes['http.duration'] > 1000) {
      return { decision: SamplingDecision.RECORD_AND_SAMPLED };
    }
    
    // Sample 10% of everything else
    if (Math.random() < 0.1) {
      return { decision: SamplingDecision.RECORD_AND_SAMPLED };
    }
    
    return { decision: SamplingDecision.NOT_RECORD };
  }
}

const sdk = new NodeSDK({
  sampler: new SmartSampler(),
  // ... other config
});
```

---

## Performance Impact

### Overhead

**With auto-instrumentation**:
- CPU: +5-10%
- Memory: +50-100MB
- Latency: +1-5ms per request

**With manual spans only**:
- CPU: +1-2%
- Memory: +20-50MB
- Latency: +0.5-1ms per request

### Optimization Tips

1. **Use sampling**: Don't trace 100% in production
2. **Batch exports**: Send traces in batches
3. **Async export**: Don't block requests
4. **Limit attributes**: Don't add huge payloads
5. **Conditional instrumentation**: Only in production

```typescript
const sdk = new NodeSDK({
  // Only trace in production
  ...(process.env.NODE_ENV === 'production' && {
    traceExporter: new OTLPTraceExporter(/* ... */),
    sampler: new TraceIdRatioBasedSampler(0.1)  // 10% sampling
  })
});
```

---

## Quick Start: Tracing in 30 Minutes

```bash
# 1. Install OpenTelemetry (2 min)
npm install @opentelemetry/sdk-node \
            @opentelemetry/auto-instrumentations-node \
            @opentelemetry/exporter-trace-otlp-http

# 2. Create src/tracing.ts (5 min)
# Copy "Basic Setup" code from above

# 3. Start Jaeger (2 min)
docker run -d --name jaeger \
  -e COLLECTOR_OTLP_ENABLED=true \
  -p 16686:16686 \
  -p 4318:4318 \
  jaegertracing/all-in-one:latest

# 4. Import tracing in server.ts (1 min)
# Add: import './tracing';  as first line

# 5. Start app and generate traffic (5 min)
npm run dev
curl http://localhost:3000/users

# 6. View traces in Jaeger UI (15 min)
# Open: http://localhost:16686
# Select service, find traces, explore!

# Total: ~30 minutes to full distributed tracing! ✅
```

---

## Best Practices

### DO ✅

1. **Start with request correlation**: Simplest, works for monoliths
2. **Use auto-instrumentation**: Let OpenTelemetry do the work
3. **Add custom spans for business logic**: Trace what matters to you
4. **Use sampling in production**: Don't trace 100%
5. **Link logs and traces**: Include traceId in logs
6. **Set meaningful attributes**: Help future debugging
7. **Record errors in spans**: Essential for debugging
8. **Monitor trace error rate**: Alert on high errors

### DON'T ❌

1. **Don't trace everything**: Focus on important paths
2. **Don't add huge payloads as attributes**: Limit sizes
3. **Don't block on export**: Use async exporters
4. **Don't forget sampling**: Production generates too many traces
5. **Don't ignore context propagation**: Breaks distributed traces
6. **Don't hardcode URLs**: Use environment variables
7. **Don't skip graceful shutdown**: May lose traces
8. **Don't trace sensitive data**: PII, passwords, tokens

---

## Tracing Checklist

### Setup Checklist

- [ ] OpenTelemetry SDK installed
- [ ] Tracing initialized before app code
- [ ] Auto-instrumentation enabled
- [ ] Trace exporter configured
- [ ] Service name and version set
- [ ] Graceful shutdown implemented
- [ ] Sampling strategy configured
- [ ] Trace backend running (Jaeger/Tempo)

### Operations Checklist

- [ ] Traces viewable in UI
- [ ] Team trained on trace viewer
- [ ] Trace retention configured (30+ days)
- [ ] Sampling working correctly
- [ ] Logs linked to traces
- [ ] Alerts on high trace error rate
- [ ] Runbooks reference how to use traces

---

## Summary: The Three Levels

### Level 1: Request Correlation (30 min)
```typescript
// Request ID in headers + logs
req.id = req.headers['x-request-id'] || randomUUID();
logger.info('Processing', { requestId: req.id });
```
✅ Good for: Monoliths, small teams  
✅ Can search logs by request ID

### Level 2: Basic Tracing (2 hours)
```typescript
// OpenTelemetry auto-instrumentation
import './tracing';
// Everything automatically traced!
```
✅ Good for: Growing apps, 2-5 services  
✅ View traces in Jaeger/Tempo UI

### Level 3: Advanced Tracing (Ongoing)
```typescript
// Custom spans + context propagation + linked logs
tracer.startActiveSpan('businessOperation', async (span) => {
  span.setAttribute('custom.attribute', value);
  // ...
});
```
✅ Good for: Microservices (5+ services)  
✅ Complete observability

### When to Implement

| System Type | Recommendation |
|-------------|---------------|
| **Monolith** | Level 1 (Request Correlation) |
| **2-5 Services** | Level 2 (Basic Tracing) |
| **5+ Services** | Level 3 (Advanced Tracing) |
| **Critical Systems** | Level 3 (Always) |

### ROI

**Time Investment**:
- Request correlation: 30 min
- Basic tracing: 2 hours
- Advanced tracing: 1-2 days

**Benefits**:
- Debug issues **10x faster**
- Find performance bottlenecks **immediately**
- Understand system behavior **visually**
- Prove SLAs met **with data**

**The Question**: "Is 2 hours to set up tracing worth saving 20 hours of debugging?"

**Answer**: Always yes. 🎯

---

## Related Documentation
- [Logging Guide](./LOGGING.md) - Structured logging foundation
- [Monitoring Guide](./MONITORING.md) - Metrics and alerting
- [Performance Optimization](./PERFORMANCE.md) - Using traces to optimize

---

**Last Updated**: 2026-01-27  
**Owner**: Platform/Observability Team  
**Review**: Quarterly or when tracing strategy changes