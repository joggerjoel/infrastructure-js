# Monitoring & Metrics Guide (Prometheus + Grafana)

---
**Status**: 💡 **GUIDANCE** - Best practices for monitoring and metrics implementation  
**Priority**: 🔴 **P1.5** - Critical for production operations  
**Last Updated**: 2026-01-28  
**Owned By**: Backend Team, DevOps  

---

## Overview

Monitoring is critical for understanding system health, detecting issues before users notice, and debugging production problems. This guide covers Prometheus metrics, Grafana dashboards, alerting, and complete observability setup.

---

## Why Monitoring Matters

### The Three Pillars of Observability

1. **Metrics** (this guide): Numbers over time (CPU, requests/sec, latency)
2. **Logs**: Individual events (already covered in LOGGING.md)
3. **Traces**: Request flows across services (future: distributed tracing)

### What You Can't See, You Can't Fix

**Without monitoring**:
- "Site is slow" - but where? why? how slow?
- "Some users can't login" - how many? when did it start?
- "Database is struggling" - connections? queries? disk?

**With monitoring**:
- P95 latency increased from 50ms to 500ms at 14:23
- 15% of login attempts failing since 14:20
- Database connection pool at 95% capacity
- Immediate alerts to on-call engineer

---

## Prometheus Fundamentals

### What is Prometheus?

Prometheus is a time-series database and monitoring system that:
- **Scrapes metrics** from your application via HTTP endpoints
- **Stores time-series data** efficiently
- **Queries** with PromQL (Prometheus Query Language)
- **Alerts** based on metric thresholds
- **Integrates with Grafana** for visualization

### Metric Types

| Type | Use Case | Example |
|------|----------|---------|
| **Counter** | Monotonically increasing value | `http_requests_total` |
| **Gauge** | Value that goes up and down | `memory_usage_bytes` |
| **Histogram** | Distribution of values | `http_request_duration_seconds` |
| **Summary** | Similar to histogram, pre-calculated percentiles | `api_latency_summary` |

### Counter vs Gauge

```typescript
// Counter - always increases (or resets to 0)
http_requests_total: 1234  // Total requests since server start
http_requests_total: 1235  // After one more request
http_requests_total: 1236  // Keeps going up

// Gauge - current value
active_connections: 42   // Current active connections
active_connections: 38   // Four connections closed
active_connections: 45   // Seven new connections

// Rate from counter
rate(http_requests_total[5m])  // Requests per second over 5 minutes
```

---

## Implementation in Node.js

### Install Dependencies

```bash
npm install prom-client express
```

### Basic Prometheus Setup

```typescript
// src/metrics.ts
import { Registry, Counter, Gauge, Histogram } from 'prom-client';

// Create registry
export const register = new Registry();

// Add default metrics (CPU, memory, etc.)
import { collectDefaultMetrics } from 'prom-client';
collectDefaultMetrics({ register });

// Custom metrics

// Counter: HTTP requests
export const httpRequestsTotal = new Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status'],
  registers: [register]
});

// Counter: HTTP errors
export const httpErrorsTotal = new Counter({
  name: 'http_errors_total',
  help: 'Total number of HTTP errors',
  labelNames: ['method', 'route', 'status'],
  registers: [register]
});

// Histogram: Request duration
export const httpRequestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status'],
  buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1, 5, 10], // 1ms to 10s
  registers: [register]
});

// Gauge: Active connections
export const activeConnections = new Gauge({
  name: 'active_connections',
  help: 'Number of active connections',
  registers: [register]
});

// Gauge: Database pool
export const dbPoolSize = new Gauge({
  name: 'database_pool_size',
  help: 'Current database connection pool size',
  labelNames: ['state'], // 'used', 'free', 'pending'
  registers: [register]
});

// Counter: Database queries
export const dbQueriesTotal = new Counter({
  name: 'database_queries_total',
  help: 'Total number of database queries',
  labelNames: ['operation'], // 'select', 'insert', 'update', 'delete'
  registers: [register]
});

// Histogram: Database query duration
export const dbQueryDuration = new Histogram({
  name: 'database_query_duration_seconds',
  help: 'Duration of database queries in seconds',
  labelNames: ['operation'],
  buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1, 5],
  registers: [register]
});

// Counter: Background jobs
export const jobsTotal = new Counter({
  name: 'jobs_total',
  help: 'Total number of background jobs processed',
  labelNames: ['queue', 'status'], // status: 'completed', 'failed'
  registers: [register]
});

// Histogram: Job duration
export const jobDuration = new Histogram({
  name: 'job_duration_seconds',
  help: 'Duration of background jobs in seconds',
  labelNames: ['queue'],
  buckets: [0.1, 0.5, 1, 5, 10, 30, 60, 300],
  registers: [register]
});

// Gauge: Job queue size
export const jobQueueSize = new Gauge({
  name: 'job_queue_size',
  help: 'Number of jobs waiting in queue',
  labelNames: ['queue'],
  registers: [register]
});
```

### Metrics Endpoint

```typescript
// src/server.ts
import express from 'express';
import { register } from './metrics';

const app = express();

// Metrics endpoint for Prometheus to scrape
app.get('/metrics', async (req, res) => {
  try {
    res.set('Content-Type', register.contentType);
    const metrics = await register.metrics();
    res.end(metrics);
  } catch (err) {
    res.status(500).end(err);
  }
});

// Example output at http://localhost:3000/metrics:
// # HELP http_requests_total Total number of HTTP requests
// # TYPE http_requests_total counter
// http_requests_total{method="GET",route="/users",status="200"} 1523
// http_requests_total{method="POST",route="/users",status="201"} 42
// http_requests_total{method="GET",route="/users",status="404"} 5
```

### Request Tracking Middleware

```typescript
// src/middleware/metrics.ts
import { Request, Response, NextFunction } from 'express';
import {
  httpRequestsTotal,
  httpErrorsTotal,
  httpRequestDuration,
  activeConnections
} from '../metrics';

export function metricsMiddleware(req: Request, res: Response, next: NextFunction) {
  // Track active connections
  activeConnections.inc();
  
  // Start timer
  const start = Date.now();
  
  // Track request completion
  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000; // Convert to seconds
    const route = req.route?.path || req.path;
    const method = req.method;
    const status = res.statusCode.toString();
    
    // Increment request counter
    httpRequestsTotal.inc({
      method,
      route,
      status
    });
    
    // Track errors
    if (res.statusCode >= 400) {
      httpErrorsTotal.inc({
        method,
        route,
        status
      });
    }
    
    // Record duration
    httpRequestDuration.observe(
      { method, route, status },
      duration
    );
    
    // Decrement active connections
    activeConnections.dec();
  });
  
  next();
}

// Apply to all routes
app.use(metricsMiddleware);
```

### Database Metrics

```typescript
// src/database/metrics.ts
import { dbPoolSize, dbQueriesTotal, dbQueryDuration } from '../metrics';

// Track pool stats periodically
export function startDbPoolMetrics(db: Knex) {
  setInterval(() => {
    const pool = db.client.pool;
    
    dbPoolSize.set({ state: 'used' }, pool.numUsed());
    dbPoolSize.set({ state: 'free' }, pool.numFree());
    dbPoolSize.set({ state: 'pending' }, pool.numPendingAcquires());
  }, 5000); // Every 5 seconds
}

// Track queries
db.on('query', (query) => {
  const start = Date.now();
  const operation = query.sql.trim().split(' ')[0].toLowerCase();
  
  query.response.on('end', () => {
    const duration = (Date.now() - start) / 1000;
    
    dbQueriesTotal.inc({ operation });
    dbQueryDuration.observe({ operation }, duration);
  });
});
```

### Background Job Metrics

```typescript
// src/jobs/metrics.ts
import Queue from 'bull';
import { jobsTotal, jobDuration, jobQueueSize } from '../metrics';

export function trackJobMetrics(queue: Queue.Queue) {
  const queueName = queue.name;
  
  // Track queue size periodically
  setInterval(async () => {
    const waiting = await queue.getWaitingCount();
    const active = await queue.getActiveCount();
    const delayed = await queue.getDelayedCount();
    
    jobQueueSize.set({ queue: queueName, state: 'waiting' }, waiting);
    jobQueueSize.set({ queue: queueName, state: 'active' }, active);
    jobQueueSize.set({ queue: queueName, state: 'delayed' }, delayed);
  }, 10000); // Every 10 seconds
  
  // Track job completion
  queue.on('completed', (job, result) => {
    jobsTotal.inc({ queue: queueName, status: 'completed' });
    
    if (job.finishedOn && job.processedOn) {
      const duration = (job.finishedOn - job.processedOn) / 1000;
      jobDuration.observe({ queue: queueName }, duration);
    }
  });
  
  // Track job failures
  queue.on('failed', (job, err) => {
    jobsTotal.inc({ queue: queueName, status: 'failed' });
  });
}
```

### Business Metrics

```typescript
// src/metrics/business.ts
import { Counter, Gauge } from 'prom-client';
import { register } from './metrics';

// User signups
export const userSignupsTotal = new Counter({
  name: 'user_signups_total',
  help: 'Total number of user signups',
  labelNames: ['source'], // 'web', 'mobile', 'api'
  registers: [register]
});

// Active users
export const activeUsers = new Gauge({
  name: 'active_users',
  help: 'Number of currently active users',
  registers: [register]
});

// Payments
export const paymentsTotal = new Counter({
  name: 'payments_total',
  help: 'Total number of payments processed',
  labelNames: ['status'], // 'succeeded', 'failed'
  registers: [register]
});

export const paymentAmount = new Counter({
  name: 'payment_amount_cents',
  help: 'Total payment amount in cents',
  registers: [register]
});

// Usage in code
app.post('/api/auth/signup', async (req, res) => {
  const user = await createUser(req.body);
  
  // Track signup
  userSignupsTotal.inc({ source: 'web' });
  
  res.json(user);
});

app.post('/api/payments', async (req, res) => {
  try {
    const payment = await processPayment(req.body);
    
    // Track successful payment
    paymentsTotal.inc({ status: 'succeeded' });
    paymentAmount.inc(payment.amount);
    
    res.json(payment);
  } catch (err) {
    // Track failed payment
    paymentsTotal.inc({ status: 'failed' });
    throw err;
  }
});
```

---

## Prometheus Server Setup

### Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Your application
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
    networks:
      - monitoring

  # Prometheus
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
    networks:
      - monitoring

  # Grafana
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    depends_on:
      - prometheus
    networks:
      - monitoring

  # Node Exporter (system metrics)
  node-exporter:
    image: prom/node-exporter:latest
    ports:
      - "9100:9100"
    networks:
      - monitoring

volumes:
  prometheus-data:
  grafana-data:

networks:
  monitoring:
```

### Prometheus Configuration

```yaml
# prometheus.yml
global:
  scrape_interval: 15s      # Scrape targets every 15 seconds
  evaluation_interval: 15s  # Evaluate rules every 15 seconds
  external_labels:
    cluster: 'production'
    region: 'us-east-1'

# Scrape configurations
scrape_configs:
  # Your application
  - job_name: 'nodejs-app'
    static_configs:
      - targets: ['app:3000']
        labels:
          app: 'myapp'
          env: 'production'
    metrics_path: '/metrics'
    scrape_interval: 10s

  # System metrics
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  # Prometheus itself
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

# Alerting rules
rule_files:
  - 'alerts.yml'

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']
```

### Alert Rules

```yaml
# alerts.yml
groups:
  - name: application_alerts
    interval: 30s
    rules:
      # High error rate
      - alert: HighErrorRate
        expr: |
          rate(http_errors_total[5m]) / rate(http_requests_total[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }} (threshold: 5%)"

      # Slow response time
      - alert: SlowResponseTime
        expr: |
          histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Slow response times detected"
          description: "P95 latency is {{ $value }}s (threshold: 1s)"

      # Database connection pool exhaustion
      - alert: DatabasePoolExhausted
        expr: |
          database_pool_size{state="used"} / 
          (database_pool_size{state="used"} + database_pool_size{state="free"}) > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Database connection pool almost exhausted"
          description: "Pool utilization is {{ $value | humanizePercentage }}"

      # Application down
      - alert: ApplicationDown
        expr: up{job="nodejs-app"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Application is down"
          description: "Application has been down for more than 1 minute"

      # High memory usage
      - alert: HighMemoryUsage
        expr: |
          process_resident_memory_bytes / 1024 / 1024 > 1024
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage"
          description: "Memory usage is {{ $value }}MB (threshold: 1GB)"

      # Job queue backing up
      - alert: JobQueueBackup
        expr: job_queue_size{state="waiting"} > 1000
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Job queue backing up"
          description: "{{ $value }} jobs waiting in queue"

      # High job failure rate
      - alert: HighJobFailureRate
        expr: |
          rate(jobs_total{status="failed"}[5m]) / 
          rate(jobs_total[5m]) > 0.1
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "High job failure rate"
          description: "Job failure rate is {{ $value | humanizePercentage }}"

  - name: business_alerts
    interval: 1m
    rules:
      # Payment failures
      - alert: HighPaymentFailureRate
        expr: |
          rate(payments_total{status="failed"}[5m]) /
          rate(payments_total[5m]) > 0.1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High payment failure rate"
          description: "Payment failure rate is {{ $value | humanizePercentage }}"
```

---

## Grafana Dashboards

### Datasource Configuration

```yaml
# grafana/provisioning/datasources/prometheus.yml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false
```

### Dashboard: Application Overview

```json
{
  "dashboard": {
    "title": "Application Overview",
    "panels": [
      {
        "title": "Request Rate",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])",
            "legendFormat": "{{method}} {{route}}"
          }
        ],
        "type": "graph"
      },
      {
        "title": "Error Rate",
        "targets": [
          {
            "expr": "rate(http_errors_total[5m]) / rate(http_requests_total[5m]) * 100",
            "legendFormat": "Error %"
          }
        ],
        "type": "graph",
        "yaxes": [
          {
            "format": "percent",
            "max": 100,
            "min": 0
          }
        ]
      },
      {
        "title": "Response Time (P50, P95, P99)",
        "targets": [
          {
            "expr": "histogram_quantile(0.50, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "P50"
          },
          {
            "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "P95"
          },
          {
            "expr": "histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "P99"
          }
        ],
        "type": "graph",
        "yaxes": [
          {
            "format": "s"
          }
        ]
      },
      {
        "title": "Active Connections",
        "targets": [
          {
            "expr": "active_connections"
          }
        ],
        "type": "graph"
      },
      {
        "title": "Database Pool",
        "targets": [
          {
            "expr": "database_pool_size{state=\"used\"}",
            "legendFormat": "Used"
          },
          {
            "expr": "database_pool_size{state=\"free\"}",
            "legendFormat": "Free"
          }
        ],
        "type": "graph"
      },
      {
        "title": "Memory Usage",
        "targets": [
          {
            "expr": "process_resident_memory_bytes / 1024 / 1024",
            "legendFormat": "Memory (MB)"
          }
        ],
        "type": "graph"
      }
    ]
  }
}
```

### Pre-built Dashboard Import

```bash
# Import Node.js Application Dashboard
# Grafana Dashboard ID: 11159
# https://grafana.com/grafana/dashboards/11159

# In Grafana UI:
# 1. Go to Dashboards → Import
# 2. Enter dashboard ID: 11159
# 3. Select Prometheus datasource
# 4. Click Import
```

---

## PromQL Query Examples

### Request Metrics

```promql
# Requests per second (overall)
rate(http_requests_total[5m])

# Requests per second by route
sum(rate(http_requests_total[5m])) by (route)

# Requests per second by status code
sum(rate(http_requests_total[5m])) by (status)

# Error rate percentage
rate(http_errors_total[5m]) / rate(http_requests_total[5m]) * 100

# Top 5 slowest routes
topk(5, 
  histogram_quantile(0.95, 
    rate(http_request_duration_seconds_bucket[5m])
  ) by (route)
)
```

### Latency Metrics

```promql
# P50 latency
histogram_quantile(0.50, rate(http_request_duration_seconds_bucket[5m]))

# P95 latency
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# P99 latency
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# Average latency
rate(http_request_duration_seconds_sum[5m]) / 
rate(http_request_duration_seconds_count[5m])

# Latency by route
histogram_quantile(0.95, 
  rate(http_request_duration_seconds_bucket[5m])
) by (route)
```

### Database Metrics

```promql
# Query rate
rate(database_queries_total[5m])

# Queries per second by operation
sum(rate(database_queries_total[5m])) by (operation)

# Slow queries (>1s)
sum(rate(database_query_duration_seconds_bucket{le="1"}[5m])) by (operation)

# Database pool utilization
database_pool_size{state="used"} / 
(database_pool_size{state="used"} + database_pool_size{state="free"})
```

### System Metrics

```promql
# CPU usage percentage
rate(process_cpu_seconds_total[5m]) * 100

# Memory usage in MB
process_resident_memory_bytes / 1024 / 1024

# Heap usage percentage
nodejs_heap_size_used_bytes / nodejs_heap_size_total_bytes * 100

# File descriptor usage
process_open_fds / process_max_fds * 100
```

### Business Metrics

```promql
# Signups per hour
rate(user_signups_total[1h]) * 3600

# Payment success rate
rate(payments_total{status="succeeded"}[5m]) / 
rate(payments_total[5m]) * 100

# Revenue per hour (in dollars)
rate(payment_amount_cents[1h]) * 3600 / 100
```

---

## Alertmanager Setup

### Alertmanager Configuration

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'

route:
  group_by: ['alertname', 'cluster']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  receiver: 'slack-notifications'
  
  routes:
    # Critical alerts
    - match:
        severity: critical
      receiver: 'slack-critical'
      continue: true
    
    # Warning alerts
    - match:
        severity: warning
      receiver: 'slack-warnings'

receivers:
  # Slack for all alerts
  - name: 'slack-notifications'
    slack_configs:
      - channel: '#alerts'
        title: 'Alert: {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

  # Critical alerts (page on-call)
  - name: 'slack-critical'
    slack_configs:
      - channel: '#critical-alerts'
        title: '🚨 CRITICAL: {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
    pagerduty_configs:
      - service_key: 'YOUR_PAGERDUTY_KEY'

  # Warnings (Slack only)
  - name: 'slack-warnings'
    slack_configs:
      - channel: '#warnings'
        title: '⚠️ Warning: {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
```

---

## Health Checks Integration

### Enhanced Health Check

```typescript
// src/routes/health.ts
import { Request, Response } from 'express';
import { register } from '../metrics';

export async function healthCheck(req: Request, res: Response) {
  const health = {
    status: 'healthy',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    version: process.env.npm_package_version,
    checks: {
      database: 'unknown',
      redis: 'unknown',
      memory: 'unknown'
    },
    metrics: {}
  };
  
  try {
    // Check database
    await db.raw('SELECT 1');
    health.checks.database = 'healthy';
  } catch (err) {
    health.checks.database = 'unhealthy';
    health.status = 'degraded';
  }
  
  try {
    // Check Redis
    await redis.ping();
    health.checks.redis = 'healthy';
  } catch (err) {
    health.checks.redis = 'unhealthy';
    health.status = 'degraded';
  }
  
  // Check memory usage
  const memUsage = process.memoryUsage();
  const heapUsedMB = memUsage.heapUsed / 1024 / 1024;
  const heapTotalMB = memUsage.heapTotal / 1024 / 1024;
  
  if (heapUsedMB / heapTotalMB > 0.9) {
    health.checks.memory = 'warning';
    health.status = 'degraded';
  } else {
    health.checks.memory = 'healthy';
  }
  
  // Include key metrics (optional)
  if (req.query.metrics === 'true') {
    const metrics = await register.metrics();
    health.metrics = parsePrometheusMetrics(metrics);
  }
  
  const statusCode = health.status === 'healthy' ? 200 : 503;
  res.status(statusCode).json(health);
}

// Helper to parse Prometheus metrics
function parsePrometheusMetrics(metrics: string) {
  const lines = metrics.split('\n');
  const parsed = {};
  
  for (const line of lines) {
    if (!line.startsWith('#') && line.trim()) {
      const [name, value] = line.split(' ');
      parsed[name] = parseFloat(value);
    }
  }
  
  return parsed;
}
```

---

## Production Deployment

### Kubernetes Deployment

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest
        ports:
        - containerPort: 3000
          name: http
        - containerPort: 9090
          name: metrics  # Metrics port
        
        # Liveness probe
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        
        # Readiness probe
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5

---
# Service for metrics
apiVersion: v1
kind: Service
metadata:
  name: myapp-metrics
  labels:
    app: myapp
spec:
  ports:
  - port: 3000
    targetPort: 3000
    name: metrics
  selector:
    app: myapp

---
# ServiceMonitor for Prometheus Operator
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
  - port: metrics
    path: /metrics
    interval: 15s
```

### AWS ECS Task Definition

```json
{
  "family": "myapp",
  "containerDefinitions": [
    {
      "name": "app",
      "image": "myapp:latest",
      "portMappings": [
        {
          "containerPort": 3000,
          "protocol": "tcp"
        }
      ],
      "healthCheck": {
        "command": [
          "CMD-SHELL",
          "curl -f http://localhost:3000/health || exit 1"
        ],
        "interval": 30,
        "timeout": 5,
        "retries": 3
      }
    }
  ]
}
```

---

## Monitoring Best Practices

### DO ✅

1. **Monitor the Four Golden Signals**:
   - **Latency**: How long requests take
   - **Traffic**: Request rate
   - **Errors**: Error rate
   - **Saturation**: Resource utilization

2. **Use appropriate metric types**:
   - Counters for cumulative values
   - Gauges for current values
   - Histograms for distributions

3. **Label strategically**:
   - Add labels for dimensions (method, route, status)
   - Keep cardinality low (avoid user IDs as labels)

4. **Set meaningful thresholds**:
   - Based on user experience
   - Alert on trends, not spikes

5. **Monitor business metrics**:
   - Not just technical metrics
   - Revenue, signups, conversions

6. **Create runbooks**:
   - What to do when alert fires
   - Link in alert annotations

### DON'T ❌

1. **Don't create too many alerts**:
   - Alert fatigue leads to ignoring alerts
   - Only alert on actionable issues

2. **Don't use high cardinality labels**:
   - Avoid: user_id, request_id, timestamp
   - Use: route, method, status

3. **Don't alert on everything**:
   - Warning != Alert
   - Only page for critical issues

4. **Don't ignore retention**:
   - Keep metrics for 30+ days
   - Longer for capacity planning

5. **Don't forget to test alerts**:
   - Manually trigger alerts
   - Verify on-call receives them

6. **Don't monitor without documentation**:
   - Document what metrics mean
   - Document alert response

---

## Monitoring Checklist

### Pre-Production Checklist

- [ ] Prometheus installed and configured
- [ ] Application exposing `/metrics` endpoint
- [ ] Default metrics enabled (CPU, memory, etc.)
- [ ] HTTP request metrics tracked
- [ ] Database metrics tracked
- [ ] Job queue metrics tracked (if applicable)
- [ ] Business metrics tracked
- [ ] Grafana dashboards created
- [ ] Alert rules configured
- [ ] Alertmanager configured
- [ ] Slack/PagerDuty integration tested
- [ ] Health check endpoints working
- [ ] Retention policy configured (30+ days)
- [ ] Backup strategy for Prometheus data

### Post-Deployment Checklist

- [ ] Metrics scraping successfully
- [ ] Dashboards showing data
- [ ] Alerts not firing (baseline established)
- [ ] Alert thresholds reasonable
- [ ] Team trained on dashboards
- [ ] On-call rotation configured
- [ ] Runbooks written for alerts
- [ ] Metrics reviewed weekly
- [ ] Alert noise managed

---

## Common Queries for Operations

### Debugging Production Issues

```promql
# Find slow endpoints
topk(10, 
  histogram_quantile(0.95, 
    rate(http_request_duration_seconds_bucket[5m])
  ) by (route)
)

# Find high-error endpoints
topk(10, 
  rate(http_errors_total[5m]) by (route)
)

# Check if database is bottleneck
rate(database_query_duration_seconds_sum[5m]) / 
rate(database_query_duration_seconds_count[5m])

# Memory leak detection
process_resident_memory_bytes - process_resident_memory_bytes offset 1h

# CPU spike investigation
rate(process_cpu_seconds_total[1m])
```

### Capacity Planning

```promql
# Request growth trend
rate(http_requests_total[1h]) - 
rate(http_requests_total[1h] offset 1w)

# Database connection trend
avg_over_time(database_pool_size{state="used"}[1d])

# Memory usage trend
avg_over_time(process_resident_memory_bytes[1d])
```

---

## Summary

### The Golden Rules of Monitoring

1. **Monitor outcomes, not just outputs**: User experience, not just server stats
2. **Alert on symptoms, not causes**: Slow responses, not high CPU
3. **Keep it simple**: Start with basics, add complexity as needed
4. **Make alerts actionable**: Every alert should have a runbook
5. **Review and iterate**: Adjust thresholds based on experience

### Minimum Viable Monitoring

```typescript
// 1. Install prom-client
npm install prom-client

// 2. Expose /metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});

// 3. Track requests
app.use(metricsMiddleware);

// 4. Run Prometheus + Grafana
docker-compose up -d

// 5. Import dashboard (ID: 11159)
// 6. Create basic alerts
```

### Time Investment

- **Initial setup**: 4-6 hours
- **Dashboard creation**: 2-3 hours
- **Alert configuration**: 2-3 hours
- **Total**: ~10 hours
- **ROI**: Prevents hours/days of debugging, catches issues before users

### Related Documentation
- [Logging Guide](./LOGGING.md) - Structured logging
- [Health Checks](./HEALTH_CHECKS.md) - Health endpoints
- [Alerting Runbooks](./RUNBOOKS.md) - Incident response
- [Performance Optimization](./PERFORMANCE.md) - Using metrics to optimize

---

**Last Updated**: 2026-01-27  
**Owner**: DevOps/SRE Team  
**Review**: Monthly or when alert patterns change