# Intelligent Caching Strategy
## Multi-Tier Caching, Smart Invalidation & Performance Optimization

---
**Status**: 💡 **GUIDANCE** - Best practices for intelligent caching implementation  
**Priority**: 🔴 **P0** - Critical for production performance  
**Last Updated**: 2026-01-28  
**Owned By**: Backend Team, DevOps  

---

## Table of Contents

1. [Overview](#overview)
2. [Cache Architecture](#cache-architecture)
3. [Caching Patterns](#caching-patterns)
4. [Smart Invalidation](#smart-invalidation)
5. [Advanced Techniques](#advanced-techniques)
6. [Implementation Examples](#implementation-examples)
7. [Monitoring & Metrics](#monitoring--metrics)
8. [Decision Framework](#decision-framework)
9. [Anti-Patterns](#anti-patterns)
10. [Testing & Benchmarks](#testing--benchmarks)

---

## Overview

### What is Intelligent Caching?

Intelligent caching goes beyond simple key-value storage with TTL. It includes:

- **Multi-tier cache hierarchy** (memory → Redis → database)
- **Smart invalidation** (event-driven, version-based, dependency graphs)
- **Adaptive TTL** (adjusts based on access patterns)
- **Predictive warming** (pre-populate before traffic spikes)
- **Thundering herd prevention** (request coalescing, distributed locks)
- **Geo-distributed caching** (edge locations with eventual consistency)

### Why It Matters

```
Without Intelligent Caching:
- Database: 100-500ms per query
- 1000 requests/sec = 100-500 seconds of DB time
- Result: Database overload, slow responses

With Intelligent Caching:
- Cache hit: 1-5ms
- Cache miss + DB: 100-500ms
- 95% hit rate = 5% miss rate
- Result: 5-50 seconds of DB time (10-100x improvement)
```

### Key Metrics

| Metric | Target | Good | Needs Work |
|--------|--------|------|------------|
| **Cache Hit Rate** | > 90% | 80-90% | < 80% |
| **Latency P50** | < 5ms | 5-10ms | > 10ms |
| **Latency P99** | < 20ms | 20-50ms | > 50ms |
| **Memory Efficiency** | < 70% used | 70-85% | > 85% |
| **Eviction Rate** | < 1% of requests | 1-5% | > 5% |

---

## Cache Architecture

### Multi-Tier Cache Hierarchy

```
┌─────────────────────────────────────────────────────────┐
│  Request Flow                                           │
└─────────────────────────────────────────────────────────┘

Browser Cache (304 Not Modified)
    ↓ miss
CDN / Edge Cache (CloudFlare, AWS CloudFront)
    ↓ miss
API Gateway Cache (nginx, Kong)
    ↓ miss
Application L1 Cache (Node.js in-memory)
    ↓ miss
Application L2 Cache (Redis cluster)
    ↓ miss
Database Query Cache (PostgreSQL, MySQL)
    ↓ miss
Database (source of truth)
```

### Layer Characteristics

| Layer | Latency | Size | TTL Strategy | Best For |
|-------|---------|------|--------------|----------|
| **Browser** | 0ms | ~50MB | Long (hours/days) | Static assets, API responses |
| **CDN** | 10-50ms | ~GBs | Medium (hours) | Images, static content |
| **API Gateway** | 1-5ms | ~GBs | Short (minutes) | API responses, rate limiting |
| **L1 (Memory)** | < 1ms | 100MB-1GB | Very short (seconds) | Hot data, user sessions |
| **L2 (Redis)** | 1-5ms | 10-100GB | Short-medium (minutes-hours) | User data, computed results |
| **DB Query Cache** | 10-50ms | ~GBs | Short (seconds) | Query results |

---

## Caching Patterns

### 1. Cache-Aside (Lazy Loading)

**When to use**: General-purpose, most common pattern

```typescript
class CacheAside {
  async get(key: string, fetcher: () => Promise<any>): Promise<any> {
    // Try cache first
    const cached = await redis.get(key);
    if (cached) {
      logger.debug({ key, source: 'cache' }, 'Cache hit');
      await this.recordHit(key);
      return JSON.parse(cached);
    }
    
    // Cache miss - fetch from source
    logger.debug({ key, source: 'database' }, 'Cache miss');
    await this.recordMiss(key);
    
    const data = await fetcher();
    
    // Populate cache
    await redis.setex(key, this.getTTL(key), JSON.stringify(data));
    
    return data;
  }
  
  private getTTL(key: string): number {
    // Default 1 hour, override for specific keys
    const ttlOverrides: Record<string, number> = {
      'user:*': 3600,      // 1 hour
      'post:*': 7200,      // 2 hours
      'config:*': 86400,   // 24 hours
    };
    
    for (const [pattern, ttl] of Object.entries(ttlOverrides)) {
      if (key.match(pattern.replace('*', '.*'))) {
        return ttl;
      }
    }
    
    return 3600; // default 1 hour
  }
}
```

**Pros**: Simple, cache only what's needed, easy to understand  
**Cons**: Cache stampede risk, first request always slow

---

### 2. Write-Through Cache

**When to use**: When consistency is critical, reads > writes

```typescript
class WriteThroughCache {
  async get(key: string): Promise<any> {
    const cached = await redis.get(key);
    if (cached) return JSON.parse(cached);
    
    // Not in cache - should not happen with write-through
    // but handle gracefully
    const data = await db.get(key);
    await redis.setex(key, 3600, JSON.stringify(data));
    return data;
  }
  
  async set(key: string, value: any): Promise<void> {
    // Write to database first (source of truth)
    await db.set(key, value);
    
    // Then update cache
    await redis.setex(key, 3600, JSON.stringify(value));
    
    logger.info({ key, operation: 'write-through' }, 'Data updated');
  }
  
  async delete(key: string): Promise<void> {
    // Delete from both
    await Promise.all([
      db.delete(key),
      redis.del(key)
    ]);
  }
}
```

**Pros**: Always consistent, no cache misses after first write  
**Cons**: Write latency increased, writes to cache even if rarely read

---

### 3. Write-Behind (Write-Back) Cache

**When to use**: High write throughput, eventual consistency acceptable

```typescript
class WriteBehindCache {
  private writeQueue: Bull.Queue;
  
  constructor() {
    this.writeQueue = new Bull('db-writes', {
      redis: redisConfig,
      defaultJobOptions: {
        attempts: 3,
        backoff: { type: 'exponential', delay: 1000 }
      }
    });
    
    this.writeQueue.process(this.processWrite.bind(this));
  }
  
  async set(key: string, value: any): Promise<void> {
    // Write to cache immediately (fast)
    await redis.setex(key, 3600, JSON.stringify(value));
    
    // Queue database write (async)
    await this.writeQueue.add('write', { key, value }, {
      priority: this.getPriority(key),
      delay: this.getDelay(key)
    });
    
    logger.debug({ key, queued: true }, 'Write queued');
  }
  
  private async processWrite(job: Bull.Job): Promise<void> {
    const { key, value } = job.data;
    
    try {
      await db.set(key, value);
      logger.info({ key, jobId: job.id }, 'Database write completed');
    } catch (error) {
      logger.error({ key, jobId: job.id, error }, 'Database write failed');
      throw error; // Bull will retry
    }
  }
  
  private getPriority(key: string): number {
    // Critical data gets higher priority
    if (key.startsWith('payment:')) return 1;
    if (key.startsWith('user:')) return 5;
    return 10;
  }
  
  private getDelay(key: string): number {
    // Batch less critical writes
    if (key.startsWith('analytics:')) return 5000; // 5 seconds
    return 0; // immediate
  }
}
```

**Pros**: Very fast writes, can batch updates, reduces DB load  
**Cons**: Risk of data loss if cache fails before DB write

---

### 4. Read-Through Cache

**When to use**: Transparent caching, hide cache logic from application

```typescript
class ReadThroughCache {
  async get(key: string): Promise<any> {
    // Try cache
    const cached = await redis.get(key);
    if (cached) return JSON.parse(cached);
    
    // Cache miss - fetch and populate automatically
    const data = await this.fetchFromSource(key);
    await redis.setex(key, this.getTTL(key), JSON.stringify(data));
    
    return data;
  }
  
  private async fetchFromSource(key: string): Promise<any> {
    // Determine source based on key pattern
    if (key.startsWith('user:')) {
      const userId = key.split(':')[1];
      return await db.users.findById(userId);
    }
    
    if (key.startsWith('post:')) {
      const postId = key.split(':')[1];
      return await db.posts.findById(postId);
    }
    
    throw new Error(`Unknown key pattern: ${key}`);
  }
  
  private getTTL(key: string): number {
    // Adaptive TTL based on key type
    if (key.startsWith('user:')) return 3600;
    if (key.startsWith('post:')) return 7200;
    return 1800;
  }
}
```

**Pros**: Application doesn't need to know about cache, consistent interface  
**Cons**: Cache layer needs to know how to fetch data

---

### 5. Refresh-Ahead Cache

**When to use**: Predictable access patterns, avoid cache expiration delays

```typescript
class RefreshAheadCache {
  async get(key: string, fetcher: () => Promise<any>): Promise<any> {
    const cached = await redis.get(key);
    const ttl = await redis.ttl(key);
    
    if (cached) {
      // Check if we should refresh proactively
      const shouldRefresh = this.shouldRefresh(ttl);
      
      if (shouldRefresh) {
        // Async refresh - don't wait
        this.refreshInBackground(key, fetcher);
      }
      
      return JSON.parse(cached);
    }
    
    // Cache miss - fetch synchronously
    const data = await fetcher();
    await redis.setex(key, 3600, JSON.stringify(data));
    return data;
  }
  
  private shouldRefresh(ttl: number): boolean {
    // Refresh when 20% of TTL remaining
    const threshold = 3600 * 0.2; // 20% of 1 hour
    return ttl > 0 && ttl < threshold;
  }
  
  private async refreshInBackground(
    key: string, 
    fetcher: () => Promise<any>
  ): Promise<void> {
    try {
      logger.debug({ key }, 'Background refresh started');
      const data = await fetcher();
      await redis.setex(key, 3600, JSON.stringify(data));
      logger.debug({ key }, 'Background refresh completed');
    } catch (error) {
      logger.error({ key, error }, 'Background refresh failed');
      // Don't throw - keep serving stale data
    }
  }
}
```

**Pros**: Never serve stale data, no cache expiration delays  
**Cons**: Extra background work, may refresh unused data

---

## Smart Invalidation

### 1. Time-Based Invalidation (TTL)

**Simplest but least intelligent**

```typescript
class TTLInvalidation {
  async set(key: string, value: any, ttl: number): Promise<void> {
    await redis.setex(key, ttl, JSON.stringify(value));
  }
  
  // Adaptive TTL based on access patterns
  async setWithAdaptiveTTL(key: string, value: any): Promise<void> {
    const stats = await this.getAccessStats(key);
    const ttl = this.calculateAdaptiveTTL(stats);
    await redis.setex(key, ttl, JSON.stringify(value));
  }
  
  private calculateAdaptiveTTL(stats: AccessStats): number {
    const { accessCount, lastAccessTime } = stats;
    
    // High access = longer TTL
    if (accessCount > 1000) return 86400;  // 24 hours
    if (accessCount > 100) return 14400;   // 4 hours
    if (accessCount > 10) return 3600;     // 1 hour
    
    // Recent access = longer TTL
    const timeSinceLastAccess = Date.now() - lastAccessTime;
    if (timeSinceLastAccess < 60000) return 3600;  // < 1 min ago
    if (timeSinceLastAccess < 300000) return 1800; // < 5 min ago
    
    return 900; // 15 minutes default
  }
}
```

**Pros**: Simple, predictable, no coordination needed  
**Cons**: May serve stale data, may evict fresh data

---

### 2. Event-Based Invalidation

**Most intelligent - invalidate when data actually changes**

```typescript
class EventBasedInvalidation {
  private eventEmitter: EventEmitter;
  private invalidationMap: Map<string, string[]>;
  
  constructor() {
    this.eventEmitter = new EventEmitter();
    this.setupInvalidationRules();
    this.registerEventHandlers();
  }
  
  private setupInvalidationRules(): void {
    // Define what caches to invalidate for each event
    this.invalidationMap = new Map([
      ['user.updated', ['user:{userId}', 'user:{userId}:profile']],
      ['user.deleted', ['user:{userId}', 'user:{userId}:*']],
      ['post.created', ['user:{userId}:posts', 'feed:*']],
      ['post.updated', ['post:{postId}', 'user:{authorId}:posts']],
      ['post.deleted', ['post:{postId}', 'user:{authorId}:posts', 'feed:*']],
      ['comment.created', ['post:{postId}:comments']],
    ]);
  }
  
  private registerEventHandlers(): void {
    this.eventEmitter.on('user.updated', async (data) => {
      await this.invalidate('user.updated', data);
    });
    
    this.eventEmitter.on('post.created', async (data) => {
      await this.invalidate('post.created', data);
    });
    
    // ... register all events
  }
  
  async invalidate(event: string, data: any): Promise<void> {
    const patterns = this.invalidationMap.get(event);
    if (!patterns) return;
    
    const keys = patterns.map(pattern => 
      this.interpolate(pattern, data)
    );
    
    // Invalidate all affected keys
    await Promise.all(
      keys.map(async (keyPattern) => {
        if (keyPattern.includes('*')) {
          // Handle wildcard patterns
          const matchingKeys = await this.findMatchingKeys(keyPattern);
          await redis.del(...matchingKeys);
        } else {
          await redis.del(keyPattern);
        }
      })
    );
    
    logger.info({ event, keys }, 'Cache invalidated');
  }
  
  private interpolate(pattern: string, data: any): string {
    return pattern.replace(/{(\w+)}/g, (_, key) => data[key] || '*');
  }
  
  private async findMatchingKeys(pattern: string): Promise<string[]> {
    // Use SCAN for production (doesn't block)
    const keys: string[] = [];
    let cursor = '0';
    
    do {
      const [newCursor, foundKeys] = await redis.scan(
        cursor,
        'MATCH',
        pattern,
        'COUNT',
        100
      );
      cursor = newCursor;
      keys.push(...foundKeys);
    } while (cursor !== '0');
    
    return keys;
  }
}

// Usage example
const cache = new EventBasedInvalidation();

// When user updates profile
await db.updateUser(userId, updates);
cache.eventEmitter.emit('user.updated', { userId });

// When post is created
const post = await db.createPost(data);
cache.eventEmitter.emit('post.created', { 
  postId: post.id, 
  userId: post.authorId 
});
```

**Pros**: Always consistent, no stale data, efficient (only invalidate what changed)  
**Cons**: More complex, requires event infrastructure

---

### 3. Version-Based Invalidation

**Elegant solution for related data**

```typescript
class VersionBasedCache {
  async getUser(userId: string): Promise<User> {
    // Get current version
    const version = await redis.get(`user:${userId}:version`);
    const versionedKey = `user:${userId}:v${version || 0}`;
    
    let cached = await redis.get(versionedKey);
    if (cached) return JSON.parse(cached);
    
    // Fetch from DB
    const user = await db.users.findById(userId);
    await redis.setex(versionedKey, 3600, JSON.stringify(user));
    
    return user;
  }
  
  async updateUser(userId: string, updates: Partial<User>): Promise<void> {
    // Update database
    await db.users.update(userId, updates);
    
    // Increment version - old caches automatically become invalid
    await redis.incr(`user:${userId}:version`);
    await redis.expire(`user:${userId}:version`, 86400); // 24 hours
    
    logger.info({ userId }, 'User version incremented');
  }
}
```

**Pros**: No need to delete old keys, clean invalidation  
**Cons**: Keys accumulate (need cleanup job), version tracking overhead

---

### 4. Dependency Graph Invalidation

**Advanced: Track relationships between cached data**

```typescript
interface CacheNode {
  key: string;
  dependencies: string[];  // Keys that depend on this
  dependsOn: string[];     // Keys this depends on
}

class DependencyGraphCache {
  private graph: Map<string, CacheNode> = new Map();
  
  async set(
    key: string, 
    value: any, 
    dependencies: string[] = []
  ): Promise<void> {
    // Store the value
    await redis.setex(key, 3600, JSON.stringify(value));
    
    // Update dependency graph
    const node: CacheNode = {
      key,
      dependencies: [],
      dependsOn: dependencies
    };
    
    // Register this key as dependent on its dependencies
    for (const dep of dependencies) {
      const depNode = this.graph.get(dep) || {
        key: dep,
        dependencies: [],
        dependsOn: []
      };
      depNode.dependencies.push(key);
      this.graph.set(dep, depNode);
    }
    
    this.graph.set(key, node);
  }
  
  async invalidate(key: string): Promise<void> {
    const visited = new Set<string>();
    await this.invalidateCascade(key, visited);
  }
  
  private async invalidateCascade(
    key: string, 
    visited: Set<string>
  ): Promise<void> {
    if (visited.has(key)) return;
    visited.add(key);
    
    // Invalidate this key
    await redis.del(key);
    logger.debug({ key }, 'Cache invalidated');
    
    // Get all keys that depend on this key
    const node = this.graph.get(key);
    if (!node) return;
    
    // Recursively invalidate dependents
    await Promise.all(
      node.dependencies.map(depKey => 
        this.invalidateCascade(depKey, visited)
      )
    );
  }
}

// Usage example
const cache = new DependencyGraphCache();

// User posts depend on user data
await cache.set(
  'user:123:posts',
  posts,
  ['user:123']  // depends on user
);

// When user is updated, posts cache is also invalidated
await cache.invalidate('user:123');
// This automatically invalidates 'user:123:posts' too
```

**Pros**: Ensures consistency across related data, automatic cascade  
**Cons**: Complex, memory overhead for graph, potential over-invalidation

---

## Advanced Techniques

### 1. Thundering Herd Prevention

```typescript
class ThunderingHerdPrevention {
  private locks: Map<string, Promise<any>> = new Map();
  
  async get(key: string, fetcher: () => Promise<any>): Promise<any> {
    // Try cache first
    const cached = await redis.get(key);
    if (cached) return JSON.parse(cached);
    
    // Check if someone else is already fetching
    if (this.locks.has(key)) {
      logger.debug({ key }, 'Waiting for concurrent fetch');
      return await this.locks.get(key);
    }
    
    // Acquire distributed lock using Redis
    const lockKey = `lock:${key}`;
    const lockAcquired = await redis.set(
      lockKey,
      '1',
      'NX',  // Only set if not exists
      'EX',  // Set expiry
      10     // 10 seconds
    );
    
    if (!lockAcquired) {
      // Someone else got the lock, wait and retry
      await this.sleep(50);
      return this.get(key, fetcher);
    }
    
    try {
      // We have the lock - fetch data
      const promise = fetcher();
      this.locks.set(key, promise);
      
      const data = await promise;
      await redis.setex(key, 3600, JSON.stringify(data));
      
      return data;
    } finally {
      // Release lock
      await redis.del(lockKey);
      this.locks.delete(key);
    }
  }
  
  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

---

### 2. Probabilistic Early Expiration (XFetch)

```typescript
class XFetchCache {
  async get(
    key: string, 
    fetcher: () => Promise<any>,
    baseTTL: number = 3600
  ): Promise<any> {
    const result = await redis.get(key);
    
    if (result) {
      const ttlRemaining = await redis.ttl(key);
      
      // Calculate refresh probability
      // Higher probability as expiration approaches
      const delta = -baseTTL * Math.log(Math.random());
      const shouldRefresh = ttlRemaining < delta;
      
      if (shouldRefresh) {
        // Refresh asynchronously - return cached data immediately
        this.refreshAsync(key, fetcher, baseTTL);
      }
      
      return JSON.parse(result);
    }
    
    // Cache miss - fetch synchronously
    const data = await fetcher();
    await redis.setex(key, baseTTL, JSON.stringify(data));
    return data;
  }
  
  private async refreshAsync(
    key: string,
    fetcher: () => Promise<any>,
    ttl: number
  ): Promise<void> {
    try {
      const data = await fetcher();
      await redis.setex(key, ttl, JSON.stringify(data));
      logger.debug({ key }, 'Cache refreshed early');
    } catch (error) {
      logger.error({ key, error }, 'Early refresh failed');
    }
  }
}
```

---

### 3. Cache Warming

```typescript
class CacheWarmer {
  async warmCache(): Promise<void> {
    logger.info('Starting cache warming');
    
    // Warm popular users
    await this.warmPopularUsers();
    
    // Warm trending posts
    await this.warmTrendingPosts();
    
    // Warm common aggregations
    await this.warmAggregations();
    
    logger.info('Cache warming completed');
  }
  
  private async warmPopularUsers(): Promise<void> {
    // Get top 1000 users by activity
    const popularUsers = await db.query(`
      SELECT id FROM users 
      ORDER BY last_active_at DESC 
      LIMIT 1000
    `);
    
    await Promise.all(
      popularUsers.map(async (user) => {
        const userData = await db.users.findById(user.id);
        await redis.setex(
          `user:${user.id}`,
          7200,  // 2 hours
          JSON.stringify(userData)
        );
      })
    );
    
    logger.info({ count: popularUsers.length }, 'Popular users warmed');
  }
  
  private async warmTrendingPosts(): Promise<void> {
    const trending = await db.query(`
      SELECT id FROM posts 
      WHERE created_at > NOW() - INTERVAL '24 hours'
      ORDER BY view_count DESC 
      LIMIT 500
    `);
    
    await Promise.all(
      trending.map(async (post) => {
        const postData = await db.posts.findById(post.id);
        await redis.setex(
          `post:${post.id}`,
          3600,
          JSON.stringify(postData)
        );
      })
    );
    
    logger.info({ count: trending.length }, 'Trending posts warmed');
  }
  
  private async warmAggregations(): Promise<void> {
    // Pre-compute expensive aggregations
    const stats = await db.query(`
      SELECT 
        COUNT(*) as total_users,
        COUNT(*) FILTER (WHERE created_at > NOW() - INTERVAL '24 hours') as new_users_24h
      FROM users
    `);
    
    await redis.setex('stats:users', 300, JSON.stringify(stats));
  }
  
  // Schedule cache warming
  scheduleWarming(): void {
    // Warm cache every hour
    setInterval(() => this.warmCache(), 3600000);
    
    // Also warm on startup
    this.warmCache();
  }
}
```

---

### 4. Bloom Filters (Negative Caching)

```typescript
import { BloomFilter } from 'bloom-filters';

class NegativeCacheWithBloom {
  private bloom: BloomFilter;
  
  constructor() {
    // 1 million items, 1% false positive rate
    this.bloom = BloomFilter.create(1000000, 0.01);
  }
  
  async exists(key: string): Promise<boolean> {
    // Quick check: definitely doesn't exist
    if (!this.bloom.has(key)) {
      logger.debug({ key }, 'Bloom filter: definitely not exists');
      return false;
    }
    
    // Might exist - check cache
    const cached = await redis.get(`exists:${key}`);
    if (cached === 'yes') return true;
    if (cached === 'no') return false;
    
    // Check database
    const exists = await db.exists(key);
    
    if (exists) {
      // Add to bloom filter
      this.bloom.add(key);
      // Positive cache
      await redis.setex(`exists:${key}`, 3600, 'yes');
    } else {
      // Negative cache (shorter TTL)
      await redis.setex(`exists:${key}`, 300, 'no');
    }
    
    return exists;
  }
}
```

---

### 5. Request Coalescing

```typescript
class RequestCoalescer {
  private pending: Map<string, Promise<any>> = new Map();
  
  async fetch(key: string, fetcher: () => Promise<any>): Promise<any> {
    // Check if request is already in flight
    if (this.pending.has(key)) {
      logger.debug({ key }, 'Coalescing request');
      return await this.pending.get(key)!;
    }
    
    // Start new request
    const promise = fetcher().finally(() => {
      this.pending.delete(key);
    });
    
    this.pending.set(key, promise);
    return await promise;
  }
}

// Combine with cache
class CacheWithCoalescing {
  private coalescer = new RequestCoalescer();
  
  async get(key: string, fetcher: () => Promise<any>): Promise<any> {
    // Try cache
    const cached = await redis.get(key);
    if (cached) return JSON.parse(cached);
    
    // Cache miss - use coalescer to prevent duplicate fetches
    return await this.coalescer.fetch(key, async () => {
      const data = await fetcher();
      await redis.setex(key, 3600, JSON.stringify(data));
      return data;
    });
  }
}
```

---

## Implementation Examples

### Complete Production-Ready Cache Manager

```typescript
import Redis from 'ioredis';
import { EventEmitter } from 'events';
import { logger } from './logger';
import { metrics } from './metrics';

interface CacheOptions {
  ttl?: number;
  refreshAhead?: boolean;
  coalesce?: boolean;
  dependencies?: string[];
}

export class IntelligentCacheManager {
  private redis: Redis;
  private eventEmitter: EventEmitter;
  private coalescer: RequestCoalescer;
  private stats: Map<string, CacheStats>;
  
  constructor(redisConfig: any) {
    this.redis = new Redis(redisConfig);
    this.eventEmitter = new EventEmitter();
    this.coalescer = new RequestCoalescer();
    this.stats = new Map();
    
    this.setupMetrics();
    this.setupEventHandlers();
  }
  
  async get<T>(
    key: string,
    fetcher: () => Promise<T>,
    options: CacheOptions = {}
  ): Promise<T> {
    const startTime = Date.now();
    
    try {
      // Try cache
      const cached = await this.redis.get(key);
      
      if (cached) {
        this.recordHit(key);
        
        // Check if we should refresh ahead
        if (options.refreshAhead) {
          await this.maybeRefreshAhead(key, fetcher, options);
        }
        
        metrics.cacheLatency.observe(Date.now() - startTime);
        return JSON.parse(cached);
      }
      
      // Cache miss
      this.recordMiss(key);
      
      // Use request coalescing if enabled
      const data = options.coalesce
        ? await this.coalescer.fetch(key, fetcher)
        : await fetcher();
      
      // Store in cache
      const ttl = options.ttl || this.getAdaptiveTTL(key);
      await this.redis.setex(key, ttl, JSON.stringify(data));
      
      metrics.cacheLatency.observe(Date.now() - startTime);
      return data;
      
    } catch (error) {
      logger.error({ key, error }, 'Cache get failed');
      metrics.cacheErrors.inc({ operation: 'get' });
      
      // Fallback to direct fetch
      return await fetcher();
    }
  }
  
  async set(
    key: string,
    value: any,
    options: CacheOptions = {}
  ): Promise<void> {
    try {
      const ttl = options.ttl || 3600;
      await this.redis.setex(key, ttl, JSON.stringify(value));
      
      // Handle dependencies
      if (options.dependencies) {
        await this.registerDependencies(key, options.dependencies);
      }
      
      metrics.cacheWrites.inc();
    } catch (error) {
      logger.error({ key, error }, 'Cache set failed');
      metrics.cacheErrors.inc({ operation: 'set' });
      throw error;
    }
  }
  
  async invalidate(key: string): Promise<void> {
    try {
      await this.redis.del(key);
      
      // Cascade to dependents
      const dependents = await this.getDependents(key);
      await Promise.all(
        dependents.map(dep => this.invalidate(dep))
      );
      
      this.eventEmitter.emit('cache.invalidated', { key });
      metrics.cacheInvalidations.inc();
    } catch (error) {
      logger.error({ key, error }, 'Cache invalidation failed');
      metrics.cacheErrors.inc({ operation: 'invalidate' });
    }
  }
  
  private getAdaptiveTTL(key: string): number {
    const stats = this.stats.get(key);
    if (!stats) return 3600;
    
    const hitRate = stats.hits / (stats.hits + stats.misses);
    
    // High hit rate = longer TTL
    if (hitRate > 0.9) return 86400;  // 24 hours
    if (hitRate > 0.7) return 14400;  // 4 hours
    if (hitRate > 0.5) return 3600;   // 1 hour
    
    return 1800; // 30 minutes
  }
  
  private async maybeRefreshAhead(
    key: string,
    fetcher: () => Promise<any>,
    options: CacheOptions
  ): Promise<void> {
    const ttl = await this.redis.ttl(key);
    const baseTTL = options.ttl || 3600;
    
    // Refresh when 20% of TTL remaining
    const threshold = baseTTL * 0.2;
    
    if (ttl > 0 && ttl < threshold) {
      // Async refresh
      this.refreshAsync(key, fetcher, baseTTL);
    }
  }
  
  private async refreshAsync(
    key: string,
    fetcher: () => Promise<any>,
    ttl: number
  ): Promise<void> {
    try {
      const data = await fetcher();
      await this.redis.setex(key, ttl, JSON.stringify(data));
      logger.debug({ key }, 'Cache refreshed ahead');
    } catch (error) {
      logger.error({ key, error }, 'Refresh ahead failed');
    }
  }
  
  private recordHit(key: string): void {
    const stats = this.stats.get(key) || { hits: 0, misses: 0 };
    stats.hits++;
    this.stats.set(key, stats);
    metrics.cacheHits.inc();
  }
  
  private recordMiss(key: string): void {
    const stats = this.stats.get(key) || { hits: 0, misses: 0 };
    stats.misses++;
    this.stats.set(key, stats);
    metrics.cacheMisses.inc();
  }
  
  private setupMetrics(): void {
    // Export cache hit rate every minute
    setInterval(() => {
      const totalHits = Array.from(this.stats.values())
        .reduce((sum, s) => sum + s.hits, 0);
      const totalMisses = Array.from(this.stats.values())
        .reduce((sum, s) => sum + s.misses, 0);
      const hitRate = totalHits / (totalHits + totalMisses);
      
      metrics.cacheHitRate.set(hitRate);
    }, 60000);
  }
  
  private setupEventHandlers(): void {
    // Listen to domain events for invalidation
    this.eventEmitter.on('user.updated', async (data) => {
      await this.invalidate(`user:${data.userId}`);
    });
    
    this.eventEmitter.on('post.updated', async (data) => {
      await this.invalidate(`post:${data.postId}`);
    });
  }
  
  getStats(): Map<string, CacheStats> {
    return this.stats;
  }
}

interface CacheStats {
  hits: number;
  misses: number;
}
```

---

## Monitoring & Metrics

### Key Metrics to Track

```typescript
// Prometheus metrics
import { Counter, Histogram, Gauge } from 'prom-client';

export const cacheMetrics = {
  hits: new Counter({
    name: 'cache_hits_total',
    help: 'Total number of cache hits',
    labelNames: ['cache_layer']
  }),
  
  misses: new Counter({
    name: 'cache_misses_total',
    help: 'Total number of cache misses',
    labelNames: ['cache_layer']
  }),
  
  hitRate: new Gauge({
    name: 'cache_hit_rate',
    help: 'Cache hit rate (hits / total requests)',
    labelNames: ['cache_layer']
  }),
  
  latency: new Histogram({
    name: 'cache_operation_duration_seconds',
    help: 'Cache operation latency',
    labelNames: ['operation', 'cache_layer'],
    buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1]
  }),
  
  memoryUsage: new Gauge({
    name: 'cache_memory_bytes',
    help: 'Cache memory usage in bytes',
    labelNames: ['cache_layer']
  }),
  
  keyCount: new Gauge({
    name: 'cache_keys_total',
    help: 'Total number of keys in cache',
    labelNames: ['cache_layer']
  }),
  
  evictions: new Counter({
    name: 'cache_evictions_total',
    help: 'Total number of cache evictions',
    labelNames: ['cache_layer', 'reason']
  }),
  
  invalidations: new Counter({
    name: 'cache_invalidations_total',
    help: 'Total number of cache invalidations',
    labelNames: ['cache_layer', 'reason']
  }),
  
  errors: new Counter({
    name: 'cache_errors_total',
    help: 'Total number of cache errors',
    labelNames: ['cache_layer', 'operation']
  })
};
```

### Grafana Dashboard Queries

```promql
# Cache Hit Rate
sum(rate(cache_hits_total[5m])) / 
(sum(rate(cache_hits_total[5m])) + sum(rate(cache_misses_total[5m])))

# Cache Latency P99
histogram_quantile(0.99, 
  sum(rate(cache_operation_duration_seconds_bucket[5m])) by (le, operation)
)

# Memory Usage
cache_memory_bytes

# Eviction Rate
rate(cache_evictions_total[5m])

# Error Rate
rate(cache_errors_total[5m])
```

---

## Decision Framework

### When to Cache?

```typescript
class CacheDecisionFramework {
  shouldCache(data: any, context: CacheContext): boolean {
    // 1. Is data expensive to compute/fetch?
    if (context.computeTimeMs > 100) return true;
    
    // 2. Is data accessed frequently?
    if (context.accessFrequency > 10) return true;
    
    // 3. Is data relatively static?
    if (context.updateFrequency < 1 / 3600) return true; // < 1/hour
    
    // 4. Is consistency requirement relaxed?
    if (context.allowStale) return true;
    
    // Don't cache if:
    // - User-specific sensitive data
    // - Real-time data requirements
    // - Low hit rate expected
    // - Data changes frequently
    
    return false;
  }
  
  selectTTL(context: CacheContext): number {
    // Static data: long TTL
    if (context.updateFrequency < 1 / 86400) {
      return 86400; // 24 hours
    }
    
    // Slow-changing data: medium TTL
    if (context.updateFrequency < 1 / 3600) {
      return 3600; // 1 hour
    }
    
    // Frequently updated: short TTL
    if (context.updateFrequency > 1 / 300) {
      return 300; // 5 minutes
    }
    
    return 1800; // 30 minutes default
  }
  
  selectPattern(context: CacheContext): CachePattern {
    // Write-heavy: write-behind
    if (context.writeRatio > 0.7) {
      return CachePattern.WRITE_BEHIND;
    }
    
    // Read-heavy + consistency: write-through
    if (context.readRatio > 0.8 && context.consistencyRequired) {
      return CachePattern.WRITE_THROUGH;
    }
    
    // General purpose: cache-aside
    return CachePattern.CACHE_ASIDE;
  }
}

interface CacheContext {
  computeTimeMs: number;
  accessFrequency: number;  // accesses per second
  updateFrequency: number;  // updates per second
  allowStale: boolean;
  consistencyRequired: boolean;
  writeRatio: number;       // writes / total operations
  readRatio: number;        // reads / total operations
}

enum CachePattern {
  CACHE_ASIDE,
  WRITE_THROUGH,
  WRITE_BEHIND,
  READ_THROUGH,
  REFRESH_AHEAD
}
```

---

## Anti-Patterns

### ❌ DON'T DO THIS

#### 1. Cache Without Invalidation Strategy

```typescript
// BAD: No way to invalidate when data changes
async function getUser(userId: string): Promise<User> {
  const cached = await redis.get(`user:${userId}`);
  if (cached) return JSON.parse(cached);
  
  const user = await db.users.findById(userId);
  await redis.set(`user:${userId}`, JSON.stringify(user));
  return user;
}

// GOOD: Clear invalidation on update
async function updateUser(userId: string, updates: any): Promise<void> {
  await db.users.update(userId, updates);
  await redis.del(`user:${userId}`); // Invalidate cache
}
```

#### 2. Cache Everything

```typescript
// BAD: Caching data that's rarely accessed
async function getRandomData(id: string): Promise<any> {
  // This ID is accessed once every 10 hours
  // Cache hit rate will be < 1%
  return cacheManager.get(`data:${id}`, () => db.fetch(id));
}

// GOOD: Only cache frequently accessed data
async function getPopularData(id: string): Promise<any> {
  const popularity = await getAccessCount(id);
  
  if (popularity > THRESHOLD) {
    return cacheManager.get(`data:${id}`, () => db.fetch(id));
  }
  
  return db.fetch(id); // Skip cache for unpopular data
}
```

#### 3. Ignoring Thundering Herd

```typescript
// BAD: All requests hit DB on cache miss
async function getData(key: string): Promise<any> {
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);
  
  // 1000 concurrent requests all hit DB simultaneously
  const data = await expensiveDBQuery();
  await redis.setex(key, 3600, JSON.stringify(data));
  return data;
}

// GOOD: Use locks or request coalescing
// See ThunderingHerdPrevention class above
```

#### 4. No Cache Metrics

```typescript
// BAD: No visibility into cache performance
async function get(key: string): Promise<any> {
  return await redis.get(key);
}

// GOOD: Track all cache operations
async function get(key: string): Promise<any> {
  const result = await redis.get(key);
  metrics.cacheHits.inc({ hit: !!result });
  return result;
}
```

#### 5. Caching Errors

```typescript
// BAD: Caching error responses
async function fetchData(key: string): Promise<any> {
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);
  
  try {
    const data = await api.fetch(key);
    await redis.setex(key, 3600, JSON.stringify(data));
    return data;
  } catch (error) {
    // BAD: Caching the error
    await redis.setex(key, 3600, JSON.stringify({ error: true }));
    throw error;
  }
}

// GOOD: Only cache successful responses
async function fetchData(key: string): Promise<any> {
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);
  
  const data = await api.fetch(key); // Let errors propagate
  await redis.setex(key, 3600, JSON.stringify(data));
  return data;
}
```

---

## Testing & Benchmarks

### Load Testing Cache Performance

```typescript
import { check } from 'k6';
import http from 'k6/http';

export const options = {
  stages: [
    { duration: '30s', target: 100 },  // Ramp up
    { duration: '1m', target: 100 },   // Sustained load
    { duration: '30s', target: 500 },  // Spike
    { duration: '1m', target: 500 },   // High load
    { duration: '30s', target: 0 },    // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<100'],  // 95% of requests < 100ms
    cache_hit_rate: ['value>0.9'],     // > 90% hit rate
  },
};

export default function () {
  // Test cache hits
  const res1 = http.get('http://api/users/123');
  check(res1, {
    'cache hit': (r) => r.headers['X-Cache-Status'] === 'HIT',
    'response time < 10ms': (r) => r.timings.duration < 10,
  });
  
  // Test cache misses
  const randomId = Math.floor(Math.random() * 1000000);
  const res2 = http.get(`http://api/users/${randomId}`);
  check(res2, {
    'cache miss handled': (r) => r.status === 200 || r.status === 404,
  });
}
```

### Unit Tests

```typescript
import { IntelligentCacheManager } from './cache';
import { expect } from 'chai';

describe('IntelligentCacheManager', () => {
  let cache: IntelligentCacheManager;
  
  beforeEach(() => {
    cache = new IntelligentCacheManager(redisConfig);
  });
  
  it('should cache value on first access', async () => {
    let fetchCount = 0;
    const fetcher = async () => {
      fetchCount++;
      return { data: 'test' };
    };
    
    await cache.get('test-key', fetcher);
    expect(fetchCount).to.equal(1);
    
    // Second access should use cache
    await cache.get('test-key', fetcher);
    expect(fetchCount).to.equal(1); // Still 1, not called again
  });
  
  it('should invalidate cache on update', async () => {
    await cache.set('test-key', { data: 'test' });
    
    let value = await cache.get('test-key', async () => ({ data: 'fresh' }));
    expect(value.data).to.equal('test');
    
    await cache.invalidate('test-key');
    
    value = await cache.get('test-key', async () => ({ data: 'fresh' }));
    expect(value.data).to.equal('fresh');
  });
  
  it('should handle thundering herd', async () => {
    let fetchCount = 0;
    const slowFetcher = async () => {
      await new Promise(resolve => setTimeout(resolve, 100));
      fetchCount++;
      return { data: 'test' };
    };
    
    // 100 concurrent requests
    await Promise.all(
      Array(100).fill(null).map(() => 
        cache.get('test-key', slowFetcher, { coalesce: true })
      )
    );
    
    // Should only fetch once
    expect(fetchCount).to.equal(1);
  });
  
  it('should adapt TTL based on hit rate', async () => {
    // Simulate high hit rate
    for (let i = 0; i < 100; i++) {
      await cache.get('hot-key', async () => ({ data: 'hot' }));
    }
    
    const stats = cache.getStats().get('hot-key');
    expect(stats!.hits).to.be.greaterThan(90);
    
    // TTL should be increased for hot data
    // (verify through Redis TTL inspection)
  });
});
```

---

## Runbook: Cache Issues

### High Cache Miss Rate

**Symptoms**: Cache hit rate < 70%

**Investigation**:
```bash
# Check hit rate
redis-cli INFO stats | grep keyspace_hits

# Check eviction rate
redis-cli INFO stats | grep evicted_keys

# Check memory usage
redis-cli INFO memory | grep used_memory_human
```

**Resolution**:
1. Increase Redis memory if evictions are high
2. Review TTL strategy (might be too short)
3. Implement cache warming for popular data
4. Check if access patterns changed

### Cache Memory Full

**Symptoms**: Evictions increasing, hit rate dropping

**Investigation**:
```bash
# Check memory
redis-cli INFO memory

# Check eviction policy
redis-cli CONFIG GET maxmemory-policy
```

**Resolution**:
1. Increase Redis memory: `redis-cli CONFIG SET maxmemory 4gb`
2. Review eviction policy: `redis-cli CONFIG SET maxmemory-policy allkeys-lru`
3. Identify large keys: `redis-cli --bigkeys`
4. Implement TTL cleanup for unused data

### Cache Stampede

**Symptoms**: Sudden DB load spike, slow responses

**Investigation**:
```bash
# Check concurrent requests
redis-cli --stat

# Check DB load
pg_stat_activity
```

**Resolution**:
1. Implement request coalescing
2. Add distributed locks
3. Use probabilistic early expiration
4. Warm cache before expiration

---

## Summary

### Quick Reference

| Pattern | Use When | Pros | Cons |
|---------|----------|------|------|
| **Cache-Aside** | General purpose | Simple, flexible | Cache stampede risk |
| **Write-Through** | Consistency critical | Always consistent | Slower writes |
| **Write-Behind** | High write load | Fast writes | Data loss risk |
| **Read-Through** | Transparent caching | Clean interface | Cache needs fetch logic |
| **Refresh-Ahead** | Predictable access | No expiration delays | Extra background work |

### Implementation Checklist

- [ ] Choose appropriate cache pattern
- [ ] Implement invalidation strategy
- [ ] Add request coalescing for hot keys
- [ ] Set up cache metrics (hit rate, latency)
- [ ] Configure alerting (hit rate < 70%)
- [ ] Implement cache warming for popular data
- [ ] Add distributed locks for expensive operations
- [ ] Test cache performance under load
- [ ] Document cache keys and TTLs
- [ ] Set up cache monitoring dashboard

---

## Next Steps

1. **Read**: [Database Layer](database-layer.md) for query optimization
2. **Read**: [Performance & Scalability](performance-scalability.md) for load balancing
3. **Implement**: Cache metrics endpoint
4. **Set up**: Grafana dashboard for cache monitoring
5. **Test**: Load test with k6 to verify performance

---

**Questions?** Check [Integration Guide](integration-guide.md) or ask in #infrastructure
