# Thread Pooling & Parallel Execution

---
**Last Updated**: 2026-01-28  
**Status**: 💡 GUIDANCE  
**Priority**: P1 - High Priority  
---

## Overview

Node.js is single-threaded by design, but modern applications need CPU-intensive operations and true parallelism. This guide covers Worker Threads, thread pooling, identifying parallel execution opportunities, and best practices.

---

## Quick Navigation

- [Understanding Node.js Concurrency](#understanding-nodejs-concurrency)
- [Worker Threads](#worker-threads)
- [Thread Pooling](#thread-pooling)
- [Identifying Parallel Execution Opportunities](#identifying-parallel-execution-opportunities)
- [CPU-Intensive Operations](#cpu-intensive-operations)
- [Best Practices](#best-practices)

---

## Understanding Node.js Concurrency

### Event Loop vs True Parallelism

```typescript
// ❌ Misconception: async = parallel
async function processUsers() {
  for (const user of users) {
    await processUser(user);  // Sequential! One at a time
  }
}

// ✅ Concurrent (but still single-threaded)
async function processUsers() {
  await Promise.all(
    users.map(user => processUser(user))  // Concurrent I/O
  );
}

// ✅ True parallelism (multi-threaded)
async function processUsers() {
  const pool = new WorkerPool('./worker.js', 4);
  await Promise.all(
    users.map(user => pool.exec('processUser', user))
  );
}
```

### When You Need True Parallelism

| Operation | Event Loop (Async) | Worker Threads |
|-----------|-------------------|----------------|
| **Database queries** | ✅ Perfect | ❌ Overkill |
| **HTTP requests** | ✅ Perfect | ❌ Overkill |
| **File I/O** | ✅ Perfect | ❌ Overkill |
| **Image processing** | ❌ Blocks event loop | ✅ Use workers |
| **Video encoding** | ❌ Blocks event loop | ✅ Use workers |
| **Cryptography** | ❌ Blocks event loop | ✅ Use workers |
| **Data parsing (large)** | ❌ Blocks event loop | ✅ Use workers |
| **Machine learning** | ❌ Blocks event loop | ✅ Use workers |

---

## Worker Threads

### Basic Worker Thread

```typescript
// worker.ts
import { parentPort, workerData } from 'worker_threads';

// CPU-intensive task
function fibonacci(n: number): number {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
}

// Receive message from main thread
parentPort?.on('message', (n: number) => {
  const result = fibonacci(n);
  
  // Send result back to main thread
  parentPort?.postMessage(result);
});

// Access initial data
console.log('Worker started with data:', workerData);
```

```typescript
// main.ts
import { Worker } from 'worker_threads';

function runWorker(workerData: any): Promise<any> {
  return new Promise((resolve, reject) => {
    const worker = new Worker('./worker.js', { workerData });
    
    worker.on('message', resolve);
    worker.on('error', reject);
    worker.on('exit', (code) => {
      if (code !== 0) {
        reject(new Error(`Worker stopped with exit code ${code}`));
      }
    });
    
    worker.postMessage(40);  // Calculate fibonacci(40)
  });
}

// Usage
const result = await runWorker({ userId: 123 });
console.log('Result:', result);
```

### Worker Thread Pool

```typescript
import { Worker } from 'worker_threads';
import { EventEmitter } from 'events';

interface Task {
  data: any;
  resolve: (value: any) => void;
  reject: (error: any) => void;
}

class WorkerPool extends EventEmitter {
  private workers: Worker[] = [];
  private freeWorkers: Worker[] = [];
  private taskQueue: Task[] = [];
  
  constructor(
    private workerScript: string,
    private poolSize: number = 4
  ) {
    super();
    this.initializePool();
  }
  
  private initializePool() {
    for (let i = 0; i < this.poolSize; i++) {
      this.createWorker();
    }
  }
  
  private createWorker() {
    const worker = new Worker(this.workerScript);
    
    worker.on('message', (result) => {
      this.freeWorkers.push(worker);
      this.processQueue();
    });
    
    worker.on('error', (err) => {
      console.error('Worker error:', err);
      // Remove failed worker and create new one
      const index = this.workers.indexOf(worker);
      if (index > -1) {
        this.workers.splice(index, 1);
      }
      this.createWorker();
    });
    
    this.workers.push(worker);
    this.freeWorkers.push(worker);
  }
  
  exec(data: any): Promise<any> {
    return new Promise((resolve, reject) => {
      this.taskQueue.push({ data, resolve, reject });
      this.processQueue();
    });
  }
  
  private processQueue() {
    while (this.taskQueue.length > 0 && this.freeWorkers.length > 0) {
      const task = this.taskQueue.shift()!;
      const worker = this.freeWorkers.shift()!;
      
      // Set up one-time listeners for this task
      const onMessage = (result: any) => {
        task.resolve(result);
        worker.removeListener('message', onMessage);
        worker.removeListener('error', onError);
      };
      
      const onError = (err: Error) => {
        task.reject(err);
        worker.removeListener('message', onMessage);
        worker.removeListener('error', onError);
      };
      
      worker.once('message', onMessage);
      worker.once('error', onError);
      
      worker.postMessage(task.data);
    }
  }
  
  async destroy() {
    await Promise.all(
      this.workers.map(worker => worker.terminate())
    );
    this.workers = [];
    this.freeWorkers = [];
  }
}

// Usage
const pool = new WorkerPool('./worker.js', 4);

// Process 100 tasks in parallel (max 4 at a time)
const results = await Promise.all(
  Array(100).fill(null).map((_, i) => pool.exec(i))
);

await pool.destroy();
```

---

## Thread Pooling

### Using Piscina (Production-Ready Pool)

```typescript
import Piscina from 'piscina';

// Create thread pool
const pool = new Piscina({
  filename: './worker.js',
  minThreads: 2,
  maxThreads: 8,
  idleTimeout: 30000,  // Terminate idle threads after 30s
  maxQueue: 1000  // Max queued tasks
});

// Execute task
const result = await pool.run({ n: 40 });

// Execute with specific function
const result = await pool.run(
  { data: imageBuffer },
  { name: 'processImage' }  // Call specific exported function
);

// Graceful shutdown
await pool.destroy();
```

```typescript
// worker.js
module.exports = {
  // Default export
  default: async (data) => {
    return fibonacci(data.n);
  },
  
  // Named exports
  processImage: async (data) => {
    // Image processing logic
    return processedImage;
  },
  
  encodeVideo: async (data) => {
    // Video encoding logic
    return encodedVideo;
  }
};
```

### Dynamic Pool Sizing

```typescript
import os from 'os';

class AdaptiveWorkerPool {
  private pool: Piscina;
  private currentLoad = 0;
  
  constructor() {
    const cpuCount = os.cpus().length;
    
    this.pool = new Piscina({
      filename: './worker.js',
      minThreads: Math.max(2, Math.floor(cpuCount / 2)),
      maxThreads: cpuCount,
      idleTimeout: 30000
    });
    
    // Monitor and adjust
    setInterval(() => this.adjustPoolSize(), 10000);
  }
  
  private adjustPoolSize() {
    const queueSize = this.pool.queueSize;
    const utilization = this.pool.utilization;
    
    if (queueSize > 100 && utilization > 0.8) {
      // High load - increase threads
      this.pool.maxThreads = Math.min(
        os.cpus().length * 2,
        this.pool.maxThreads + 2
      );
    } else if (queueSize === 0 && utilization < 0.2) {
      // Low load - decrease threads
      this.pool.maxThreads = Math.max(
        2,
        this.pool.maxThreads - 1
      );
    }
  }
  
  async exec(data: any) {
    this.currentLoad++;
    try {
      return await this.pool.run(data);
    } finally {
      this.currentLoad--;
    }
  }
}
```

---

## Identifying Parallel Execution Opportunities

### 1. Profile Your Application

```typescript
import { performance } from 'perf_hooks';

// Identify slow operations
async function profileOperation(name: string, fn: () => Promise<void>) {
  const start = performance.now();
  await fn();
  const duration = performance.now() - start;
  
  console.log(`${name}: ${duration.toFixed(2)}ms`);
  
  if (duration > 100) {
    console.warn(`⚠️  ${name} is slow - consider parallelization`);
  }
}

// Usage
await profileOperation('Process users', async () => {
  for (const user of users) {
    await processUser(user);
  }
});
```

### 2. Analyze CPU Usage

```typescript
import os from 'os';

setInterval(() => {
  const cpus = os.cpus();
  const usage = cpus.map((cpu, i) => {
    const total = Object.values(cpu.times).reduce((a, b) => a + b);
    const idle = cpu.times.idle;
    const used = ((total - idle) / total) * 100;
    return { core: i, usage: used.toFixed(2) };
  });
  
  console.log('CPU Usage:', usage);
  
  // If one core is maxed out, you're CPU-bound
  const maxUsage = Math.max(...usage.map(u => parseFloat(u.usage)));
  if (maxUsage > 90) {
    console.warn('⚠️  CPU-bound - consider worker threads');
  }
}, 5000);
```

### 3. Detect Event Loop Blocking

```typescript
import { monitorEventLoopDelay } from 'perf_hooks';

const h = monitorEventLoopDelay({ resolution: 20 });
h.enable();

setInterval(() => {
  const delay = h.mean / 1000000;  // Convert to ms
  
  console.log(`Event loop delay: ${delay.toFixed(2)}ms`);
  
  if (delay > 50) {
    console.error('⚠️  Event loop blocked - CPU-intensive task detected!');
  }
  
  h.reset();
}, 10000);
```

### 4. Parallelization Decision Tree

```
Is the operation CPU-intensive? (image processing, crypto, parsing)
├─ YES → Use Worker Threads
└─ NO → Is it I/O-bound? (database, HTTP, file)
    ├─ YES → Use async/await with Promise.all
    └─ NO → Keep sequential
```

---

## CPU-Intensive Operations

### Image Processing

```typescript
// worker.js
import sharp from 'sharp';

module.exports = async (imageBuffer: Buffer) => {
  return await sharp(imageBuffer)
    .resize(800, 600)
    .jpeg({ quality: 80 })
    .toBuffer();
};
```

```typescript
// main.ts
import Piscina from 'piscina';

const pool = new Piscina({ filename: './image-worker.js' });

app.post('/upload', upload.single('image'), async (req, res) => {
  // Process image in worker thread (doesn't block event loop)
  const processed = await pool.run(req.file.buffer);
  
  await saveToS3(processed);
  res.json({ success: true });
});
```

### Data Parsing

```typescript
// worker.js
import Papa from 'papaparse';

module.exports = (csvData: string) => {
  return Papa.parse(csvData, {
    header: true,
    dynamicTyping: true,
    skipEmptyLines: true
  }).data;
};
```

```typescript
// main.ts
app.post('/import-csv', async (req, res) => {
  const csvData = req.body.csv;
  
  // Parse large CSV in worker thread
  const parsed = await pool.run(csvData);
  
  // Insert into database
  await db.batchInsert('users', parsed);
  
  res.json({ imported: parsed.length });
});
```

### Cryptography

```typescript
// worker.js
import bcrypt from 'bcrypt';

module.exports = {
  hashPassword: async (password: string) => {
    return await bcrypt.hash(password, 12);
  },
  
  verifyPassword: async ({ password, hash }: any) => {
    return await bcrypt.compare(password, hash);
  }
};
```

```typescript
// main.ts
app.post('/register', async (req, res) => {
  // Hash password in worker thread (bcrypt is CPU-intensive)
  const hashedPassword = await pool.run(
    req.body.password,
    { name: 'hashPassword' }
  );
  
  await db.createUser({ email: req.body.email, password: hashedPassword });
  res.json({ success: true });
});
```

---

## Best Practices

### 1. Don't Overuse Workers

```typescript
// ❌ Bad: Worker overhead > task duration
await pool.run({ a: 1, b: 2 });  // Simple addition

// ✅ Good: Task duration > worker overhead
await pool.run(largeImageBuffer);  // Image processing
```

### 2. Batch Small Tasks

```typescript
// ❌ Bad: Create worker for each small task
for (const item of items) {
  await pool.run(item);  // 1000 worker calls
}

// ✅ Good: Batch tasks
const batchSize = 100;
for (let i = 0; i < items.length; i += batchSize) {
  const batch = items.slice(i, i + batchSize);
  await pool.run(batch);  // 10 worker calls
}
```

### 3. Handle Worker Errors

```typescript
try {
  const result = await pool.run(data);
} catch (err) {
  if (err.message.includes('Worker terminated')) {
    // Worker crashed - retry or handle gracefully
    logger.error('Worker crashed', { error: err });
  } else {
    // Task failed - handle error
    logger.error('Task failed', { error: err });
  }
}
```

### 4. Monitor Pool Metrics

```typescript
import { Gauge } from 'prom-client';

const workerPoolSize = new Gauge({
  name: 'worker_pool_size',
  help: 'Number of worker threads',
  labelNames: ['state']
});

const workerQueueSize = new Gauge({
  name: 'worker_queue_size',
  help: 'Number of queued tasks'
});

setInterval(() => {
  workerPoolSize.set({ state: 'total' }, pool.threads.length);
  workerPoolSize.set({ state: 'active' }, pool.utilization * pool.threads.length);
  workerQueueSize.set(pool.queueSize);
}, 5000);
```

### 5. Graceful Shutdown

```typescript
process.on('SIGTERM', async () => {
  logger.info('SIGTERM received, shutting down worker pool...');
  
  // Stop accepting new tasks
  pool.maxQueue = 0;
  
  // Wait for current tasks to complete (max 30s)
  const timeout = setTimeout(() => {
    logger.warn('Force terminating worker pool');
    pool.destroy();
  }, 30000);
  
  await pool.destroy();
  clearTimeout(timeout);
  
  logger.info('Worker pool shut down gracefully');
  process.exit(0);
});
```

---

## Performance Comparison

### Benchmark: Image Processing

```typescript
// Sequential (single-threaded)
const start = Date.now();
for (const image of images) {
  await processImage(image);
}
console.log(`Sequential: ${Date.now() - start}ms`);  // 10000ms

// Concurrent (async, still single-threaded)
const start = Date.now();
await Promise.all(images.map(processImage));
console.log(`Concurrent: ${Date.now() - start}ms`);  // 10000ms (no improvement!)

// Parallel (worker threads)
const start = Date.now();
await Promise.all(images.map(img => pool.run(img)));
console.log(`Parallel: ${Date.now() - start}ms`);  // 2500ms (4x faster on 4 cores!)
```

---

## Common Pitfalls

### 1. Sharing Memory

```typescript
// ❌ Bad: Can't share objects directly
const sharedData = { count: 0 };
await pool.run(sharedData);  // Serialized, not shared!

// ✅ Good: Use SharedArrayBuffer for true shared memory
const sharedBuffer = new SharedArrayBuffer(4);
const sharedArray = new Int32Array(sharedBuffer);
await pool.run(sharedBuffer);  // Actually shared
```

### 2. Worker Startup Cost

```typescript
// ❌ Bad: Create worker per request
app.post('/process', async (req, res) => {
  const worker = new Worker('./worker.js');  // Slow!
  // ...
});

// ✅ Good: Reuse worker pool
const pool = new Piscina({ filename: './worker.js' });
app.post('/process', async (req, res) => {
  await pool.run(req.body);  // Fast!
});
```

### 3. Not Handling Backpressure

```typescript
// ❌ Bad: Unlimited queue
for (const item of millionItems) {
  pool.run(item);  // Memory explosion!
}

// ✅ Good: Limit concurrent tasks
import pLimit from 'p-limit';

const limit = pLimit(100);  // Max 100 concurrent
await Promise.all(
  millionItems.map(item => limit(() => pool.run(item)))
);
```

---

## See Also

- [Performance & Scalability](PERFORMANCE_SCALABILITY.md) - Overall performance optimization
- [Node.js Cluster Mode](PERFORMANCE_SCALABILITY.md#nodejs-cluster-mode) - Process-level parallelism
- [Request Queuing](PERFORMANCE_SCALABILITY.md#request-queuing) - Background job processing
- [Monitoring](prometheus.md) - Track worker pool metrics
