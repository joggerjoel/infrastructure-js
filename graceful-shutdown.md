# Graceful Shutdown & Resource Cleanup

---
**Status**: 💡 **GUIDANCE** - Best practices for graceful shutdown implementation  
**Priority**: 🔴 **P0.5** - Critical before production launch  
**Last Updated**: 2026-01-28  
**Owned By**: Backend Team, DevOps  

---

## Overview

Proper shutdown handling ensures that when the application terminates (via Ctrl+C, kill signal, or crash), all resources are cleaned up gracefully. This prevents:
- Database connection leaks
- Incomplete transactions
- Orphaned background jobs
- File descriptor leaks
- Memory leaks
- Corrupted data

---

## Signal Handling in Node.js

### Common Termination Signals

| Signal | Source | Default Behavior | Should Handle? |
|--------|--------|------------------|----------------|
| `SIGINT` | Ctrl+C in terminal | Terminate | ✅ Yes |
| `SIGTERM` | `kill <pid>` or orchestrator | Terminate | ✅ Yes |
| `SIGKILL` | `kill -9 <pid>` | Force terminate | ❌ Cannot catch |
| `SIGHUP` | Terminal closed | Terminate | ⚠️ Optional |
| `uncaughtException` | Unhandled error | Crash | ✅ Yes |
| `unhandledRejection` | Unhandled promise rejection | Crash | ✅ Yes |

### Basic Signal Handler Pattern

```typescript
// server.ts
import express from 'express';
import { createServer } from 'http';

const app = express();
const server = createServer(app);
const PORT = process.env.PORT || 3000;

// Start server
server.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});

// Graceful shutdown handler
async function gracefulShutdown(signal: string) {
  console.log(`\n${signal} received, starting graceful shutdown...`);
  
  // Stop accepting new connections
  server.close(() => {
    console.log('HTTP server closed');
  });
  
  // Clean up resources (detailed below)
  await cleanupResources();
  
  console.log('Graceful shutdown complete');
  process.exit(0);
}

// Register signal handlers
process.on('SIGINT', () => gracefulShutdown('SIGINT'));
process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));

// Handle uncaught errors
process.on('uncaughtException', (error) => {
  console.error('Uncaught Exception:', error);
  gracefulShutdown('uncaughtException');
});

process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled Rejection at:', promise, 'reason:', reason);
  gracefulShutdown('unhandledRejection');
});
```

---

## Resource Cleanup Checklist

### 1. HTTP Server Connections

**Problem**: Active HTTP connections prevent clean shutdown  
**Solution**: Stop accepting new connections, drain existing ones

```typescript
import { Server } from 'http';

async function closeHttpServer(server: Server): Promise<void> {
  return new Promise((resolve, reject) => {
    // Stop accepting new connections
    server.close((err) => {
      if (err) {
        console.error('Error closing HTTP server:', err);
        reject(err);
      } else {
        console.log('HTTP server closed successfully');
        resolve();
      }
    });
  });
}

// With timeout
async function closeHttpServerWithTimeout(
  server: Server, 
  timeoutMs: number = 10000
): Promise<void> {
  return new Promise((resolve, reject) => {
    const timeout = setTimeout(() => {
      console.warn(`Force closing HTTP server after ${timeoutMs}ms timeout`);
      resolve(); // Resolve anyway to allow shutdown
    }, timeoutMs);

    server.close((err) => {
      clearTimeout(timeout);
      if (err) {
        console.error('Error closing HTTP server:', err);
        reject(err);
      } else {
        console.log('HTTP server closed successfully');
        resolve();
      }
    });
  });
}
```

### 2. Database Connections

**Problem**: Open database connections can cause connection pool exhaustion  
**Solution**: Close all connection pools gracefully

#### PostgreSQL (pg/Knex)

```typescript
import Knex from 'knex';

const db = Knex({
  client: 'postgresql',
  connection: process.env.DATABASE_URL,
  pool: {
    min: 2,
    max: 10
  }
});

async function closeDatabaseConnections(): Promise<void> {
  try {
    console.log('Closing database connections...');
    await db.destroy();
    console.log('Database connections closed');
  } catch (error) {
    console.error('Error closing database:', error);
    throw error;
  }
}
```

#### With Multiple Connection Pools

```typescript
import { Pool } from 'pg';

const readPool = new Pool({ /* config */ });
const writePool = new Pool({ /* config */ });

async function closeAllDatabasePools(): Promise<void> {
  console.log('Closing database pools...');
  
  await Promise.all([
    readPool.end().then(() => console.log('Read pool closed')),
    writePool.end().then(() => console.log('Write pool closed'))
  ]);
  
  console.log('All database pools closed');
}
```

### 3. Redis Connections

**Problem**: Redis connections stay open indefinitely  
**Solution**: Quit Redis clients properly

```typescript
import Redis from 'ioredis';

const redis = new Redis({
  host: process.env.REDIS_HOST,
  port: 6379
});

async function closeRedisConnection(): Promise<void> {
  try {
    console.log('Closing Redis connection...');
    await redis.quit(); // Graceful close, waits for pending commands
    console.log('Redis connection closed');
  } catch (error) {
    console.error('Error closing Redis:', error);
    // If graceful quit fails, force disconnect
    redis.disconnect();
  }
}

// For multiple Redis clients
const redisCache = new Redis(/* config */);
const redisPubSub = new Redis(/* config */);

async function closeAllRedisConnections(): Promise<void> {
  console.log('Closing Redis connections...');
  
  await Promise.all([
    redisCache.quit().catch(() => redisCache.disconnect()),
    redisPubSub.quit().catch(() => redisPubSub.disconnect())
  ]);
  
  console.log('All Redis connections closed');
}
```

### 4. Background Job Queues

**Problem**: Jobs in progress get interrupted  
**Solution**: Gracefully close job queues, finish or requeue active jobs

#### Bull Queue

```typescript
import Queue from 'bull';

const emailQueue = new Queue('email', {
  redis: {
    host: process.env.REDIS_HOST,
    port: 6379
  }
});

async function closeJobQueues(): Promise<void> {
  console.log('Closing job queues...');
  
  // Stop processing new jobs
  await emailQueue.pause(true, true); // Pause local and global
  
  // Wait for active jobs to complete (with timeout)
  const activeJobs = await emailQueue.getActive();
  if (activeJobs.length > 0) {
    console.log(`Waiting for ${activeJobs.length} active jobs to complete...`);
    
    // Wait up to 30 seconds for jobs to finish
    const timeout = 30000;
    const start = Date.now();
    
    while ((await emailQueue.getActive()).length > 0) {
      if (Date.now() - start > timeout) {
        console.warn('Timeout waiting for jobs, forcing close');
        break;
      }
      await new Promise(resolve => setTimeout(resolve, 1000));
    }
  }
  
  // Close the queue
  await emailQueue.close();
  console.log('Job queues closed');
}
```

#### BullMQ (newer version)

```typescript
import { Queue, Worker } from 'bullmq';

const queue = new Queue('tasks');
const worker = new Worker('tasks', async (job) => {
  // Job processing
});

async function closeBullMQResources(): Promise<void> {
  console.log('Closing BullMQ resources...');
  
  // Close worker (stops processing new jobs)
  await worker.close();
  console.log('Worker closed');
  
  // Close queue
  await queue.close();
  console.log('Queue closed');
}
```

### 5. WebSocket Connections

**Problem**: Active WebSocket connections prevent shutdown  
**Solution**: Close all WebSocket connections gracefully

```typescript
import { WebSocketServer } from 'ws';

const wss = new WebSocketServer({ server });

async function closeWebSocketConnections(): Promise<void> {
  console.log('Closing WebSocket connections...');
  
  return new Promise((resolve) => {
    // Close all client connections
    wss.clients.forEach((client) => {
      client.close(1001, 'Server shutting down');
    });
    
    // Close the WebSocket server
    wss.close(() => {
      console.log('WebSocket server closed');
      resolve();
    });
  });
}
```

### 6. File Handles & Streams

**Problem**: Open file handles can corrupt data  
**Solution**: Close all streams and file handles

```typescript
import * as fs from 'fs';
import { pipeline } from 'stream/promises';

const activeStreams: Set<NodeJS.WritableStream> = new Set();

// Track streams
function createTrackedWriteStream(path: string): fs.WriteStream {
  const stream = fs.createWriteStream(path);
  activeStreams.add(stream);
  
  stream.on('close', () => {
    activeStreams.delete(stream);
  });
  
  return stream;
}

async function closeAllStreams(): Promise<void> {
  console.log(`Closing ${activeStreams.size} active streams...`);
  
  const closePromises = Array.from(activeStreams).map(stream => {
    return new Promise<void>((resolve) => {
      if (stream.closed) {
        resolve();
      } else {
        stream.end(() => resolve());
      }
    });
  });
  
  await Promise.all(closePromises);
  console.log('All streams closed');
}
```

### 7. Timers & Intervals

**Problem**: Active timers prevent process from exiting  
**Solution**: Clear all timers and intervals

```typescript
// Track timers
const activeTimers: Set<NodeJS.Timeout> = new Set();
const activeIntervals: Set<NodeJS.Timeout> = new Set();

// Wrapper functions
function trackedSetTimeout(callback: () => void, ms: number): NodeJS.Timeout {
  const timer = setTimeout(() => {
    activeTimers.delete(timer);
    callback();
  }, ms);
  activeTimers.add(timer);
  return timer;
}

function trackedSetInterval(callback: () => void, ms: number): NodeJS.Timeout {
  const interval = setInterval(callback, ms);
  activeIntervals.add(interval);
  return interval;
}

function clearAllTimers(): void {
  console.log(`Clearing ${activeTimers.size} timers and ${activeIntervals.size} intervals...`);
  
  activeTimers.forEach(timer => clearTimeout(timer));
  activeIntervals.forEach(interval => clearInterval(interval));
  
  activeTimers.clear();
  activeIntervals.clear();
  
  console.log('All timers cleared');
}
```

### 8. External Service Connections

**Problem**: Third-party client libraries may hold connections  
**Solution**: Close clients explicitly

```typescript
import AWS from 'aws-sdk';
import { Client as ElasticsearchClient } from '@elastic/elasticsearch';

const s3 = new AWS.S3();
const esClient = new ElasticsearchClient({ node: 'http://localhost:9200' });

async function closeExternalConnections(): Promise<void> {
  console.log('Closing external service connections...');
  
  // Elasticsearch
  await esClient.close();
  console.log('Elasticsearch connection closed');
  
  // AWS SDK clients don't need explicit cleanup
  // But you can destroy HTTP agents if needed
  if (s3.config.httpOptions?.agent) {
    s3.config.httpOptions.agent.destroy();
  }
  
  console.log('External connections closed');
}
```

---

## Complete Graceful Shutdown Implementation

### Full Example with All Resources

```typescript
// server.ts
import express from 'express';
import { createServer } from 'http';
import Knex from 'knex';
import Redis from 'ioredis';
import Queue from 'bull';
import { logger } from './logger';

// Initialize resources
const app = express();
const server = createServer(app);
const db = Knex({ /* config */ });
const redis = new Redis({ /* config */ });
const emailQueue = new Queue('email', { redis: { /* config */ } });

// Track shutdown state
let isShuttingDown = false;

// Health check endpoint
app.get('/health', (req, res) => {
  if (isShuttingDown) {
    res.status(503).json({ status: 'shutting down' });
  } else {
    res.json({ status: 'healthy' });
  }
});

// Graceful shutdown handler
async function gracefulShutdown(signal: string): Promise<void> {
  if (isShuttingDown) {
    logger.warn('Shutdown already in progress, ignoring signal');
    return;
  }
  
  isShuttingDown = true;
  logger.info(`${signal} received, starting graceful shutdown...`);
  
  const shutdownTimeout = 30000; // 30 seconds
  const shutdownTimer = setTimeout(() => {
    logger.error('Graceful shutdown timeout, forcing exit');
    process.exit(1);
  }, shutdownTimeout);
  
  try {
    // 1. Stop accepting new HTTP connections
    logger.info('Closing HTTP server...');
    await closeHttpServer(server);
    
    // 2. Close background job queues
    logger.info('Closing job queues...');
    await emailQueue.close();
    
    // 3. Close Redis connections
    logger.info('Closing Redis...');
    await redis.quit();
    
    // 4. Close database connections
    logger.info('Closing database...');
    await db.destroy();
    
    // 5. Any other cleanup
    logger.info('Performing final cleanup...');
    // ... additional cleanup
    
    clearTimeout(shutdownTimer);
    logger.info('Graceful shutdown complete');
    process.exit(0);
    
  } catch (error) {
    clearTimeout(shutdownTimer);
    logger.error('Error during graceful shutdown:', error);
    process.exit(1);
  }
}

// Helper: Close HTTP server with timeout
function closeHttpServer(server: Server, timeout = 10000): Promise<void> {
  return new Promise((resolve, reject) => {
    const timer = setTimeout(() => {
      logger.warn('HTTP server close timeout, forcing');
      resolve();
    }, timeout);
    
    server.close((err) => {
      clearTimeout(timer);
      if (err) reject(err);
      else resolve();
    });
  });
}

// Register signal handlers
process.on('SIGINT', () => gracefulShutdown('SIGINT'));
process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));

// Handle uncaught errors
process.on('uncaughtException', (error) => {
  logger.error('Uncaught Exception:', error);
  gracefulShutdown('uncaughtException');
});

process.on('unhandledRejection', (reason, promise) => {
  logger.error('Unhandled Rejection:', { reason, promise });
  gracefulShutdown('unhandledRejection');
});

// Start server
server.listen(3000, () => {
  logger.info('Server started on port 3000');
});
```

### Class-Based Resource Manager

For more complex applications, use a resource manager:

```typescript
// resource-manager.ts
import { logger } from './logger';

type CleanupFunction = () => Promise<void> | void;

class ResourceManager {
  private resources: Map<string, CleanupFunction> = new Map();
  private isShuttingDown = false;
  
  // Register a cleanup function
  register(name: string, cleanup: CleanupFunction): void {
    if (this.resources.has(name)) {
      logger.warn(`Resource ${name} already registered, overwriting`);
    }
    this.resources.set(name, cleanup);
    logger.debug(`Registered cleanup for ${name}`);
  }
  
  // Unregister a cleanup function
  unregister(name: string): void {
    this.resources.delete(name);
    logger.debug(`Unregistered cleanup for ${name}`);
  }
  
  // Clean up all resources
  async cleanup(): Promise<void> {
    if (this.isShuttingDown) {
      logger.warn('Cleanup already in progress');
      return;
    }
    
    this.isShuttingDown = true;
    logger.info(`Cleaning up ${this.resources.size} resources...`);
    
    const results = await Promise.allSettled(
      Array.from(this.resources.entries()).map(async ([name, cleanup]) => {
        try {
          logger.debug(`Cleaning up ${name}...`);
          await cleanup();
          logger.debug(`${name} cleaned up successfully`);
        } catch (error) {
          logger.error(`Error cleaning up ${name}:`, error);
          throw error;
        }
      })
    );
    
    const failures = results.filter(r => r.status === 'rejected');
    if (failures.length > 0) {
      logger.error(`${failures.length} resources failed to clean up`);
    } else {
      logger.info('All resources cleaned up successfully');
    }
  }
  
  // Check if shutdown is in progress
  isShutdownInProgress(): boolean {
    return this.isShuttingDown;
  }
}

// Singleton instance
export const resourceManager = new ResourceManager();

// Usage:
// resourceManager.register('database', () => db.destroy());
// resourceManager.register('redis', () => redis.quit());
```

### Using Resource Manager in Application

```typescript
// app.ts
import { resourceManager } from './resource-manager';
import { logger } from './logger';

// Register all resources
resourceManager.register('http-server', () => closeHttpServer(server));
resourceManager.register('database', () => db.destroy());
resourceManager.register('redis', () => redis.quit());
resourceManager.register('job-queue', () => emailQueue.close());

// Graceful shutdown
async function gracefulShutdown(signal: string): Promise<void> {
  logger.info(`${signal} received, starting graceful shutdown...`);
  
  try {
    await resourceManager.cleanup();
    logger.info('Graceful shutdown complete');
    process.exit(0);
  } catch (error) {
    logger.error('Error during shutdown:', error);
    process.exit(1);
  }
}

// Register handlers
process.on('SIGINT', () => gracefulShutdown('SIGINT'));
process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
```

---

## Testing Graceful Shutdown

### Manual Testing

```bash
# Start server
npm start

# In another terminal, send SIGINT
kill -SIGINT $(pgrep -f "node.*server")

# Or just Ctrl+C in the server terminal

# Check logs for cleanup messages
tail -f logs/backend-current.log | grep -i shutdown
```

### Automated Testing

```typescript
// tests/graceful-shutdown.test.ts
import { spawn } from 'child_process';
import { describe, it, expect } from '@jest/globals';

describe('Graceful Shutdown', () => {
  it('should handle SIGTERM gracefully', async () => {
    const server = spawn('node', ['dist/server.js']);
    
    // Wait for server to start
    await new Promise(resolve => setTimeout(resolve, 2000));
    
    // Send SIGTERM
    server.kill('SIGTERM');
    
    // Capture output
    let output = '';
    server.stdout.on('data', (data) => {
      output += data.toString();
    });
    
    // Wait for shutdown
    await new Promise(resolve => {
      server.on('exit', resolve);
    });
    
    // Verify cleanup messages
    expect(output).toContain('Closing HTTP server');
    expect(output).toContain('Closing database');
    expect(output).toContain('Graceful shutdown complete');
  });
  
  it('should close database connections', async () => {
    // Test that db connections are actually closed
    // by checking connection count before/after
  });
});
```

---

## Docker & Kubernetes Considerations

### Dockerfile Best Practices

```dockerfile
# Use SIGTERM-aware entry point
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .

# Use node directly (not npm) for proper signal handling
CMD ["node", "dist/server.js"]

# Don't use: CMD ["npm", "start"]
# npm doesn't forward signals properly!
```

### Kubernetes Pod Spec

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: api-server
spec:
  containers:
  - name: api
    image: myapp:latest
    
    # Graceful shutdown period
    terminationGracePeriodSeconds: 30
    
    # Lifecycle hooks
    lifecycle:
      preStop:
        exec:
          # Optional: trigger custom shutdown logic
          command: ["/bin/sh", "-c", "sleep 5"]
    
    # Health checks
    livenessProbe:
      httpGet:
        path: /health
        port: 3000
      initialDelaySeconds: 10
      periodSeconds: 10
    
    readinessProbe:
      httpGet:
        path: /health
        port: 3000
      initialDelaySeconds: 5
      periodSeconds: 5
```

### Process Manager (PM2) Configuration

```javascript
// ecosystem.config.js
module.exports = {
  apps: [{
    name: 'api-server',
    script: './dist/server.js',
    instances: 4,
    exec_mode: 'cluster',
    
    // Graceful shutdown
    kill_timeout: 10000, // Wait 10s for graceful shutdown
    wait_ready: true, // Wait for ready signal
    listen_timeout: 10000,
    
    // Signal handling
    shutdown_with_message: true
  }]
};
```

---

## Common Pitfalls & Solutions

### Problem 1: Process Won't Exit

**Symptom**: Process hangs after Ctrl+C

**Causes**:
- Active timers/intervals not cleared
- Open database connections
- Background threads still running
- Event listeners not removed

**Solution**:
```typescript
// Add force exit timeout
const shutdownTimeout = setTimeout(() => {
  logger.error('Graceful shutdown timeout, forcing exit');
  process.exit(1);
}, 30000); // 30 seconds

// Clear timeout after successful cleanup
clearTimeout(shutdownTimeout);
```

### Problem 2: Data Corruption on Shutdown

**Symptom**: Incomplete transactions, corrupted files

**Causes**:
- Not waiting for in-flight transactions
- Force-closing database before commits complete
- Streams not flushed

**Solution**:
```typescript
// Wait for active transactions
await db.transaction.commit();

// Flush streams before closing
await new Promise((resolve) => {
  logStream.end(() => resolve());
});
```

### Problem 3: Duplicate Shutdown Handlers

**Symptom**: Cleanup runs multiple times

**Causes**:
- Multiple signal handlers registered
- Not tracking shutdown state

**Solution**:
```typescript
let isShuttingDown = false;

async function gracefulShutdown(signal: string) {
  if (isShuttingDown) {
    logger.warn('Shutdown already in progress');
    return;
  }
  isShuttingDown = true;
  // ... cleanup
}
```

### Problem 4: NPM Doesn't Forward Signals

**Symptom**: `npm start` doesn't respond to Ctrl+C properly

**Cause**: npm doesn't forward SIGTERM/SIGINT to child process

**Solution**:
```json
// package.json - DON'T do this in production:
{
  "scripts": {
    "start": "npm run build && node dist/server.js"
  }
}

// DO this instead:
{
  "scripts": {
    "start": "node dist/server.js"
  }
}
```

Or use this in development:

```json
{
  "scripts": {
    "dev": "tsx watch src/server.ts"
  }
}
```

---

## Checklist: Graceful Shutdown Readiness

### Pre-Production Checklist

- [ ] SIGINT handler registered (Ctrl+C)
- [ ] SIGTERM handler registered (kill)
- [ ] uncaughtException handler registered
- [ ] unhandledRejection handler registered
- [ ] HTTP server closes gracefully
- [ ] Database connections close
- [ ] Redis connections close
- [ ] Job queues stop and drain
- [ ] WebSocket connections close
- [ ] File streams flush and close
- [ ] Timers/intervals cleared
- [ ] External service clients close
- [ ] Shutdown timeout prevents hang (30s max)
- [ ] Duplicate shutdown prevented
- [ ] Health endpoint returns 503 during shutdown
- [ ] Logging shows all cleanup steps
- [ ] Tested manually with Ctrl+C
- [ ] Tested with `kill -SIGTERM`
- [ ] Tested in Docker container
- [ ] Works with process manager (PM2)

### Production Validation

```bash
# Test 1: Normal shutdown
curl http://localhost:3000/health  # Should return 200
kill -SIGTERM $(pgrep node)
# Should see cleanup logs
# Process should exit with code 0

# Test 2: Force shutdown timeout
curl http://localhost:3000/health
kill -SIGTERM $(pgrep node)
# Should force exit after timeout if cleanup hangs

# Test 3: During active requests
ab -n 1000 -c 10 http://localhost:3000/api/endpoint &
sleep 2
kill -SIGTERM $(pgrep node)
# Should finish active requests or timeout gracefully
```

---

## Monitoring Shutdown Metrics

### Key Metrics to Track

```typescript
import { logger } from './logger';

async function gracefulShutdown(signal: string): Promise<void> {
  const shutdownStart = Date.now();
  
  try {
    await resourceManager.cleanup();
    
    const shutdownDuration = Date.now() - shutdownStart;
    
    logger.info('Shutdown metrics', {
      signal,
      duration_ms: shutdownDuration,
      success: true
    });
    
  } catch (error) {
    const shutdownDuration = Date.now() - shutdownStart;
    
    logger.error('Shutdown failed', {
      signal,
      duration_ms: shutdownDuration,
      success: false,
      error: error.message
    });
  }
}
```

### Alerting on Shutdown Issues

- Alert if shutdown takes > 25s (approaching 30s timeout)
- Alert if shutdown fails (error during cleanup)
- Alert if force-exit triggered (timeout reached)
- Track shutdown frequency (frequent restarts may indicate issues)

---

## Summary

### Why Graceful Shutdown Matters

✅ **Data Integrity**: Prevents corrupted transactions  
✅ **Resource Cleanup**: Prevents connection/memory leaks  
✅ **Zero-Downtime Deploys**: Enables rolling deployments  
✅ **Debugging**: Clear logs show what happened  
✅ **Professionalism**: Production-ready behavior

### The 5-Minute Implementation

Minimum viable graceful shutdown:

```typescript
// Minimum code to add to server.ts
process.on('SIGTERM', async () => {
  console.log('SIGTERM received, closing server...');
  server.close(() => {
    console.log('Server closed');
    process.exit(0);
  });
});
```

### The Production Implementation

Full graceful shutdown (15-30 minutes to implement):
- Resource manager pattern
- Timeout protection
- Multiple signal handlers
- Comprehensive logging
- Health check during shutdown
- Tested in Docker/K8s

**Start with the minimum, evolve to production implementation before launch.**

---

## Related Documentation

- [Logging Configuration](./LOGGING.md) - Ensure logs capture shutdown events
- [Deployment Runbook](./DEPLOYMENT.md) - Rolling deployment procedures
- [Incident Response](./INCIDENT_RESPONSE.md) - Handling shutdown failures
- [Monitoring Dashboard](./MONITORING.md) - Track shutdown metrics

---

**Last Updated**: 2026-01-27  
**Owner**: Infrastructure Team  
**Review**: Quarterly or after shutdown incidents