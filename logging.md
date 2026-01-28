# Logging Configuration

---
**Status**: ✅ **IMPLEMENTED** - Pino-based structured logging with file rotation  
**Priority**: 🔴 **P0.7** - Critical for debugging production issues  
**Last Updated**: 2026-01-28  
**Owned By**: Backend Team  

---

The backend uses [Pino](https://getpino.io/) for structured logging with file rotation and compression.

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
| `baseLogFile`     | string | Logger init | Full path to `backend-YYYYMMDD_HHMMSS.log` |
| `rotationPolicy`  | string | Logger init | `"size-based only (20MB limit, no daily rotation)"` |
| `logLevel`        | string | Logger init | Effective level (e.g. `debug`, `info`) |
| `isDevelopment`   | bool   | Logger init | From `NODE_ENV !== 'production'` |
| `nodeEnv`         | string | Logger init | `process.env.NODE_ENV \|\| 'development'` |
| `compressedCount` | number | Post-startup | Count of log files compressed (when &gt; 0) |

### Request / query-tracking attributes

| Attribute           | Type    | When | Description |
|--------------------|---------|------|-------------|
| `requestHash`      | string  | getRecords, getSummary, history | Hash used to correlate frontend request with backend; from `X-Request-Id` or generated |
| `xRequestId`       | string? | getRecords debug | `req.headers['x-request-id']` |
| `XRequestId`       | string? | getRecords debug | `req.headers['X-Request-Id']` |
| `allHeaders`       | string[]| getRecords debug | Header names containing `"request"` |
| `trackerExists`    | boolean | getRecords debug | Whether `getQueryTracker(req)` returned a tracker |
| `trackerQueryCount`| number  | getRecords debug | Number of queries tracked so far for this request |
| `reqHasTracker`    | boolean | getRecords debug | Whether `req._queryTracker` exists |
| `hasReq`           | boolean | getRecords debug | Request object passed to query |
| `countSqlPreview`  | string  | getRecords debug | First ~100 chars of count SQL |
| `countResult`      | string  | getRecords debug | `'array'` or `'not array'` |
| `sqlPreview`       | string  | DB query tracking | First portion of executed SQL |
| `paramCount`       | number  | DB query tracking, buildQuery | Number of bound parameters |
| `duration`         | number  | DB query tracking | Query duration (ms) |
| `rowCount`         | number  | DB query tracking | Rows returned |
| `params`           | any[]   | buildQuery debug | Query parameters (may be truncated) |
| `placeholders`     | number  | getSummary debug | Number of `?` in SQL |
| `endpoint`         | string  | getRecords history | e.g. `/api/reconciliation/records?...` |
| `queries`          | array   | getRecords history | Tracked query objects (sql, params, duration, etc.) |
| `queryCount`       | number  | getRecords | Number of tracked queries for this request |
| `rawQueriesCount`  | number  | getRecords debug | Length of `tracker.getQueries()` |
| `rawQueries`       | array   | getRecords debug | Serialized tracked queries |

### Error and operational attributes

| Attribute   | Type   | When | Description |
|------------|--------|------|-------------|
| `err`      | object | error/warn | Error object (message, stack, etc.) |
| `url`      | string | error | `req.originalUrl` |
| `query`    | object | error | `req.query` |
| `recordId` | number | bulk-review warn | Record id that failed |
| `review_status` | string | bulk-review warn | Status for failed update |
| `crosswalkIdsToRemove` | array | error | IDs that could not be removed |
| `jobId`    | string | crosswalk error/info | Approve-to-crosswalk job id |
| `compressedCount` | number | info | Number of log files compressed on startup |

### Feature summary

- **Structured JSON**: Every log line is a JSON object; search and filter by any attribute.
- **Request correlation**: Use `requestHash` or `X-Request-Id` to tie frontend requests to backend logs and query history.
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
