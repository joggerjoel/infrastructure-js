# Performance & Scalability

---
**Last Updated**: 2026-01-28  
**Status**: 💡 GUIDANCE  
**Priority**: P1 - High Priority  
---

## Overview

Comprehensive guide to load balancing, horizontal scaling, request queuing, performance optimization, and resource limits for building scalable Node.js applications.

---

## Quick Navigation

- [Load Balancing](#load-balancing)
- [Horizontal Scaling](#horizontal-scaling)
- [Request Queuing](#request-queuing)
- [Performance Optimization](#performance-optimization)
- [Resource Limits](#resource-limits)
- [Monitoring & Benchmarking](#monitoring--benchmarking)

---

## Load Balancing

### Nginx Load Balancer

```nginx
# /etc/nginx/nginx.conf
upstream backend {
  # Load balancing method
  least_conn;  # Route to server with fewest connections
  # Other options: round_robin (default), ip_hash, random
  
  # Backend servers
  server 192.168.1.10:3000 weight=3;  # Higher weight = more traffic
  server 192.168.1.11:3000 weight=2;
  server 192.168.1.12:3000 weight=1;
  
  # Health checks
  server 192.168.1.13:3000 backup;  # Only used if others fail
  
  # Connection limits
  keepalive 32;  # Keep connections alive
}

server {
  listen 80;
  server_name api.myapp.com;
  
  location / {
    proxy_pass http://backend;
    
    # Headers
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    
    # Timeouts
    proxy_connect_timeout 5s;
    proxy_send_timeout 60s;
    proxy_read_timeout 60s;
    
    # Buffering
    proxy_buffering on;
    proxy_buffer_size 4k;
    proxy_buffers 8 4k;
    
    # Keep-alive
    proxy_http_version 1.1;
    proxy_set_header Connection "";
  }
  
  # Health check endpoint
  location /health {
    access_log off;
    proxy_pass http://backend/health;
  }
}
```

### AWS Application Load Balancer

```yaml
# infrastructure/alb.yml (CloudFormation)
Resources:
  ApplicationLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Name: my-app-alb
      Subnets:
        - !Ref PublicSubnet1
        - !Ref PublicSubnet2
      SecurityGroups:
        - !Ref ALBSecurityGroup
      Scheme: internet-facing
      
  TargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      Name: my-app-targets
      Port: 3000
      Protocol: HTTP
      VpcId: !Ref VPC
      HealthCheckPath: /health
      HealthCheckIntervalSeconds: 30
      HealthCheckTimeoutSeconds: 5
      HealthyThresholdCount: 2
      UnhealthyThresholdCount: 3
      TargetType: ip
      
  Listener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref ApplicationLoadBalancer
      Port: 443
      Protocol: HTTPS
      Certificates:
        - CertificateArn: !Ref Certificate
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref TargetGroup
```

### Node.js Cluster Mode

```typescript
import cluster from 'cluster';
import os from 'os';

if (cluster.isPrimary) {
  const numCPUs = os.cpus().length;
  
  console.log(`Primary ${process.pid} is running`);
  console.log(`Forking ${numCPUs} workers...`);
  
  // Fork workers
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }
  
  // Replace dead workers
  cluster.on('exit', (worker, code, signal) => {
    console.log(`Worker ${worker.process.pid} died`);
    console.log('Starting a new worker...');
    cluster.fork();
  });
  
} else {
  // Workers share TCP connection
  const app = express();
  
  // Your app code...
  
  app.listen(3000, () => {
    console.log(`Worker ${process.pid} started`);
  });
}
```

---

## Horizontal Scaling

### Stateless Design

```typescript
// ❌ Bad: Storing state in memory (doesn't scale)
const sessions = new Map();

app.post('/login', (req, res) => {
  const sessionId = generateSessionId();
  sessions.set(sessionId, { userId: req.body.userId });
  res.json({ sessionId });
});

app.get('/profile', (req, res) => {
  const session = sessions.get(req.headers.sessionId);
  // Problem: Session only exists on this server!
  res.json({ userId: session.userId });
});

// ✅ Good: Store state in Redis (shared across servers)
app.post('/login', async (req, res) => {
  const sessionId = generateSessionId();
  await redis.setex(
    `session:${sessionId}`,
    3600,
    JSON.stringify({ userId: req.body.userId })
  );
  res.json({ sessionId });
});

app.get('/profile', async (req, res) => {
  const session = await redis.get(`session:${req.headers.sessionId}`);
  res.json(JSON.parse(session));
});
```

### Session Management

```typescript
import session from 'express-session';
import RedisStore from 'connect-redis';

// Redis session store (shared across servers)
app.use(session({
  store: new RedisStore({ client: redis }),
  secret: process.env.SESSION_SECRET!,
  resave: false,
  saveUninitialized: false,
  cookie: {
    secure: true,  // HTTPS only
    httpOnly: true,  // No JavaScript access
    maxAge: 24 * 60 * 60 * 1000  // 24 hours
  }
}));
```

### Auto-Scaling Configuration

```yaml
# kubernetes/deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3  # Initial replicas
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: my-app:latest
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

---

## Request Queuing

### Bull Queue for Background Jobs

```typescript
import Queue from 'bull';

// Create queue
const emailQueue = new Queue('email', {
  redis: {
    host: process.env.REDIS_HOST,
    port: parseInt(process.env.REDIS_PORT || '6379')
  }
});

// Add job to queue
app.post('/send-email', async (req, res) => {
  await emailQueue.add({
    to: req.body.to,
    subject: req.body.subject,
    body: req.body.body
  }, {
    attempts: 3,  // Retry 3 times
    backoff: {
      type: 'exponential',
      delay: 2000  // 2s, 4s, 8s
    },
    removeOnComplete: true,
    removeOnFail: false
  });
  
  res.json({ message: 'Email queued' });
});

// Process jobs
emailQueue.process(async (job) => {
  const { to, subject, body } = job.data;
  
  await sendEmail(to, subject, body);
  
  return { sent: true };
});

// Handle failures
emailQueue.on('failed', (job, err) => {
  logger.error('Email job failed', {
    jobId: job.id,
    error: err.message,
    data: job.data
  });
});

// Monitor queue
emailQueue.on('completed', (job) => {
  logger.info('Email sent', { jobId: job.id });
});
```

### Priority Queues

```typescript
// High priority jobs processed first
await emailQueue.add(
  { to: 'vip@example.com', subject: 'Important' },
  { priority: 1 }  // Lower number = higher priority
);

await emailQueue.add(
  { to: 'user@example.com', subject: 'Newsletter' },
  { priority: 10 }
);
```

### Rate-Limited Queue Processing

```typescript
// Process max 10 jobs per second
emailQueue.process(10, async (job) => {
  // Process job
});

// Or use limiter
const limiter = new Bottleneck({
  maxConcurrent: 5,  // Max 5 concurrent jobs
  minTime: 100  // Min 100ms between jobs
});

emailQueue.process(async (job) => {
  return limiter.schedule(() => sendEmail(job.data));
});
```

---

## Performance Optimization

### Database Query Optimization

See [database-layer.md](database-layer.md#query-optimization) for details.

### Caching Strategy

See [intelligent-caching.md](intelligent-caching.md) for comprehensive caching patterns.

### Response Compression

```typescript
import compression from 'compression';

// Compress responses
app.use(compression({
  level: 6,  // Compression level (0-9)
  threshold: 1024,  // Only compress if > 1KB
  filter: (req, res) => {
    // Don't compress if client doesn't support it
    if (req.headers['x-no-compression']) {
      return false;
    }
    return compression.filter(req, res);
  }
}));
```

### CDN for Static Assets

```typescript
// Serve static files with long cache
app.use('/static', express.static('public', {
  maxAge: '1y',  // Cache for 1 year
  immutable: true,
  etag: false
}));

// Use CDN URL in production
const CDN_URL = process.env.CDN_URL || '';

app.get('/config', (req, res) => {
  res.json({
    assetsUrl: CDN_URL + '/static'
  });
});
```

### Async/Await Best Practices

```typescript
// ❌ Bad: Sequential execution
async function getUsers() {
  const user1 = await db.getUser(1);  // 100ms
  const user2 = await db.getUser(2);  // 100ms
  const user3 = await db.getUser(3);  // 100ms
  return [user1, user2, user3];  // Total: 300ms
}

// ✅ Good: Parallel execution
async function getUsers() {
  const [user1, user2, user3] = await Promise.all([
    db.getUser(1),
    db.getUser(2),
    db.getUser(3)
  ]);
  return [user1, user2, user3];  // Total: 100ms
}

// ✅ Better: Batch query
async function getUsers() {
  return db.query('SELECT * FROM users WHERE id IN ($1, $2, $3)', [1, 2, 3]);
}
```

---

## Resource Limits

### Memory Limits

```typescript
// Start Node.js with memory limit
// node --max-old-space-size=4096 server.js  // 4GB

// Monitor memory usage
import v8 from 'v8';

setInterval(() => {
  const heapStats = v8.getHeapStatistics();
  const usedHeap = heapStats.used_heap_size / 1024 / 1024;
  const totalHeap = heapStats.heap_size_limit / 1024 / 1024;
  const usage = (usedHeap / totalHeap) * 100;
  
  logger.info('Memory usage', {
    usedMB: usedHeap.toFixed(2),
    totalMB: totalHeap.toFixed(2),
    percentage: usage.toFixed(2)
  });
  
  if (usage > 90) {
    logger.error('High memory usage!', { usage });
  }
}, 60000);  // Every minute
```

### CPU Limits

```yaml
# docker-compose.yml
services:
  app:
    image: my-app
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 4G
        reservations:
          cpus: '0.5'
          memory: 1G
```

### Connection Limits

```typescript
// Limit concurrent requests
import pLimit from 'p-limit';

const limit = pLimit(10);  // Max 10 concurrent operations

const results = await Promise.all(
  items.map(item => limit(() => processItem(item)))
);

// Request timeout
import timeout from 'connect-timeout';

app.use(timeout('30s'));
app.use((req, res, next) => {
  if (!req.timedout) next();
});
```

---

## Monitoring & Benchmarking

### Load Testing with k6

```javascript
// loadtest.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 100 },  // Ramp up to 100 users
    { duration: '5m', target: 100 },  // Stay at 100 users
    { duration: '2m', target: 200 },  // Ramp up to 200 users
    { duration: '5m', target: 200 },  // Stay at 200 users
    { duration: '2m', target: 0 },    // Ramp down to 0 users
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],  // 95% of requests < 500ms
    http_req_failed: ['rate<0.01'],    // Error rate < 1%
  },
};

export default function () {
  const res = http.get('https://api.myapp.com/users');
  
  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });
  
  sleep(1);
}
```

```bash
# Run load test
k6 run loadtest.js

# Run with more virtual users
k6 run --vus 1000 --duration 30s loadtest.js
```

### Performance Benchmarking

```typescript
import { performance } from 'perf_hooks';

// Benchmark function
async function benchmark(name: string, fn: () => Promise<void>, iterations: number = 1000) {
  const start = performance.now();
  
  for (let i = 0; i < iterations; i++) {
    await fn();
  }
  
  const end = performance.now();
  const total = end - start;
  const avg = total / iterations;
  
  console.log(`${name}:`);
  console.log(`  Total: ${total.toFixed(2)}ms`);
  console.log(`  Average: ${avg.toFixed(2)}ms`);
  console.log(`  Ops/sec: ${(1000 / avg).toFixed(2)}`);
}

// Usage
await benchmark('Database query', async () => {
  await db.query('SELECT * FROM users LIMIT 10');
}, 100);

await benchmark('Cache lookup', async () => {
  await redis.get('user:123');
}, 1000);
```

### Metrics to Track

```typescript
import { Histogram, Gauge } from 'prom-client';

// Request duration
const httpDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration',
  labelNames: ['method', 'route', 'status'],
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 5]
});

// Active connections
const activeConnections = new Gauge({
  name: 'http_active_connections',
  help: 'Number of active HTTP connections'
});

// Queue size
const queueSize = new Gauge({
  name: 'queue_size',
  help: 'Number of jobs in queue',
  labelNames: ['queue']
});

// Update metrics
app.use((req, res, next) => {
  activeConnections.inc();
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    httpDuration.observe(
      { method: req.method, route: req.route?.path, status: res.statusCode },
      duration
    );
    activeConnections.dec();
  });
  
  next();
});
```

---

## Performance Checklist

### Application Level
- [ ] Use connection pooling (database, Redis)
- [ ] Implement caching strategy
- [ ] Enable response compression
- [ ] Optimize database queries (indexes, EXPLAIN)
- [ ] Use async/await properly (parallel when possible)
- [ ] Implement request queuing for heavy tasks
- [ ] Set appropriate timeouts
- [ ] Monitor memory usage

### Infrastructure Level
- [ ] Use load balancer (Nginx, ALB)
- [ ] Enable horizontal scaling (Kubernetes HPA)
- [ ] Use CDN for static assets
- [ ] Configure auto-scaling rules
- [ ] Set resource limits (CPU, memory)
- [ ] Use read replicas for read-heavy workloads
- [ ] Implement health checks
- [ ] Monitor performance metrics

### Testing
- [ ] Run load tests (k6, Artillery)
- [ ] Benchmark critical paths
- [ ] Test auto-scaling behavior
- [ ] Verify cache effectiveness
- [ ] Test failover scenarios
- [ ] Monitor production metrics

---

## See Also

- [Database Layer](database-layer.md) - Query optimization, connection pooling
- [Intelligent Caching](intelligent-caching.md) - Caching strategies
- [Prometheus Monitoring](prometheus.md) - Metrics and dashboards
- [DevOps Overview](devops.md) - Deployment and operations
