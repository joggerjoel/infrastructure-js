# Logging Configuration

---
**Status**: ✅ **IMPLEMENTED** - Pino-based structured logging with file rotation  
**Priority**: 🔴 **P0.7** - Critical for debugging production issues  
**Last Updated**: 2026-01-28  
**Owned By**: Infrastructure Team  

---

Services use [Pino](https://getpino.io/) for structured logging with file rotation and compression.

**In this project ({app_name}):** Logging is implemented in **`logger.js`** at the project root. Services use `const { log, warn, error } = require('./logger')`. Log files use the **`{app_name}-YYYYMMDD_HHMMSS`** prefix (e.g. `{app_name}-20260127_191934.log`) under `logs/YYYY/YYYYMM/YYYYMMDD/`. Set `LOG_DIR`, `LOG_LEVEL`, and `NODE_ENV` in `.env` as below.

**Related docs**

- **[Structured log attributes and features](#structured-log-attributes-and-features)** — All attributes and features used in log lines (below).

## Features

- **File-based logging**: All logs are written to files in `./logs/` directory (relative to project root)
- **One log file per startup**: Each server restart creates a new log file with format `{app_name}-YYYYMMDD_HHMMSS.log`
- **Same file throughout app lifetime**: The startup log file is used for the entire lifetime of the app instance
- **Size-based rotation**: Logs rotate when they exceed 20MB, creating numbered versions (`.1.log`, `.2.log`, etc.) while the base file continues to be used
- **Compression**: Old log files are automatically compressed using gzip (native Node.js, no Python needed)
- **Pretty console output**: In development, logs are also printed to console with colors
- **Web log viewer**: View logs easily at http://localhost:3001/log-viewer.html

## Log File Structure

Logs are organized in a hierarchical date-based structure:

```
logs/
└── 2026/                                    # Year
    └── 202601/                              # Year-Month
        └── 20260126/                        # Year-Month-Day
            ├── {app_name}-20260126_211530.log          # Current log file (with startup timestamp, used throughout app lifetime)
            ├── {app_name}-20260126_211530.1.log        # Rotated log (when size exceeds 20MB)
            ├── {app_name}-20260126_211530.2.log        # Next rotation
            └── {app_name}-20260126_211530.1.log.gz     # Compressed old log
```

This structure makes it easy to:
- Find logs by date
- Archive old logs by year/month
- Navigate logs chronologically

## Configuration

### Environment Variables

- `LOG_DIR`: Directory for log files (default: `./logs/` relative to project root)
- `LOG_LEVEL`: Logging level - `trace`, `debug`, `info`, `warn`, `error`, `fatal` (default: `debug` in dev, `info` in prod)
- `NODE_ENV`: Set to `production` for JSON logs, otherwise uses pretty console output

### Example `.env` configuration:

```bash
LOG_DIR=/var/log/{app_name}
LOG_LEVEL=info
NODE_ENV=production
```

## Compression

### Automatic Compression

Log files older than 1 day are automatically compressed on server startup using Node.js's built-in `zlib` module (no external dependencies).

### Manual Compression

You can manually compress old logs using the provided script:

```bash
# Compress logs older than 1 day
tsx scripts/compress-logs.ts

# Compress logs older than 7 days
tsx scripts/compress-logs.ts /path/to/logs 7
```

### Cron Job Setup

To compress logs daily via cron (optional, since compression happens on startup):

```bash
# Add to crontab (crontab -e)
# Compress logs older than 1 day at 2 AM daily
0 2 * * * cd /path/to/project && tsx scripts/compress-logs.ts >> /var/log/log-compression.log 2>&1
```

## Log Retention

- **Default**: Logs are kept indefinitely (no automatic deletion)
- **Compressed logs**: Compressed `.gz` files are kept until manually deleted
- **Rotation**: Files rotate only when they exceed 20MB (not daily, since we organize by date in directories)
- **One file per startup**: Each server restart creates a new log file with a unique startup timestamp

## Reading Logs

### Current Log (via symlink)
```bash
tail -f logs/{app_name}-current.log
```

### Specific Log File (by date path)
```bash
# Using the date-based path
tail -f logs/2026/202601/20260126/{app_name}-20260126_211530.log

# Or use the symlink
tail -f logs/{app_name}-current.log
```

### Compressed Logs
```bash
# View compressed log (using date path)
zcat logs/2026/202601/20260126/{app_name}-20260126_211530.2026-01-26.1.log.gz | less

# Or decompress first
gunzip logs/2026/202601/20260126/{app_name}-20260126_211530.2026-01-26.1.log.gz
```

### Browse by Date
```bash
# List all years
ls logs/

# List all months in a year
ls logs/2026/

# List all days in a month
ls logs/2026/202601/

# View logs for a specific day
ls logs/2026/202601/20260126/
```

### JSON Logs (Production)
In production, logs are in JSON format. Use `jq` for pretty printing:

```bash
tail -f logs/{app_name}-current.log | jq
```

## Log Levels

- **trace**: Very detailed debugging information
- **debug**: Debug information (SQL queries, request details)
- **info**: General information (server startup, successful operations)
- **warn**: Warnings (slow queries, missing data)
- **error**: Errors (exceptions, failures)
- **fatal**: Critical errors that may cause the application to crash

## Structured log attributes and features

Every log line is a JSON object. Attributes come from Pino plus the object passed to `logger.info()`, `logger.debug()`, etc.

### Pino standard attributes (every line)

| Attribute   | Type   | Description |
|------------|--------|-------------|
| `level`    | string | `trace`, `debug`, `info`, `warn`, `error`, `fatal` |
| `time`     | string | Timestamp; we use EDT/EST (`America/New_York`) via custom `edtTimeFunction` |
| `msg`      | string | Log message (second argument to `logger.info(obj, msg)`) |
| `pid`      | number | Process ID (unless ignored in pretty output) |
| `hostname` | string | Host name (unless ignored in pretty output) |

### Logger configuration

- **Timestamp**: Custom `edtTimeFunction` — format `YYYY-MM-DDTHH:mm:ss.sss±HH:mm` in America/New_York.
- **Level formatter**: `level` is emitted as `{ level: label }` (label only, no numeric code in our formatter).

### Startup / initialization attributes

| Attribute          | Type   | When | Description |
|-------------------|--------|------|-------------|
| `startupTimestamp`| string | Logger init | `YYYYMMDD_HHMMSS` for this process |
| `logDir`          | string | Logger init | Absolute path to log root |
| `dateDir`         | string | Logger init | `logs/YYYY/YYYYMM/YYYYMMDD` for today |
| `baseLogFile`     | string | Logger init | Full path to `{app_name}-YYYYMMDD_HHMMSS.log` |
| `rotationPolicy`  | string | Logger init | `"size-based only (20MB limit, no daily rotation)"` |
| `logLevel`        | string | Logger init | Effective level (e.g. `debug`, `info`) |
| `isDevelopment`   | bool   | Logger init | From `NODE_ENV !== 'production'` |
| `nodeEnv`         | string | Logger init | `process.env.NODE_ENV \|\| 'development'` |
| `compressedCount` | number | Post-startup | Count of log files compressed (when &gt; 0) |

### Request / query-tracking attributes

| Attribute           | Type    | When | Description |
|--------------------|---------|------|-------------|
| `requestHash`      | string  | API requests | Hash used to correlate frontend request with service; from `X-Request-Id` or generated |
| `xRequestId`       | string? | Request debug | `req.headers['x-request-id']` |
| `XRequestId`       | string? | Request debug | `req.headers['X-Request-Id']` |
| `allHeaders`       | string[]| Request debug | Header names containing `"request"` |
| `trackerExists`    | boolean | Request debug | Whether `getQueryTracker(req)` returned a tracker |
| `trackerQueryCount`| number  | Request debug | Number of queries tracked so far for this request |
| `reqHasTracker`    | boolean | Request debug | Whether `req._queryTracker` exists |
| `hasReq`           | boolean | Request debug | Request object passed to query |
| `countSqlPreview`  | string  | Request debug | First ~100 chars of count SQL |
| `countResult`      | string  | Request debug | `'array'` or `'not array'` |
| `sqlPreview`       | string  | DB query tracking | First portion of executed SQL |
| `paramCount`       | number  | DB query tracking, buildQuery | Number of bound parameters |
| `duration`         | number  | DB query tracking | Query duration (ms) |
| `rowCount`         | number  | DB query tracking | Rows returned |
| `params`           | any[]   | buildQuery debug | Query parameters (may be truncated) |
| `placeholders`     | number  | Query debug | Number of `?` in SQL |
| `endpoint`         | string  | Request history | e.g. `/api/entities/records?...` |
| `queries`          | array   | Request history | Tracked query objects (sql, params, duration, etc.) |
| `queryCount`       | number  | API requests | Number of tracked queries for this request |
| `rawQueriesCount`  | number  | Request debug | Length of `tracker.getQueries()` |
| `rawQueries`       | array   | Request debug | Serialized tracked queries |

### Error and operational attributes

| Attribute   | Type   | When | Description |
|------------|--------|------|-------------|
| `err`      | object | error/warn | Error object (message, stack, etc.) |
| `url`      | string | error | `req.originalUrl` |
| `query`    | object | error | `req.query` |
| `entityId` | number | Operations warn | Entity id that failed |
| `status`   | string | Operations warn | Status for failed update |
| `idsToRemove` | array | error | IDs that could not be removed |
| `jobId`    | string | Background jobs error/info | Background job id |
| `compressedCount` | number | info | Number of log files compressed on startup |

### Feature summary

- **Structured JSON**: Every log line is a JSON object; search and filter by any attribute.
- **Request correlation**: Use `requestHash` or `X-Request-Id` to tie frontend requests to service logs and query history.
- **Query debugging**: Use `trackerQueryCount`, `sqlPreview`, `paramCount`, `duration`, `rowCount` and the sequence of request messages to debug SQL and tracking.
- **Viewing logs**: Use the web log viewer, API, or command-line tools.

## Native JavaScript/Node.js

All logging features are implemented using **native Node.js modules**:
- **File rotation**: `pino-roll` (Node.js package)
- **Compression**: `zlib` (built into Node.js)
- **No Python required**: Everything runs in Node.js

## Performance

Pino is one of the fastest Node.js loggers:
- Asynchronous logging (doesn't block the event loop)
- JSON serialization is optimized
- File writes are buffered
- Minimal overhead on application performance

---

## Template-Based Spam Detection

### Why Template Hashing for Logs?

Log spam can be a DoS vector and makes debugging difficult. Template hashing helps:
- **Detect repeated patterns**: "User {ID} failed login" repeated 1000x
- **Prevent log storage exhaustion**: Throttle repeated log messages
- **Identify attack patterns**: Same error template with different values
- **Improve monitoring**: Alert on unusual log patterns

**See**: [Spam Detection Strategy](spam-detection-strategy.md) for complete guide on template hashing.

### Integration with Pino Logger

```typescript
// logger.js - Enhanced with template-based spam detection
import pino from 'pino';
import Redis from 'ioredis';
import { createHash } from 'crypto';

class TemplateHasher {
  extractTemplate(message: string): string {
    let template = message;
    
    // Replace common variable patterns
    template = template.replace(/\b\d{4,}\b/g, '{NUMBER}'); // IDs, order numbers
    template = template.replace(/\b[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}\b/gi, '{UUID}');
    template = template.replace(/\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b/g, '{EMAIL}');
    template = template.replace(/https?:\/\/[^\s]+/g, '{URL}');
    template = template.replace(/\d{4}-\d{2}-\d{2}[T ]\d{2}:\d{2}:\d{2}/g, '{TIMESTAMP}');
    template = template.replace(/\b\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}\b/g, '{IP}');
    
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

class LogSpamDetector {
  private redis: Redis;
  private templateHasher: TemplateHasher;
  
  constructor(redis: Redis) {
    this.redis = redis;
    this.templateHasher = new TemplateHasher();
  }
  
  /**
   * Check if log message should be throttled due to spam
   * @param level - Log level (error, warn, etc.)
   * @param message - Log message
   * @param maxOccurrences - Max occurrences before throttling (default: 10 per minute)
   * @param timeWindow - Time window in seconds (default: 60)
   * @returns { shouldLog: boolean, occurrences: number, templateHash: string }
   */
  async shouldLog(
    level: string,
    message: string,
    maxOccurrences: number = 10,
    timeWindow: number = 60
  ): Promise<{ shouldLog: boolean; occurrences: number; templateHash: string }> {
    // Only check error/warn levels (most spam-prone)
    if (level !== 'error' && level !== 'warn') {
      return { shouldLog: true, occurrences: 0, templateHash: '' };
    }
    
    const templateHash = this.templateHasher.getTemplateHash(message);
    const key = `log:spam:${level}:${templateHash}`;
    
    // Use Lua script for atomic check-and-increment
    const script = `
      local key = KEYS[1]
      local max = tonumber(ARGV[1])
      local ttl = tonumber(ARGV[2])
      
      local count = redis.call('INCR', key)
      
      if count == 1 then
        redis.call('EXPIRE', key, ttl)
      end
      
      if count > max then
        return {0, count}  -- Don't log (spam detected)
      else
        return {1, count}  -- Log normally
      end
    `;
    
    const result = await this.redis.eval(
      script,
      1,
      key,
      maxOccurrences,
      timeWindow
    ) as [number, number];
    
    return {
      shouldLog: result[0] === 1,
      occurrences: result[1],
      templateHash
    };
  }
  
  /**
   * Get spam statistics for a template hash
   */
  async getSpamStats(templateHash: string, level: string): Promise<number> {
    const key = `log:spam:${level}:${templateHash}`;
    const count = await this.redis.get(key);
    return parseInt(count || '0');
  }
}

// Create Pino logger with spam detection
function createLoggerWithSpamDetection(redis?: Redis) {
  const spamDetector = redis ? new LogSpamDetector(redis) : null;
  
  const logger = pino({
    level: process.env.LOG_LEVEL || 'info',
    // ... existing config
  });
  
  // Wrap logger methods to add spam detection
  if (spamDetector) {
    const originalError = logger.error.bind(logger);
    const originalWarn = logger.warn.bind(logger);
    
    logger.error = async function(obj: any, msg?: string, ...args: any[]) {
      const message = msg || obj?.msg || '';
      const check = await spamDetector.shouldLog('error', message, 10, 60);
      
      if (!check.shouldLog) {
        // Log once that spam was detected (to avoid infinite loop, use original logger)
        if (check.occurrences === 11) { // First time exceeding threshold
          originalError({
            ...obj,
            logSpamDetected: true,
            templateHash: check.templateHash,
            occurrences: check.occurrences,
            message: `Log spam detected: ${message.substring(0, 100)}...`
          }, 'Log spam detected - throttling repeated error messages');
        }
        return; // Don't log the spam message
      }
      
      return originalError(obj, msg, ...args);
    };
    
    logger.warn = async function(obj: any, msg?: string, ...args: any[]) {
      const message = msg || obj?.msg || '';
      const check = await spamDetector.shouldLog('warn', message, 10, 60);
      
      if (!check.shouldLog) {
        if (check.occurrences === 11) {
          originalWarn({
            ...obj,
            logSpamDetected: true,
            templateHash: check.templateHash,
            occurrences: check.occurrences,
            message: `Log spam detected: ${message.substring(0, 100)}...`
          }, 'Log spam detected - throttling repeated warning messages');
        }
        return;
      }
      
      return originalWarn(obj, msg, ...args);
    };
  }
  
  return logger;
}

// Usage
const redis = process.env.REDIS_URL ? new Redis(process.env.REDIS_URL) : undefined;
export const logger = createLoggerWithSpamDetection(redis);
```

### Pino Stream with Spam Detection

Alternative approach using Pino streams:

```typescript
import pino from 'pino';
import { Transform } from 'stream';

class SpamDetectionStream extends Transform {
  private spamDetector: LogSpamDetector;
  
  constructor(spamDetector: LogSpamDetector) {
    super({ objectMode: true });
    this.spamDetector = spamDetector;
  }
  
  async _transform(chunk: any, encoding: string, callback: Function) {
    const logEntry = JSON.parse(chunk.toString());
    const level = pino.levels.labels[logEntry.level];
    const message = logEntry.msg || '';
    
    // Check spam for error/warn levels
    if (level === 'error' || level === 'warn') {
      const check = await this.spamDetector.shouldLog(level, message);
      
      if (!check.shouldLog) {
        // Don't pass through spam logs
        // But log once that spam was detected
        if (check.occurrences === 11) {
          this.push(JSON.stringify({
            ...logEntry,
            msg: `Log spam detected - throttling: ${message.substring(0, 100)}...`,
            logSpamDetected: true,
            templateHash: check.templateHash,
            occurrences: check.occurrences
          }) + '\n');
        }
        return callback();
      }
    }
    
    // Pass through normal logs
    this.push(chunk);
    callback();
  }
}

// Create logger with spam detection stream
const redis = new Redis(process.env.REDIS_URL);
const spamDetector = new LogSpamDetector(redis);
const spamStream = new SpamDetectionStream(spamDetector);

const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
}, spamStream);
```

### Configuration

Add to `.env`:

```bash
# Enable log spam detection
LOG_SPAM_DETECTION_ENABLED=true
REDIS_URL=redis://localhost:6379

# Spam detection thresholds
LOG_SPAM_MAX_ERRORS=10      # Max error occurrences per minute
LOG_SPAM_MAX_WARNINGS=10    # Max warning occurrences per minute
LOG_SPAM_WINDOW=60          # Time window in seconds
```

### Monitoring Log Spam

```typescript
// Monitor log spam patterns
async function getLogSpamReport(level: string = 'error') {
  const redis = new Redis(process.env.REDIS_URL);
  const keys = await redis.keys(`log:spam:${level}:*`);
  
  const report = await Promise.all(
    keys.map(async (key) => {
      const count = await redis.get(key);
      const templateHash = key.split(':').pop();
      return {
        templateHash,
        occurrences: parseInt(count || '0'),
        level
      };
    })
  );
  
  // Sort by occurrences (most spam first)
  return report.sort((a, b) => b.occurrences - a.occurrences);
}

// Alert on high spam rates
async function checkLogSpamAlerts() {
  const errorSpam = await getLogSpamReport('error');
  const warningSpam = await getLogSpamReport('warn');
  
  // Alert if any template exceeds 100 occurrences
  const highSpam = [...errorSpam, ...warningSpam].filter(s => s.occurrences > 100);
  
  if (highSpam.length > 0) {
    await alertManager.sendAlert({
      severity: 'warning',
      message: 'High log spam detected',
      details: highSpam
    });
  }
}
```

### Benefits

1. **Prevents log storage exhaustion**: Throttles repeated messages
2. **Improves signal-to-noise**: Focus on unique errors, not spam
3. **Detects attack patterns**: Identifies repeated error templates
4. **Performance**: Redis lookups are < 1ms (vs database queries)
5. **Monitoring**: Track spam patterns for security analysis

### Best Practices

1. **Only check error/warn levels**: Don't throttle info/debug (less spam-prone)
2. **Whitelist legitimate repeated messages**: System health checks, etc.
3. **Monitor spam patterns**: Alert on unusual spikes
4. **Adjust thresholds**: Tune based on your application's normal patterns
5. **Log spam detection events**: Record when spam is detected for analysis

**See Also**: [Spam Detection Strategy](spam-detection-strategy.md) for complete template hashing guide.
