# Database Layer Documentation
## Connection Pooling, Query Optimization, Migrations & Backup Strategy

---
**Status**: 💡 **GUIDANCE** - Best practices for database management  
**Priority**: 🔴 **P0** - Critical for production reliability  
**Last Updated**: 2026-01-28  
**Owned By**: Backend Team, DBA  

---

## Table of Contents

1. [Overview](#overview)
2. [Connection Pooling](#connection-pooling)
3. [Query Optimization](#query-optimization)
4. [Database Migrations](#database-migrations)
5. [Transaction Management](#transaction-management)
6. [Backup & Recovery](#backup--recovery)
7. [Database Monitoring](#database-monitoring)
8. [Read Replicas](#read-replicas)
9. [Best Practices](#best-practices)
10. [Troubleshooting](#troubleshooting)

---

## Overview

### Database Stack

```
Application Layer
    ↓
Connection Pool (pg-pool / Prisma)
    ↓
Primary Database (Write)
    ↓ replication
Read Replicas (Read-only)
    ↓ backup
Point-in-Time Recovery (PITR)
```

### Key Metrics

| Metric | Target | Warning | Critical |
|--------|--------|---------|----------|
| **Connection Pool Usage** | < 70% | 70-85% | > 85% |
| **Query Time P95** | < 100ms | 100-500ms | > 500ms |
| **Slow Queries** | 0/min | 1-10/min | > 10/min |
| **Active Connections** | < 80 | 80-100 | > 100 |
| **Replication Lag** | < 1s | 1-10s | > 10s |
| **Disk Usage** | < 70% | 70-85% | > 85% |

---

## Connection Pooling

### Why Connection Pooling?

```
Without pooling:
- Each request opens new connection (100-500ms overhead)
- 1000 concurrent requests = 1000 DB connections
- Database crashes under load

With pooling:
- Reuse existing connections (< 1ms overhead)
- 1000 requests share 20 connections
- Stable, predictable performance
```

### PostgreSQL with pg-pool

```typescript
import { Pool, PoolConfig } from 'pg';
import { logger } from './logger';

const poolConfig: PoolConfig = {
  // Connection details
  host: process.env.DB_HOST,
  port: parseInt(process.env.DB_PORT || '5432'),
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  
  // Pool configuration
  max: 20,                    // Maximum connections in pool
  min: 5,                     // Minimum idle connections
  idleTimeoutMillis: 30000,   // Close idle connections after 30s
  connectionTimeoutMillis: 5000, // Fail fast if no connection available
  
  // Query configuration
  statement_timeout: 10000,   // Kill queries after 10s
  query_timeout: 10000,       // Client-side timeout
  
  // SSL configuration (production)
  ssl: process.env.NODE_ENV === 'production' ? {
    rejectUnauthorized: true,
    ca: process.env.DB_SSL_CA,
  } : false,
  
  // Application name for monitoring
  application_name: 'myapp-api',
};

export const pool = new Pool(poolConfig);

// Monitor pool events
pool.on('connect', (client) => {
  logger.debug('New database connection established');
});

pool.on('acquire', (client) => {
  logger.debug('Connection acquired from pool');
});

pool.on('error', (err, client) => {
  logger.error({ err }, 'Unexpected database error');
});

pool.on('remove', (client) => {
  logger.debug('Connection removed from pool');
});

// Graceful shutdown
export async function closePool(): Promise<void> {
  logger.info('Closing database connection pool');
  await pool.end();
  logger.info('Database connection pool closed');
}

// Health check
export async function checkDatabaseHealth(): Promise<boolean> {
  try {
    const result = await pool.query('SELECT 1');
    return result.rows.length === 1;
  } catch (error) {
    logger.error({ error }, 'Database health check failed');
    return false;
  }
}
```

### Prisma Connection Pooling

```typescript
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient({
  log: [
    { level: 'query', emit: 'event' },
    { level: 'error', emit: 'event' },
    { level: 'warn', emit: 'event' },
  ],
  datasources: {
    db: {
      url: process.env.DATABASE_URL,
    },
  },
});

// Connection pool configuration in DATABASE_URL
// postgresql://user:pass@host:5432/db?connection_limit=20&pool_timeout=10

// Monitor queries
prisma.$on('query', (e) => {
  if (e.duration > 1000) {
    logger.warn({ 
      query: e.query, 
      duration: e.duration,
      params: e.params 
    }, 'Slow query detected');
  }
});

prisma.$on('error', (e) => {
  logger.error({ error: e }, 'Prisma error');
});

// Graceful shutdown
export async function closePrisma(): Promise<void> {
  await prisma.$disconnect();
}
```

### Pool Size Calculation

```typescript
/**
 * Calculate optimal pool size
 * 
 * Formula: connections = ((core_count * 2) + effective_spindle_count)
 * 
 * Example:
 * - 4 CPU cores
 * - SSD (spindle_count = 1)
 * - Result: (4 * 2) + 1 = 9 connections
 * 
 * Add buffer: 9 * 1.5 = ~14 connections
 * Round up: 15-20 connections
 */

function calculatePoolSize(): number {
  const cpuCount = require('os').cpus().length;
  const isDisk = process.env.DB_STORAGE_TYPE === 'disk';
  const spindleCount = isDisk ? 4 : 1; // HDD vs SSD
  
  const optimal = (cpuCount * 2) + spindleCount;
  const withBuffer = Math.ceil(optimal * 1.5);
  
  return withBuffer;
}
```

### Monitoring Pool Usage

```typescript
import { Counter, Gauge } from 'prom-client';

export const dbMetrics = {
  poolSize: new Gauge({
    name: 'db_pool_size',
    help: 'Total connections in pool'
  }),
  
  poolAvailable: new Gauge({
    name: 'db_pool_available',
    help: 'Available connections in pool'
  }),
  
  poolWaiting: new Gauge({
    name: 'db_pool_waiting',
    help: 'Clients waiting for connection'
  }),
  
  connectionErrors: new Counter({
    name: 'db_connection_errors_total',
    help: 'Total connection errors'
  }),
};

// Update metrics every 10 seconds
setInterval(() => {
  dbMetrics.poolSize.set(pool.totalCount);
  dbMetrics.poolAvailable.set(pool.idleCount);
  dbMetrics.poolWaiting.set(pool.waitingCount);
}, 10000);
```

---

## Query Optimization

### 1. Use Indexes Effectively

```sql
-- Before: Full table scan (slow)
SELECT * FROM users WHERE email = 'user@example.com';
EXPLAIN ANALYZE;
-- Seq Scan on users (cost=0.00..1234.00 rows=1 width=123) (actual time=45.123..45.123 rows=1)

-- After: Index scan (fast)
CREATE INDEX idx_users_email ON users(email);
SELECT * FROM users WHERE email = 'user@example.com';
EXPLAIN ANALYZE;
-- Index Scan using idx_users_email (cost=0.42..8.44 rows=1 width=123) (actual time=0.023..0.024 rows=1)
```

### Common Index Types

```sql
-- B-tree (default, most common)
CREATE INDEX idx_users_created_at ON users(created_at);

-- Partial index (only index subset)
CREATE INDEX idx_active_users ON users(email) 
WHERE deleted_at IS NULL;

-- Composite index (multiple columns)
CREATE INDEX idx_posts_user_status ON posts(user_id, status);

-- Covering index (include extra columns)
CREATE INDEX idx_users_email_covering ON users(email) 
INCLUDE (name, created_at);

-- Full-text search
CREATE INDEX idx_posts_content_fts ON posts 
USING gin(to_tsvector('english', content));

-- JSON/JSONB
CREATE INDEX idx_metadata_tags ON users 
USING gin(metadata);
```

### 2. Avoid N+1 Queries

```typescript
// BAD: N+1 queries
async function getUsersWithPosts(): Promise<UserWithPosts[]> {
  const users = await db.query('SELECT * FROM users');
  
  // This runs 1 query per user (N queries!)
  for (const user of users) {
    user.posts = await db.query(
      'SELECT * FROM posts WHERE user_id = $1',
      [user.id]
    );
  }
  
  return users;
}

// GOOD: Single query with JOIN
async function getUsersWithPosts(): Promise<UserWithPosts[]> {
  const result = await db.query(`
    SELECT 
      u.*,
      json_agg(
        json_build_object(
          'id', p.id,
          'title', p.title,
          'content', p.content
        )
      ) as posts
    FROM users u
    LEFT JOIN posts p ON p.user_id = u.id
    GROUP BY u.id
  `);
  
  return result.rows;
}

// GOOD: Prisma with includes
async function getUsersWithPosts(): Promise<UserWithPosts[]> {
  return await prisma.user.findMany({
    include: {
      posts: true
    }
  });
}
```

### 3. Use EXPLAIN ANALYZE

```typescript
async function analyzeQuery(query: string, params: any[] = []): Promise<void> {
  const explainQuery = `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) ${query}`;
  const result = await pool.query(explainQuery, params);
  
  const plan = result.rows[0]['QUERY PLAN'][0];
  const executionTime = plan['Execution Time'];
  const planningTime = plan['Planning Time'];
  
  logger.info({
    query,
    executionTime,
    planningTime,
    plan
  }, 'Query analysis');
  
  // Alert on slow queries
  if (executionTime > 1000) {
    logger.warn({ query, executionTime }, 'Slow query detected');
  }
}

// Usage
await analyzeQuery(
  'SELECT * FROM users WHERE email = $1',
  ['user@example.com']
);
```

### 4. Pagination Best Practices

```typescript
// BAD: OFFSET pagination (slow for large offsets)
async function getUsers(page: number, limit: number) {
  const offset = (page - 1) * limit;
  return await db.query(
    'SELECT * FROM users ORDER BY id LIMIT $1 OFFSET $2',
    [limit, offset]
  );
  // OFFSET 10000 means scanning 10000 rows to skip them!
}

// GOOD: Cursor-based pagination (always fast)
async function getUsers(cursor: number | null, limit: number) {
  const query = cursor
    ? 'SELECT * FROM users WHERE id > $1 ORDER BY id LIMIT $2'
    : 'SELECT * FROM users ORDER BY id LIMIT $1';
  
  const params = cursor ? [cursor, limit] : [limit];
  return await db.query(query, params);
}

// Usage
const page1 = await getUsers(null, 20);
const lastId = page1.rows[page1.rows.length - 1].id;
const page2 = await getUsers(lastId, 20); // Fast even for page 1000
```

### 5. Batch Operations

```typescript
// BAD: Individual inserts (slow)
async function insertUsers(users: User[]): Promise<void> {
  for (const user of users) {
    await db.query(
      'INSERT INTO users (name, email) VALUES ($1, $2)',
      [user.name, user.email]
    );
  }
  // 100 users = 100 round trips to database
}

// GOOD: Batch insert (fast)
async function insertUsers(users: User[]): Promise<void> {
  const values = users
    .map((_, i) => `($${i * 2 + 1}, $${i * 2 + 2})`)
    .join(', ');
  
  const params = users.flatMap(u => [u.name, u.email]);
  
  await db.query(
    `INSERT INTO users (name, email) VALUES ${values}`,
    params
  );
  // 100 users = 1 round trip
}

// GOOD: Use COPY for massive inserts
async function bulkInsertUsers(users: User[]): Promise<void> {
  const { Readable } = require('stream');
  
  const stream = Readable.from(
    users.map(u => `${u.name}\t${u.email}\n`)
  );
  
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    await client.query('COPY users (name, email) FROM STDIN');
    stream.pipe(client.query(copyFrom('STDIN')));
    await client.query('COMMIT');
  } finally {
    client.release();
  }
  // 1 million users in seconds
}
```

### 6. Query Timeout

```typescript
// Set statement timeout
await pool.query("SET statement_timeout = '5s'");

// Or per-query timeout
const client = await pool.connect();
try {
  await client.query("SET statement_timeout = '10s'");
  const result = await client.query('SELECT * FROM huge_table');
  return result.rows;
} catch (error) {
  if (error.code === '57014') {
    logger.error('Query timeout exceeded');
  }
  throw error;
} finally {
  client.release();
}
```

---

## Database Migrations

### Migration Strategy

```typescript
import { Pool } from 'pg';
import * as fs from 'fs';
import * as path from 'path';

class MigrationManager {
  private pool: Pool;
  private migrationsDir: string;
  
  constructor(pool: Pool, migrationsDir: string) {
    this.pool = pool;
    this.migrationsDir = migrationsDir;
  }
  
  async init(): Promise<void> {
    // Create migrations table
    await this.pool.query(`
      CREATE TABLE IF NOT EXISTS migrations (
        id SERIAL PRIMARY KEY,
        name VARCHAR(255) UNIQUE NOT NULL,
        applied_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
      )
    `);
  }
  
  async migrate(): Promise<void> {
    await this.init();
    
    // Get applied migrations
    const { rows } = await this.pool.query(
      'SELECT name FROM migrations ORDER BY id'
    );
    const applied = new Set(rows.map(r => r.name));
    
    // Get pending migrations
    const files = fs.readdirSync(this.migrationsDir)
      .filter(f => f.endsWith('.sql'))
      .sort();
    
    const pending = files.filter(f => !applied.has(f));
    
    if (pending.length === 0) {
      logger.info('No pending migrations');
      return;
    }
    
    // Apply pending migrations
    for (const file of pending) {
      logger.info({ file }, 'Applying migration');
      await this.applyMigration(file);
    }
    
    logger.info({ count: pending.length }, 'Migrations completed');
  }
  
  private async applyMigration(filename: string): Promise<void> {
    const filepath = path.join(this.migrationsDir, filename);
    const sql = fs.readFileSync(filepath, 'utf8');
    
    const client = await this.pool.connect();
    try {
      await client.query('BEGIN');
      
      // Apply migration
      await client.query(sql);
      
      // Record migration
      await client.query(
        'INSERT INTO migrations (name) VALUES ($1)',
        [filename]
      );
      
      await client.query('COMMIT');
      logger.info({ filename }, 'Migration applied successfully');
    } catch (error) {
      await client.query('ROLLBACK');
      logger.error({ filename, error }, 'Migration failed');
      throw error;
    } finally {
      client.release();
    }
  }
  
  async rollback(steps: number = 1): Promise<void> {
    const { rows } = await this.pool.query(
      'SELECT name FROM migrations ORDER BY id DESC LIMIT $1',
      [steps]
    );
    
    for (const row of rows) {
      logger.info({ migration: row.name }, 'Rolling back migration');
      await this.rollbackMigration(row.name);
    }
  }
  
  private async rollbackMigration(name: string): Promise<void> {
    // Load rollback SQL (convention: migration_name.down.sql)
    const downFile = name.replace('.sql', '.down.sql');
    const filepath = path.join(this.migrationsDir, downFile);
    
    if (!fs.existsSync(filepath)) {
      throw new Error(`Rollback file not found: ${downFile}`);
    }
    
    const sql = fs.readFileSync(filepath, 'utf8');
    
    const client = await this.pool.connect();
    try {
      await client.query('BEGIN');
      await client.query(sql);
      await client.query('DELETE FROM migrations WHERE name = $1', [name]);
      await client.query('COMMIT');
      logger.info({ name }, 'Migration rolled back');
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }
}

// Usage
const migrationManager = new MigrationManager(
  pool,
  './migrations'
);

// Run migrations on startup
await migrationManager.migrate();
```

### Migration File Format

```sql
-- migrations/001_create_users_table.sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  name VARCHAR(255) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);

-- migrations/001_create_users_table.down.sql
DROP INDEX IF EXISTS idx_users_email;
DROP TABLE IF EXISTS users;
```

### Prisma Migrations

```bash
# Create migration
npx prisma migrate dev --name add_user_profile

# Apply migrations in production
npx prisma migrate deploy

# Generate Prisma Client
npx prisma generate
```

---

## Transaction Management

### Basic Transactions

```typescript
async function transferMoney(
  fromUserId: string,
  toUserId: string,
  amount: number
): Promise<void> {
  const client = await pool.connect();
  
  try {
    await client.query('BEGIN');
    
    // Deduct from sender
    await client.query(
      'UPDATE accounts SET balance = balance - $1 WHERE user_id = $2',
      [amount, fromUserId]
    );
    
    // Add to receiver
    await client.query(
      'UPDATE accounts SET balance = balance + $1 WHERE user_id = $2',
      [amount, toUserId]
    );
    
    await client.query('COMMIT');
    logger.info({ fromUserId, toUserId, amount }, 'Transfer completed');
  } catch (error) {
    await client.query('ROLLBACK');
    logger.error({ error }, 'Transfer failed, rolled back');
    throw error;
  } finally {
    client.release();
  }
}
```

### Isolation Levels

```typescript
enum IsolationLevel {
  READ_UNCOMMITTED = 'READ UNCOMMITTED',
  READ_COMMITTED = 'READ COMMITTED',      // PostgreSQL default
  REPEATABLE_READ = 'REPEATABLE READ',
  SERIALIZABLE = 'SERIALIZABLE',
}

async function executeWithIsolation<T>(
  isolationLevel: IsolationLevel,
  fn: (client: PoolClient) => Promise<T>
): Promise<T> {
  const client = await pool.connect();
  
  try {
    await client.query('BEGIN');
    await client.query(`SET TRANSACTION ISOLATION LEVEL ${isolationLevel}`);
    
    const result = await fn(client);
    
    await client.query('COMMIT');
    return result;
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}

// Usage: Prevent concurrent updates
await executeWithIsolation(
  IsolationLevel.SERIALIZABLE,
  async (client) => {
    const { rows } = await client.query(
      'SELECT balance FROM accounts WHERE user_id = $1',
      [userId]
    );
    
    if (rows[0].balance < amount) {
      throw new Error('Insufficient funds');
    }
    
    await client.query(
      'UPDATE accounts SET balance = balance - $1 WHERE user_id = $2',
      [amount, userId]
    );
  }
);
```

### Deadlock Handling

```typescript
async function withDeadlockRetry<T>(
  fn: () => Promise<T>,
  maxRetries: number = 3
): Promise<T> {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error: any) {
      // PostgreSQL deadlock error code
      if (error.code === '40P01' && attempt < maxRetries) {
        const delay = Math.min(100 * Math.pow(2, attempt), 1000);
        logger.warn({ attempt, delay }, 'Deadlock detected, retrying');
        await new Promise(resolve => setTimeout(resolve, delay));
        continue;
      }
      throw error;
    }
  }
  
  throw new Error('Max retries exceeded');
}

// Usage
await withDeadlockRetry(async () => {
  await transferMoney(fromId, toId, amount);
});
```

---

## Backup & Recovery

### Automated Backups

```bash
#!/bin/bash
# backup.sh - Daily automated backup script

BACKUP_DIR="/backups/postgresql"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/backup_$DATE.sql.gz"

# Create backup
pg_dump \
  -h $DB_HOST \
  -U $DB_USER \
  -d $DB_NAME \
  --format=custom \
  --compress=9 \
  | gzip > $BACKUP_FILE

# Verify backup
if [ -f "$BACKUP_FILE" ]; then
  echo "Backup created: $BACKUP_FILE"
  
  # Upload to S3
  aws s3 cp $BACKUP_FILE s3://my-backups/postgresql/
  
  # Keep only last 7 days locally
  find $BACKUP_DIR -name "backup_*.sql.gz" -mtime +7 -delete
else
  echo "Backup failed!"
  exit 1
fi
```

### Point-in-Time Recovery (PITR)

```bash
# Enable WAL archiving in postgresql.conf
archive_mode = on
archive_command = 'aws s3 cp %p s3://my-wal-archive/%f'
wal_level = replica

# Create base backup
pg_basebackup \
  -h $DB_HOST \
  -U $DB_USER \
  -D /backup/base \
  -Ft -z -P

# Restore to specific point in time
# 1. Stop PostgreSQL
# 2. Restore base backup
# 3. Create recovery.conf
cat > recovery.conf << EOF
restore_command = 'aws s3 cp s3://my-wal-archive/%f %p'
recovery_target_time = '2026-01-28 10:00:00'
EOF
# 4. Start PostgreSQL
```

### Backup Monitoring

```typescript
import { execSync } from 'child_process';

export async function verifyBackup(): Promise<boolean> {
  try {
    // List recent backups
    const backups = execSync('aws s3 ls s3://my-backups/postgresql/')
      .toString()
      .split('\n')
      .filter(line => line.includes('.sql.gz'));
    
    if (backups.length === 0) {
      logger.error('No backups found');
      return false;
    }
    
    // Check last backup age
    const lastBackup = backups[backups.length - 1];
    const match = lastBackup.match(/backup_(\d{8}_\d{6})/);
    
    if (!match) {
      logger.error('Cannot parse backup timestamp');
      return false;
    }
    
    const timestamp = match[1];
    const backupDate = new Date(
      timestamp.substring(0, 4) + '-' +
      timestamp.substring(4, 6) + '-' +
      timestamp.substring(6, 8)
    );
    
    const age = Date.now() - backupDate.getTime();
    const maxAge = 25 * 60 * 60 * 1000; // 25 hours
    
    if (age > maxAge) {
      logger.error({ age, lastBackup }, 'Backup is too old');
      return false;
    }
    
    logger.info({ lastBackup }, 'Backup verification passed');
    return true;
  } catch (error) {
    logger.error({ error }, 'Backup verification failed');
    return false;
  }
}

// Schedule backup verification
setInterval(verifyBackup, 3600000); // Every hour
```

---

## Database Monitoring

### Metrics Collection

```typescript
import { Gauge, Histogram, Counter } from 'prom-client';

export const dbMetrics = {
  queryDuration: new Histogram({
    name: 'db_query_duration_seconds',
    help: 'Database query duration',
    labelNames: ['query_type', 'table'],
    buckets: [0.01, 0.05, 0.1, 0.5, 1, 5]
  }),
  
  activeConnections: new Gauge({
    name: 'db_active_connections',
    help: 'Number of active database connections'
  }),
  
  slowQueries: new Counter({
    name: 'db_slow_queries_total',
    help: 'Total number of slow queries',
    labelNames: ['query_type']
  }),
  
  errors: new Counter({
    name: 'db_errors_total',
    help: 'Total database errors',
    labelNames: ['error_type']
  }),
  
  replicationLag: new Gauge({
    name: 'db_replication_lag_seconds',
    help: 'Replication lag in seconds'
  }),
};

// Monitor slow queries
pool.on('query', (query) => {
  const duration = query.duration / 1000; // Convert to seconds
  
  dbMetrics.queryDuration.observe(
    { query_type: query.type, table: query.table },
    duration
  );
  
  if (duration > 1) {
    dbMetrics.slowQueries.inc({ query_type: query.type });
    logger.warn({ query, duration }, 'Slow query detected');
  }
});

// Monitor connections
setInterval(async () => {
  const { rows } = await pool.query(
    'SELECT count(*) FROM pg_stat_activity WHERE state = \'active\''
  );
  dbMetrics.activeConnections.set(parseInt(rows[0].count));
}, 10000);
```

### Query Logging

```typescript
import { QueryConfig, QueryResult } from 'pg';

export async function queryWithLogging(
  query: string | QueryConfig,
  values?: any[]
): Promise<QueryResult> {
  const start = Date.now();
  const queryText = typeof query === 'string' ? query : query.text;
  
  try {
    const result = await pool.query(query, values);
    const duration = Date.now() - start;
    
    logger.debug({
      query: queryText,
      duration,
      rows: result.rowCount
    }, 'Query executed');
    
    return result;
  } catch (error) {
    const duration = Date.now() - start;
    
    logger.error({
      query: queryText,
      duration,
      error
    }, 'Query failed');
    
    throw error;
  }
}
```

---

## Read Replicas

### Configuration

```typescript
import { Pool } from 'pg';

// Primary database (writes)
export const primaryPool = new Pool({
  host: process.env.DB_PRIMARY_HOST,
  port: 5432,
  max: 20,
  // ... other config
});

// Read replica (reads)
export const replicaPool = new Pool({
  host: process.env.DB_REPLICA_HOST,
  port: 5432,
  max: 40, // Can be larger for read replicas
  // ... other config
});

// Smart query router
export async function query(
  sql: string,
  params: any[] = [],
  forceWrite: boolean = false
): Promise<QueryResult> {
  // Determine if this is a write query
  const isWrite = /^(INSERT|UPDATE|DELETE|CREATE|ALTER|DROP)/i.test(sql);
  
  const pool = (isWrite || forceWrite) ? primaryPool : replicaPool;
  
  return await pool.query(sql, params);
}

// Usage
await query('SELECT * FROM users'); // → replica
await query('INSERT INTO users (name) VALUES ($1)', ['John']); // → primary
await query('SELECT * FROM users WHERE id = $1', [123], true); // → primary (consistent read)
```

### Replication Lag Monitoring

```typescript
async function checkReplicationLag(): Promise<number> {
  try {
    // Query primary for current WAL position
    const { rows: primaryRows } = await primaryPool.query(
      'SELECT pg_current_wal_lsn() as lsn'
    );
    
    // Query replica for replay position
    const { rows: replicaRows } = await replicaPool.query(
      'SELECT pg_last_wal_replay_lsn() as lsn'
    );
    
    // Calculate lag in bytes
    const { rows: lagRows } = await replicaPool.query(
      'SELECT $1::pg_lsn - $2::pg_lsn as lag',
      [primaryRows[0].lsn, replicaRows[0].lsn]
    );
    
    const lagBytes = parseInt(lagRows[0].lag);
    const lagSeconds = lagBytes / (16 * 1024 * 1024); // Rough estimate
    
    dbMetrics.replicationLag.set(lagSeconds);
    
    if (lagSeconds > 10) {
      logger.warn({ lagSeconds }, 'High replication lag');
    }
    
    return lagSeconds;
  } catch (error) {
    logger.error({ error }, 'Failed to check replication lag');
    return -1;
  }
}

// Check every 30 seconds
setInterval(checkReplicationLag, 30000);
```

---

## Best Practices

### 1. Always Use Prepared Statements

```typescript
// BAD: SQL injection vulnerable
const email = req.body.email;
await pool.query(`SELECT * FROM users WHERE email = '${email}'`);

// GOOD: Parameterized query
await pool.query(
  'SELECT * FROM users WHERE email = $1',
  [email]
);
```

### 2. Close Connections Properly

```typescript
// BAD: Connection leak
const client = await pool.connect();
const result = await client.query('SELECT * FROM users');
// Forgot to release!

// GOOD: Always release in finally
const client = await pool.connect();
try {
  const result = await client.query('SELECT * FROM users');
  return result.rows;
} finally {
  client.release();
}
```

### 3. Use Transactions for Related Operations

```typescript
// BAD: No transaction, inconsistent state possible
await pool.query('INSERT INTO orders (user_id, total) VALUES ($1, $2)', [userId, total]);
await pool.query('UPDATE users SET order_count = order_count + 1 WHERE id = $1', [userId]);
// What if second query fails?

// GOOD: Atomic transaction
const client = await pool.connect();
try {
  await client.query('BEGIN');
  await client.query('INSERT INTO orders (user_id, total) VALUES ($1, $2)', [userId, total]);
  await client.query('UPDATE users SET order_count = order_count + 1 WHERE id = $1', [userId]);
  await client.query('COMMIT');
} catch (error) {
  await client.query('ROLLBACK');
  throw error;
} finally {
  client.release();
}
```

### 4. Monitor and Optimize

```typescript
// Log slow queries
pool.on('query', (query) => {
  if (query.duration > 100) {
    logger.warn({
      query: query.text,
      duration: query.duration,
      params: query.values
    }, 'Slow query detected');
  }
});
```

---

## Troubleshooting

### Connection Pool Exhausted

**Symptoms**: "Sorry, too many clients already"

```typescript
// Check pool status
console.log('Total:', pool.totalCount);
console.log('Idle:', pool.idleCount);
console.log('Waiting:', pool.waitingCount);

// Solution 1: Increase pool size
const pool = new Pool({ max: 40 });

// Solution 2: Find connection leaks
// Look for queries without .release()
```

### Slow Queries

```sql
-- Find slow queries
SELECT 
  query,
  calls,
  total_time,
  mean_time,
  max_time
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 10;

-- Find missing indexes
SELECT 
  schemaname,
  tablename,
  idx_scan,
  seq_scan,
  seq_scan - idx_scan as too_much_seq
FROM pg_stat_user_tables
WHERE seq_scan - idx_scan > 1000
ORDER BY too_much_seq DESC;
```

### High Replication Lag

```bash
# Check replication status
SELECT * FROM pg_stat_replication;

# Check for long-running queries blocking replication
SELECT 
  pid,
  usename,
  state,
  query_start,
  query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY query_start;

# Kill blocking query
SELECT pg_terminate_backend(pid);
```

---

## Summary

### Checklist

- [ ] Connection pooling configured (max 20-40)
- [ ] Query timeout set (10s default)
- [ ] Indexes on all foreign keys
- [ ] Indexes on frequently queried columns
- [ ] N+1 queries eliminated
- [ ] Pagination using cursors, not OFFSET
- [ ] Batch inserts for bulk operations
- [ ] Migration system in place
- [ ] Automated daily backups
- [ ] Backup verification scheduled
- [ ] Database metrics exported
- [ ] Slow query logging enabled
- [ ] Read replicas for scale (if needed)
- [ ] Replication lag monitoring

---

## Next Steps

1. **Read**: [Intelligent Caching](INTELLIGENT_CACHING.md) for query result caching
2. **Read**: [Performance & Scalability](PERFORMANCE_SCALABILITY.md) for load balancing
3. **Implement**: Database metrics endpoint
4. **Set up**: Automated backup testing
5. **Review**: All queries with EXPLAIN ANALYZE

---

**Questions?** Check [Integration Guide](INTEGRATION_GUIDE.md#database-integration) or ask in #database
