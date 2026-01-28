# Spam Detection Strategy
## Template Hashing, Duplicate Detection & Performance Optimization

---
**Status**: 💡 **GUIDANCE** - Spam detection patterns and best practices  
**Priority**: 🟡 **P1** - Important for preventing abuse  
**Last Updated**: 2026-01-28  
**Owned By**: Security Team, Infrastructure Team  
---

## Table of Contents

1. [Overview](#overview)
2. [The Problem with Database-Based Detection](#the-problem-with-database-based-detection)
3. [Template Hashing Solution](#template-hashing-solution)
4. [Redis-Based Duplicate Detection](#redis-based-duplicate-detection)
5. [Content Fingerprinting](#content-fingerprinting)
6. [Multi-Layer Spam Detection](#multi-layer-spam-detection)
7. [Performance Optimization](#performance-optimization)
8. [Implementation Examples](#implementation-examples)

---

## Overview

### The Challenge

**Problem**: Detecting spam messages when:
- Values change frequently (user IDs, timestamps, dynamic content)
- Same message template with different values
- Database queries become sluggish at scale
- Need sub-second detection

**Solution**: **Template hashing** + **Redis** for fast duplicate detection

### Why Template Hashing?

```
Message 1: "Hello user123, your order #456 is ready!"
Message 2: "Hello user789, your order #123 is ready!"

Template: "Hello user{ID}, your order #{ORDER} is ready!"
Template Hash: sha256("Hello user{ID}, your order #{ORDER} is ready!")
```

**Benefits**:
- ✅ Identifies same message pattern despite different values
- ✅ Fast hash comparison (O(1))
- ✅ Works with Redis for sub-millisecond lookups
- ✅ Handles dynamic content gracefully

---

## The Problem with Database-Based Detection

### Why MariaDB/PostgreSQL Becomes Sluggish

**Issue 1: Full-Text Comparison**
```sql
-- Slow: Full table scan for each message
SELECT COUNT(*) FROM messages 
WHERE content LIKE '%pattern%' 
AND created_at > NOW() - INTERVAL 1 HOUR;

-- Even with index, LIKE queries are slow
-- O(n) complexity where n = number of messages
```

**Issue 2: Exact Match Doesn't Work**
```sql
-- This won't catch spam with different values:
SELECT * FROM messages 
WHERE content = 'Hello user123, your order #456 is ready!';
-- Misses: "Hello user789, your order #123 is ready!"
```

**Issue 3: Pattern Matching is Expensive**
```sql
-- Very slow, especially with regex
SELECT * FROM messages 
WHERE content REGEXP 'Hello user[0-9]+, your order #[0-9]+ is ready!';
-- Database has to scan every row and apply regex
```

**Performance Impact**:
- **Small scale** (< 10k messages): Acceptable (100-500ms)
- **Medium scale** (100k messages): Slow (1-5 seconds)
- **Large scale** (> 1M messages): Unusable (> 10 seconds)

---

## Template Hashing Solution

### What is Template Hashing?

Template hashing normalizes messages by replacing variable values with placeholders, then hashes the template.

**Example**:
```typescript
// Original messages
const msg1 = "Hello user123, your order #456 is ready!";
const msg2 = "Hello user789, your order #123 is ready!";

// Extract template
const template1 = extractTemplate(msg1);
// Result: "Hello user{ID}, your order #{ORDER} is ready!"

const template2 = extractTemplate(msg2);
// Result: "Hello user{ID}, your order #{ORDER} is ready!"

// Hash templates
const hash1 = sha256(template1);
const hash2 = sha256(template2);

// hash1 === hash2 ✅ (same template detected!)
```

### Template Extraction Strategies

#### Strategy 1: Pattern-Based Extraction

```typescript
import { createHash } from 'crypto';

class TemplateHasher {
  // Extract template by replacing common patterns
  extractTemplate(message: string): string {
    let template = message;
    
    // Replace UUIDs
    template = template.replace(
      /[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}/gi,
      '{UUID}'
    );
    
    // Replace numbers (IDs, order numbers, etc.)
    template = template.replace(/\b\d{4,}\b/g, '{NUMBER}');
    
    // Replace emails
    template = template.replace(
      /\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b/g,
      '{EMAIL}'
    );
    
    // Replace URLs
    template = template.replace(
      /https?:\/\/[^\s]+/g,
      '{URL}'
    );
    
    // Replace timestamps
    template = template.replace(
      /\d{4}-\d{2}-\d{2}[T ]\d{2}:\d{2}:\d{2}/g,
      '{TIMESTAMP}'
    );
    
    // Replace IP addresses
    template = template.replace(
      /\b\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}\b/g,
      '{IP}'
    );
    
    return template;
  }
  
  hashTemplate(template: string): string {
    return createHash('sha256')
      .update(template.toLowerCase().trim())
      .digest('hex');
  }
  
  getTemplateHash(message: string): string {
    const template = this.extractTemplate(message);
    return this.hashTemplate(template);
  }
}
```

#### Strategy 2: Machine Learning-Based Extraction

```typescript
// More sophisticated: Use ML to identify variable parts
class MLTemplateExtractor {
  // Train on sample messages to identify patterns
  // Replace variable parts with placeholders
  extractTemplate(message: string): string {
    // Use NLP or pattern recognition
    // Identify which parts are likely variable vs static
    // This is more accurate but requires training data
  }
}
```

#### Strategy 3: Rule-Based with Custom Patterns

```typescript
class CustomTemplateExtractor {
  private patterns: Array<{ regex: RegExp; placeholder: string }>;
  
  constructor() {
    this.patterns = [
      { regex: /\buser\d+\b/gi, placeholder: '{USER_ID}' },
      { regex: /order\s*#?\d+/gi, placeholder: '{ORDER_ID}' },
      { regex: /\$\d+\.\d{2}/g, placeholder: '{AMOUNT}' },
      { regex: /\d{1,2}\/\d{1,2}\/\d{4}/g, placeholder: '{DATE}' },
      // Add domain-specific patterns
    ];
  }
  
  extractTemplate(message: string): string {
    let template = message;
    
    for (const { regex, placeholder } of this.patterns) {
      template = template.replace(regex, placeholder);
    }
    
    return template;
  }
}
```

---

## Redis-Based Duplicate Detection

### Why Redis Instead of Database?

| Feature | MariaDB/PostgreSQL | Redis |
|---------|-------------------|-------|
| **Lookup Speed** | 10-100ms | < 1ms |
| **Write Speed** | 10-50ms | < 1ms |
| **TTL Support** | Manual cleanup | Built-in |
| **Memory Usage** | Disk-based | Memory-based |
| **Scalability** | Vertical scaling | Horizontal scaling |

**Verdict**: Redis is **10-100x faster** for duplicate detection.

### Implementation: Template Hash Detection

```typescript
import Redis from 'ioredis';
import { createHash } from 'crypto';

class SpamDetector {
  private redis: Redis;
  private templateHasher: TemplateHasher;
  
  constructor(redis: Redis) {
    this.redis = redis;
    this.templateHasher = new TemplateHasher();
  }
  
  /**
   * Check if message is spam based on template hash
   * @param message - The message to check
   * @param userId - User ID (optional, for user-specific limits)
   * @param timeWindow - Time window in seconds (default: 3600 = 1 hour)
   * @param maxOccurrences - Max occurrences before considered spam (default: 5)
   */
  async isSpam(
    message: string,
    userId?: string,
    timeWindow: number = 3600,
    maxOccurrences: number = 5
  ): Promise<{ isSpam: boolean; occurrences: number; templateHash: string }> {
    // Extract template and hash
    const templateHash = this.templateHasher.getTemplateHash(message);
    
    // Create Redis key
    const key = userId
      ? `spam:template:${templateHash}:user:${userId}`
      : `spam:template:${templateHash}`;
    
    // Check current count
    const count = await this.redis.incr(key);
    
    // Set TTL on first occurrence
    if (count === 1) {
      await this.redis.expire(key, timeWindow);
    }
    
    // Also track global template hash (across all users)
    const globalKey = `spam:template:${templateHash}:global`;
    const globalCount = await this.redis.incr(globalKey);
    if (globalCount === 1) {
      await this.redis.expire(globalKey, timeWindow);
    }
    
    const isSpam = count > maxOccurrences || globalCount > maxOccurrences * 10;
    
    return {
      isSpam,
      occurrences: count,
      templateHash
    };
  }
  
  /**
   * Check spam with multiple criteria
   */
  async checkSpam(
    message: string,
    userId?: string,
    ipAddress?: string
  ): Promise<SpamCheckResult> {
    const templateHash = this.templateHasher.getTemplateHash(message);
    
    // Check template hash
    const templateCheck = await this.isSpam(message, userId);
    
    // Check user-specific rate limit
    const userRateLimit = userId
      ? await this.checkUserRateLimit(userId)
      : { allowed: true, remaining: 100 };
    
    // Check IP-based rate limit
    const ipRateLimit = ipAddress
      ? await this.checkIPRateLimit(ipAddress)
      : { allowed: true, remaining: 100 };
    
    // Check content similarity (fuzzy matching)
    const similarityCheck = await this.checkContentSimilarity(message, userId);
    
    return {
      isSpam: templateCheck.isSpam || 
              !userRateLimit.allowed || 
              !ipRateLimit.allowed ||
              similarityCheck.isSpam,
      reasons: [
        templateCheck.isSpam && 'Template hash match',
        !userRateLimit.allowed && 'User rate limit exceeded',
        !ipRateLimit.allowed && 'IP rate limit exceeded',
        similarityCheck.isSpam && 'Content similarity detected'
      ].filter(Boolean) as string[],
      templateHash,
      templateOccurrences: templateCheck.occurrences,
      userRateLimit: userRateLimit.remaining,
      ipRateLimit: ipRateLimit.remaining
    };
  }
  
  private async checkUserRateLimit(userId: string): Promise<RateLimitResult> {
    const key = `spam:rate:user:${userId}`;
    const count = await this.redis.incr(key);
    
    if (count === 1) {
      await this.redis.expire(key, 3600); // 1 hour window
    }
    
    return {
      allowed: count <= 100, // 100 messages per hour
      remaining: Math.max(0, 100 - count)
    };
  }
  
  private async checkIPRateLimit(ipAddress: string): Promise<RateLimitResult> {
    const key = `spam:rate:ip:${ipAddress}`;
    const count = await this.redis.incr(key);
    
    if (count === 1) {
      await this.redis.expire(key, 3600);
    }
    
    return {
      allowed: count <= 200, // 200 messages per hour per IP
      remaining: Math.max(0, 200 - count)
    };
  }
  
  private async checkContentSimilarity(
    message: string,
    userId?: string
  ): Promise<{ isSpam: boolean; similarity: number }> {
    // Use MinHash or SimHash for fuzzy duplicate detection
    // See Content Fingerprinting section below
    return { isSpam: false, similarity: 0 };
  }
}
```

### Usage Example

```typescript
// Middleware for spam detection
export function spamDetectionMiddleware() {
  return async (req: Request, res: Response, next: NextFunction) => {
    const message = req.body.message || req.body.content;
    
    if (!message) {
      return next();
    }
    
    const spamDetector = new SpamDetector(redis);
    const result = await spamDetector.checkSpam(
      message,
      req.user?.id,
      req.ip
    );
    
    if (result.isSpam) {
      logger.warn('Spam detected', {
        userId: req.user?.id,
        ip: req.ip,
        templateHash: result.templateHash,
        reasons: result.reasons
      });
      
      return res.status(429).json({
        error: 'Spam detected',
        message: 'This message appears to be spam. Please try again later.',
        retryAfter: 3600 // seconds
      });
    }
    
    // Add spam check result to request for logging
    req.spamCheck = result;
    next();
  };
}

// Apply to routes
app.post('/api/messages', 
  authenticateJWT,
  spamDetectionMiddleware(),
  async (req, res) => {
    // Process message
    const message = await messageService.create(req.body.message);
    res.json(message);
  }
);
```

---

## Content Fingerprinting

### SimHash for Fuzzy Duplicate Detection

SimHash creates a fingerprint that allows detecting similar (not just identical) messages.

```typescript
import { createHash } from 'crypto';

class SimHashGenerator {
  /**
   * Generate SimHash fingerprint for message
   * Similar messages will have similar hashes
   */
  generateSimHash(message: string, hashBits: number = 64): bigint {
    const words = this.tokenize(message);
    const hashValues: number[] = new Array(hashBits).fill(0);
    
    // Hash each word
    for (const word of words) {
      const hash = this.hashWord(word, hashBits);
      
      // Add to vector
      for (let i = 0; i < hashBits; i++) {
        if (hash & (1 << i)) {
          hashValues[i]++;
        } else {
          hashValues[i]--;
        }
      }
    }
    
    // Convert to binary
    let simHash = 0n;
    for (let i = 0; i < hashBits; i++) {
      if (hashValues[i] > 0) {
        simHash |= (1n << BigInt(i));
      }
    }
    
    return simHash;
  }
  
  /**
   * Check if two messages are similar
   * Returns Hamming distance (lower = more similar)
   */
  similarity(hash1: bigint, hash2: bigint, hashBits: number = 64): number {
    const xor = hash1 ^ hash2;
    let distance = 0;
    
    for (let i = 0; i < hashBits; i++) {
      if (xor & (1n << BigInt(i))) {
        distance++;
      }
    }
    
    return distance;
  }
  
  private tokenize(message: string): string[] {
    // Simple tokenization (can be improved with NLP)
    return message
      .toLowerCase()
      .replace(/[^\w\s]/g, ' ')
      .split(/\s+/)
      .filter(word => word.length > 2);
  }
  
  private hashWord(word: string, bits: number): number {
    const hash = createHash('sha256').update(word).digest();
    // Convert to number with specified bits
    let value = 0;
    for (let i = 0; i < Math.min(bits / 8, hash.length); i++) {
      value = (value << 8) | hash[i];
    }
    return value;
  }
}

// Usage
const simHash = new SimHashGenerator();

const hash1 = simHash.generateSimHash("Hello user123, your order #456 is ready!");
const hash2 = simHash.generateSimHash("Hello user789, your order #123 is ready!");

const distance = simHash.similarity(hash1, hash2);
// Distance < 10 → Very similar (likely same template)
// Distance > 20 → Different messages
```

### MinHash for Set Similarity

```typescript
class MinHashGenerator {
  private numHashes: number = 128;
  
  /**
   * Generate MinHash signature for message
   * Good for detecting similar content with different ordering
   */
  generateMinHash(message: string): number[] {
    const shingles = this.getShingles(message, 3); // 3-grams
    const signature: number[] = new Array(this.numHashes).fill(Infinity);
    
    for (const shingle of shingles) {
      const hash = this.hashShingle(shingle);
      
      for (let i = 0; i < this.numHashes; i++) {
        const hashValue = this.hashWithSeed(hash, i);
        signature[i] = Math.min(signature[i], hashValue);
      }
    }
    
    return signature;
  }
  
  /**
   * Calculate Jaccard similarity between two MinHash signatures
   * Returns value between 0 (different) and 1 (identical)
   */
  jaccardSimilarity(sig1: number[], sig2: number[]): number {
    let matches = 0;
    for (let i = 0; i < this.numHashes; i++) {
      if (sig1[i] === sig2[i]) {
        matches++;
      }
    }
    return matches / this.numHashes;
  }
  
  private getShingles(text: string, n: number): string[] {
    const shingles = new Set<string>();
    const words = text.toLowerCase().split(/\s+/);
    
    for (let i = 0; i <= words.length - n; i++) {
      const shingle = words.slice(i, i + n).join(' ');
      shingles.add(shingle);
    }
    
    return Array.from(shingles);
  }
  
  private hashShingle(shingle: string): number {
    let hash = 0;
    for (let i = 0; i < shingle.length; i++) {
      hash = ((hash << 5) - hash) + shingle.charCodeAt(i);
      hash = hash & hash; // Convert to 32-bit integer
    }
    return Math.abs(hash);
  }
  
  private hashWithSeed(value: number, seed: number): number {
    return (value * 31 + seed) % 2147483647;
  }
}
```

---

## Multi-Layer Spam Detection

### Layered Approach

```
┌─────────────────────────────────────────┐
│  Layer 1: Rate Limiting                 │
│  (IP, User, Endpoint-based)             │
└─────────────────────────────────────────┘
              ↓ (if passed)
┌─────────────────────────────────────────┐
│  Layer 2: Template Hash Detection       │
│  (Redis-based, fast)                     │
└─────────────────────────────────────────┘
              ↓ (if passed)
┌─────────────────────────────────────────┐
│  Layer 3: Content Fingerprinting        │
│  (SimHash/MinHash for fuzzy matching)   │
└─────────────────────────────────────────┘
              ↓ (if passed)
┌─────────────────────────────────────────┐
│  Layer 4: ML-Based Classification        │
│  (Optional, for advanced detection)     │
└─────────────────────────────────────────┘
```

### Complete Implementation

```typescript
class MultiLayerSpamDetector {
  private redis: Redis;
  private templateHasher: TemplateHasher;
  private simHash: SimHashGenerator;
  private minHash: MinHashGenerator;
  
  constructor(redis: Redis) {
    this.redis = redis;
    this.templateHasher = new TemplateHasher();
    this.simHash = new SimHashGenerator();
    this.minHash = new MinHashGenerator();
  }
  
  async detectSpam(
    message: string,
    userId?: string,
    ipAddress?: string
  ): Promise<SpamDetectionResult> {
    const checks: SpamCheck[] = [];
    
    // Layer 1: Rate Limiting
    if (userId) {
      const userRate = await this.checkRateLimit(`user:${userId}`, 100, 3600);
      checks.push({ layer: 'rate_limit_user', passed: userRate.allowed });
      if (!userRate.allowed) {
        return this.createSpamResult(false, checks, 'User rate limit exceeded');
      }
    }
    
    if (ipAddress) {
      const ipRate = await this.checkRateLimit(`ip:${ipAddress}`, 200, 3600);
      checks.push({ layer: 'rate_limit_ip', passed: ipRate.allowed });
      if (!ipRate.allowed) {
        return this.createSpamResult(false, checks, 'IP rate limit exceeded');
      }
    }
    
    // Layer 2: Template Hash Detection
    const templateHash = this.templateHasher.getTemplateHash(message);
    const templateKey = `spam:template:${templateHash}`;
    const templateCount = await this.redis.incr(templateKey);
    
    if (templateCount === 1) {
      await this.redis.expire(templateKey, 3600);
    }
    
    checks.push({ 
      layer: 'template_hash', 
      passed: templateCount <= 5,
      metadata: { templateHash, count: templateCount }
    });
    
    if (templateCount > 5) {
      return this.createSpamResult(false, checks, 'Template hash spam detected');
    }
    
    // Layer 3: Content Fingerprinting (SimHash)
    const simHashValue = this.simHash.generateSimHash(message);
    const simHashKey = `spam:simhash:${simHashValue.toString(16)}`;
    const similarMessages = await this.redis.smembers(simHashKey);
    
    // Check similarity with recent messages
    if (similarMessages.length > 0) {
      for (const similarHash of similarMessages) {
        const distance = this.simHash.similarity(
          simHashValue,
          BigInt('0x' + similarHash)
        );
        
        if (distance < 5) { // Very similar
          checks.push({ 
            layer: 'simhash', 
            passed: false,
            metadata: { distance, similarHash }
          });
          return this.createSpamResult(false, checks, 'Similar content detected');
        }
      }
    }
    
    // Store SimHash for future comparisons
    await this.redis.sadd(simHashKey, simHashValue.toString(16));
    await this.redis.expire(simHashKey, 3600);
    
    checks.push({ layer: 'simhash', passed: true });
    
    // Layer 4: MinHash for set similarity
    const minHashSig = this.minHash.generateMinHash(message);
    const minHashKey = `spam:minhash:${this.hashSignature(minHashSig)}`;
    const similarSigs = await this.redis.smembers(minHashKey);
    
    for (const sigStr of similarSigs) {
      const similarSig = JSON.parse(sigStr);
      const similarity = this.minHash.jaccardSimilarity(minHashSig, similarSig);
      
      if (similarity > 0.8) { // 80% similar
        checks.push({ 
          layer: 'minhash', 
          passed: false,
          metadata: { similarity }
        });
        return this.createSpamResult(false, checks, 'High content similarity detected');
      }
    }
    
    // Store MinHash signature
    await this.redis.sadd(minHashKey, JSON.stringify(minHashSig));
    await this.redis.expire(minHashKey, 3600);
    
    checks.push({ layer: 'minhash', passed: true });
    
    // All checks passed
    return this.createSpamResult(true, checks);
  }
  
  private async checkRateLimit(
    key: string,
    maxRequests: number,
    windowSeconds: number
  ): Promise<{ allowed: boolean; remaining: number }> {
    const redisKey = `spam:rate:${key}`;
    const count = await this.redis.incr(redisKey);
    
    if (count === 1) {
      await this.redis.expire(redisKey, windowSeconds);
    }
    
    return {
      allowed: count <= maxRequests,
      remaining: Math.max(0, maxRequests - count)
    };
  }
  
  private hashSignature(signature: number[]): string {
    return createHash('sha256')
      .update(signature.join(','))
      .digest('hex')
      .substring(0, 16); // Use first 16 chars
  }
  
  private createSpamResult(
    isSpam: boolean,
    checks: SpamCheck[],
    reason?: string
  ): SpamDetectionResult {
    return {
      isSpam,
      reason,
      checks,
      timestamp: new Date()
    };
  }
}
```

---

## Performance Optimization

### Optimization Strategies

#### 1. **Redis Pipeline for Batch Operations**

```typescript
// ❌ Slow: Multiple round trips
const count1 = await redis.incr('key1');
const count2 = await redis.incr('key2');
const count3 = await redis.incr('key3');

// ✅ Fast: Single round trip
const pipeline = redis.pipeline();
pipeline.incr('key1');
pipeline.incr('key2');
pipeline.incr('key3');
const results = await pipeline.exec();
```

#### 2. **Lua Scripts for Atomic Operations**

```typescript
// Atomic check-and-increment with TTL
const checkSpamScript = `
  local key = KEYS[1]
  local max = tonumber(ARGV[1])
  local ttl = tonumber(ARGV[2])
  
  local count = redis.call('INCR', key)
  
  if count == 1 then
    redis.call('EXPIRE', key, ttl)
  end
  
  if count > max then
    return {1, count}  -- Spam detected
  else
    return {0, count}  -- Not spam
  end
`;

async function checkSpamAtomic(key: string, max: number, ttl: number) {
  const result = await redis.eval(
    checkSpamScript,
    1,
    key,
    max,
    ttl
  ) as [number, number];
  
  return {
    isSpam: result[0] === 1,
    count: result[1]
  };
}
```

#### 3. **Bloom Filters for Pre-Filtering**

```typescript
// Use Bloom filter to quickly check if template hash might exist
// Only query Redis if Bloom filter says "might exist"
import { BloomFilter } from 'bloom-filters';

class BloomFilterSpamDetector {
  private bloomFilter: BloomFilter;
  private redis: Redis;
  
  constructor() {
    // Bloom filter: 1M items, 0.1% false positive rate
    this.bloomFilter = BloomFilter.create(1000000, 0.001);
  }
  
  async checkSpam(templateHash: string): Promise<boolean> {
    // Quick check: Bloom filter
    if (!this.bloomFilter.has(templateHash)) {
      return false; // Definitely not spam
    }
    
    // Might be spam: Check Redis
    const count = await this.redis.get(`spam:template:${templateHash}`);
    return parseInt(count || '0') > 5;
  }
  
  async addTemplate(templateHash: string): Promise<void> {
    // Add to Bloom filter
    this.bloomFilter.add(templateHash);
    
    // Add to Redis
    await this.redis.incr(`spam:template:${templateHash}`);
  }
}
```

#### 4. **Caching Template Hashes**

```typescript
// Cache template extraction results
class CachedTemplateHasher {
  private cache: Map<string, string> = new Map();
  private maxCacheSize = 10000;
  
  getTemplateHash(message: string): string {
    // Check cache first
    const cached = this.cache.get(message);
    if (cached) {
      return cached;
    }
    
    // Compute template hash
    const template = this.extractTemplate(message);
    const hash = this.hashTemplate(template);
    
    // Cache result
    if (this.cache.size >= this.maxCacheSize) {
      // Remove oldest (simple FIFO)
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
    this.cache.set(message, hash);
    
    return hash;
  }
}
```

### Performance Comparison

| Approach | Lookup Time | Memory | Scalability |
|----------|-------------|--------|-------------|
| **MariaDB (exact match)** | 10-100ms | Low | Poor (sluggish) |
| **MariaDB (LIKE query)** | 100-1000ms | Low | Very Poor |
| **Redis (template hash)** | < 1ms | Medium | Excellent |
| **Redis + Bloom Filter** | < 0.5ms | Low | Excellent |

---

## Implementation Examples

### Complete Spam Detection Service

```typescript
import Redis from 'ioredis';
import { createHash } from 'crypto';

interface SpamCheckResult {
  isSpam: boolean;
  reason?: string;
  templateHash: string;
  occurrences: number;
  checks: {
    rateLimit: boolean;
    templateHash: boolean;
    contentSimilarity: boolean;
  };
}

class SpamDetectionService {
  private redis: Redis;
  private templateHasher: TemplateHasher;
  
  constructor(redis: Redis) {
    this.redis = redis;
    this.templateHasher = new TemplateHasher();
  }
  
  /**
   * Main spam detection method
   */
  async detectSpam(
    message: string,
    userId?: string,
    ipAddress?: string,
    options: {
      templateThreshold?: number;
      rateLimitUser?: number;
      rateLimitIP?: number;
      timeWindow?: number;
    } = {}
  ): Promise<SpamCheckResult> {
    const {
      templateThreshold = 5,
      rateLimitUser = 100,
      rateLimitIP = 200,
      timeWindow = 3600
    } = options;
    
    // Extract template hash
    const templateHash = this.templateHasher.getTemplateHash(message);
    
    // Use Lua script for atomic operations
    const script = `
      local templateKey = KEYS[1]
      local userKey = KEYS[2]
      local ipKey = KEYS[3]
      local templateMax = tonumber(ARGV[1])
      local userMax = tonumber(ARGV[2])
      local ipMax = tonumber(ARGV[3])
      local ttl = tonumber(ARGV[4])
      
      -- Check template hash
      local templateCount = redis.call('INCR', templateKey)
      if templateCount == 1 then
        redis.call('EXPIRE', templateKey, ttl)
      end
      
      -- Check user rate limit
      local userCount = 0
      if userKey ~= '' then
        userCount = redis.call('INCR', userKey)
        if userCount == 1 then
          redis.call('EXPIRE', userKey, ttl)
        end
      end
      
      -- Check IP rate limit
      local ipCount = 0
      if ipKey ~= '' then
        ipCount = redis.call('INCR', ipKey)
        if ipCount == 1 then
          redis.call('EXPIRE', ipKey, ttl)
        end
      end
      
      -- Determine if spam
      local isSpam = 0
      local reason = ''
      
      if templateCount > templateMax then
        isSpam = 1
        reason = 'template_hash'
      elseif userKey ~= '' and userCount > userMax then
        isSpam = 1
        reason = 'user_rate_limit'
      elseif ipKey ~= '' and ipCount > ipMax then
        isSpam = 1
        reason = 'ip_rate_limit'
      end
      
      return {isSpam, templateCount, userCount, ipCount, reason}
    `;
    
    const templateKey = `spam:template:${templateHash}`;
    const userKey = userId ? `spam:rate:user:${userId}` : '';
    const ipKey = ipAddress ? `spam:rate:ip:${ipAddress}` : '';
    
    const result = await this.redis.eval(
      script,
      3,
      templateKey,
      userKey,
      ipKey,
      templateThreshold,
      rateLimitUser,
      rateLimitIP,
      timeWindow
    ) as [number, number, number, number, string];
    
    return {
      isSpam: result[0] === 1,
      reason: result[4] || undefined,
      templateHash,
      occurrences: result[1],
      checks: {
        rateLimit: (userId && result[2] > rateLimitUser) || 
                   (ipAddress && result[3] > rateLimitIP),
        templateHash: result[1] > templateThreshold,
        contentSimilarity: false // Can add SimHash check here
      }
    };
  }
  
  /**
   * Whitelist a template hash (for legitimate repeated messages)
   */
  async whitelistTemplate(templateHash: string, ttl: number = 86400): Promise<void> {
    await this.redis.setex(`spam:whitelist:${templateHash}`, ttl, '1');
  }
  
  /**
   * Check if template is whitelisted
   */
  async isWhitelisted(templateHash: string): Promise<boolean> {
    const result = await this.redis.get(`spam:whitelist:${templateHash}`);
    return result === '1';
  }
}

// Usage in Express middleware
export function spamDetectionMiddleware(options?: SpamDetectionOptions) {
  const spamService = new SpamDetectionService(redis);
  
  return async (req: Request, res: Response, next: NextFunction) => {
    const message = req.body.message || req.body.content || req.body.text;
    
    if (!message || typeof message !== 'string') {
      return next();
    }
    
    // Check whitelist first (for system messages, etc.)
    const templateHash = templateHasher.getTemplateHash(message);
    if (await spamService.isWhitelisted(templateHash)) {
      return next();
    }
    
    const result = await spamService.detectSpam(
      message,
      req.user?.id,
      req.ip,
      options
    );
    
    if (result.isSpam) {
      logger.warn('Spam detected', {
        userId: req.user?.id,
        ip: req.ip,
        templateHash: result.templateHash,
        occurrences: result.occurrences,
        reason: result.reason
      });
      
      return res.status(429).json({
        error: 'Spam detected',
        message: 'This message appears to be spam. Please try again later.',
        retryAfter: 3600
      });
    }
    
    // Attach result to request for logging
    req.spamCheck = result;
    next();
  };
}
```

### Template Extraction Examples

```typescript
// Example: Different messages, same template

const messages = [
  "Hello user123, your order #456 is ready!",
  "Hello user789, your order #123 is ready!",
  "Hello user456, your order #789 is ready!"
];

const hasher = new TemplateHasher();

for (const msg of messages) {
  const hash = hasher.getTemplateHash(msg);
  console.log(`Message: ${msg}`);
  console.log(`Template Hash: ${hash}`);
  console.log('---');
}

// Output:
// Message: Hello user123, your order #456 is ready!
// Template Hash: a1b2c3d4e5f6... (same for all)
// ---
// Message: Hello user789, your order #123 is ready!
// Template Hash: a1b2c3d4e5f6... (same!)
// ---
// Message: Hello user456, your order #789 is ready!
// Template Hash: a1b2c3d4e5f6... (same!)
```

---

## Best Practices

### 1. **Template Extraction Tuning**

- **Start simple**: Use pattern-based extraction
- **Tune patterns**: Add domain-specific patterns as you see spam
- **Monitor false positives**: Adjust thresholds based on real data
- **Whitelist legitimate templates**: System messages, notifications

### 2. **Redis Key Strategy**

```typescript
// Good key naming
`spam:template:${templateHash}`           // Template-based
`spam:rate:user:${userId}`                // User-based
`spam:rate:ip:${ipAddress}`              // IP-based
`spam:simhash:${simHash}`                 // SimHash-based
`spam:whitelist:${templateHash}`         // Whitelist

// Bad key naming (too generic)
`spam:${userId}`                         // Unclear purpose
`spam:${message}`                        // Too long, not normalized
```

### 3. **TTL Management**

- **Short TTL** (1 hour): For rate limiting
- **Medium TTL** (24 hours): For template hashes
- **Long TTL** (7 days): For known spam templates
- **No TTL**: For whitelisted templates (until manually removed)

### 4. **Monitoring & Alerting**

```typescript
// Track spam detection metrics
class SpamMetrics {
  async recordSpamDetection(result: SpamCheckResult): Promise<void> {
    // Increment counters
    await redis.incr(`metrics:spam:detected:${result.reason}`);
    await redis.incr(`metrics:spam:template:${result.templateHash}`);
    
    // Alert if spike detected
    const recentDetections = await redis.get(`metrics:spam:recent`);
    if (parseInt(recentDetections || '0') > 100) {
      await alertManager.sendAlert({
        severity: 'warning',
        message: 'High spam detection rate',
        templateHash: result.templateHash
      });
    }
  }
}
```

### 5. **False Positive Handling**

```typescript
// Allow users to report false positives
async function reportFalsePositive(
  templateHash: string,
  userId: string
): Promise<void> {
  // Log for review
  await logger.info('False positive reported', {
    templateHash,
    userId,
    timestamp: new Date()
  });
  
  // Temporarily whitelist for this user
  await redis.setex(
    `spam:whitelist:user:${userId}:${templateHash}`,
    3600, // 1 hour
    '1'
  );
  
  // Queue for manual review
  await reviewQueue.add({
    type: 'false_positive',
    templateHash,
    userId
  });
}
```

---

## Summary

### ✅ Template Hashing is the Right Approach

**Why**:
- ✅ Identifies same message pattern despite different values
- ✅ Fast (sub-millisecond with Redis)
- ✅ Scalable (works at any volume)
- ✅ Handles dynamic content gracefully

### Recommended Architecture

```
Message → Template Extraction → Template Hash → Redis Lookup
                                              ↓
                                    < 1ms response
                                    (vs 100-1000ms with DB)
```

### Technology Stack

- **Template Hashing**: Custom extractor (pattern-based or ML-based)
- **Storage**: Redis (not MariaDB/PostgreSQL)
- **Rate Limiting**: Redis counters with TTL
- **Fuzzy Matching**: SimHash or MinHash (optional, for advanced detection)

### Performance Gains

- **MariaDB approach**: 100-1000ms per check
- **Redis + Template Hash**: < 1ms per check
- **Improvement**: 100-1000x faster

---

## See Also

- [Security](security.md) - Rate limiting and security practices
- [Messaging & Search Strategy](messaging-and-search-strategy.md) - Redis usage patterns
- [Intelligent Caching](intelligent-caching.md) - Caching strategies

---

**Last Reviewed**: 2026-01-28  
**Next Review**: 2026-04-28 (Quarterly)
