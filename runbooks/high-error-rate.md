# High Error Rate Runbook

## Symptoms
- Alert: "Error rate > 5% for 5 minutes"
- Users reporting errors or failed requests
- Grafana dashboard shows spike in `http_errors_total` metric

## Impact
- **Severity**: 🔴 Critical
- **Affected**: All users or specific endpoint users
- **Business Impact**: Service degradation, potential revenue loss

---

## Triage Steps (5 minutes)

### 1. Check Grafana Dashboard
```bash
# Open: https://grafana.myapp.com/d/app-overview
# Look at "Error Rate" panel
```

**Questions to answer**:
- What is the current error rate? (e.g., 8%)
- Which endpoint(s) are affected?
- What HTTP status codes? (4xx vs 5xx)
- When did it start?

### 2. Identify Affected Endpoint
```promql
# Grafana query to find worst endpoint
topk(5, rate(http_errors_total[5m])) by (route, status)
```

### 3. Check Recent Deployments
```bash
# Check if error started after deployment
git log --since="1 hour ago" --oneline
# Check deployment annotations in Grafana
```

### 4. Review Error Logs
```bash
# SSH to server or use log aggregation
tail -f logs/{app_name}-current.log | grep -i error

# Or filter by endpoint
tail -f logs/{app_name}-current.log | grep "POST /api/users" | grep -i error
```

---

## Common Causes & Solutions

### Cause 1: Recent Deployment Introduced Bug
**Symptoms**: Error rate spiked immediately after deployment

**Solution**:
```bash
# Rollback to previous version
git revert HEAD
git push origin main

# Or use deployment tool
./scripts/rollback.sh

# Verify error rate decreases
```

**Follow-up**: Fix bug, add tests, redeploy

---

### Cause 2: Database Connection Pool Exhausted
**Symptoms**: 
- Errors: "Connection pool timeout"
- Database pool metric shows 100% utilization

**Check**:
```promql
# Grafana query
database_pool_size{state="used"} / 
(database_pool_size{state="used"} + database_pool_size{state="free"})
```

**Solution**:
```bash
# Temporary: Increase pool size
# Edit .env or config
DATABASE_POOL_MAX=100  # Was 50

# Restart application
pm2 restart app

# Long-term: Optimize queries, add connection pooling
```

---

### Cause 3: External API Timeout/Failure
**Symptoms**:
- Errors: "Request timeout", "ECONNREFUSED"
- Specific to endpoints calling external APIs

**Check**:
```bash
# Check external API status
curl -I https://external-api.com/health

# Check logs for external API errors
grep "external-api" logs/{app_name}-current.log | grep -i error
```

**Solution**:
```typescript
// Add circuit breaker (if not already present)
import CircuitBreaker from 'opossum';

const breaker = new CircuitBreaker(callExternalAPI, {
  timeout: 5000,
  errorThresholdPercentage: 50,
  resetTimeout: 30000
});

// Fallback behavior
breaker.fallback(() => {
  return { status: 'degraded', data: cachedData };
});
```

---

### Cause 4: Invalid Input Validation
**Symptoms**:
- 400/422 errors
- Started after frontend deployment

**Check**:
```bash
# Look for validation errors in logs
grep "validation" logs/{app_name}-current.log | tail -20
```

**Solution**:
```bash
# If frontend sent bad data, coordinate with frontend team
# If backend validation too strict, adjust validation rules
# Add better error messages to help frontend debug
```

---

### Cause 5: Rate Limiting Triggered
**Symptoms**:
- 429 errors
- Specific user or IP

**Check**:
```bash
# Check rate limit logs
grep "rate limit" logs/{app_name}-current.log
```

**Solution**:
```bash
# If legitimate traffic, increase rate limits
# If attack, block IP at firewall level
# Add CAPTCHA for suspicious traffic
```

---

## Resolution Verification

After applying fix, verify:

### 1. Check Error Rate
```promql
# Should drop below 5%
rate(http_errors_total[5m]) / rate(http_requests_total[5m]) * 100
```

### 2. Check Logs
```bash
# Should see fewer errors
tail -f logs/{app_name}-current.log | grep -i error
```

### 3. Test Affected Endpoint
```bash
# Manual test
curl -X POST https://api.myapp.com/users \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","name":"Test"}'

# Should return 200/201, not 500
```

### 4. Monitor for 15 Minutes
- Watch Grafana dashboard
- Ensure error rate stays below threshold
- Check for any new errors

---

## Prevention

### Short-Term
- [ ] Add monitoring for this specific error type
- [ ] Add integration tests covering this scenario
- [ ] Document the fix in post-mortem

### Long-Term
- [ ] Implement circuit breakers for external APIs
- [ ] Add request validation tests
- [ ] Set up canary deployments
- [ ] Add database query performance monitoring
- [ ] Implement graceful degradation

---

## Escalation

### When to Escalate
- Error rate > 20% for > 10 minutes
- Unable to identify cause within 15 minutes
- Rollback doesn't fix the issue
- Database or infrastructure issue suspected

### Who to Contact
1. **Tech Lead**: @tech-lead (Slack)
2. **Database Admin**: @dba (if database-related)
3. **DevOps**: @devops (if infrastructure-related)
4. **On-Call Manager**: +1-555-0100

---

## Post-Incident

### 1. Create Post-Mortem
```markdown
# Incident: High Error Rate on 2026-01-28

## Timeline
- 14:23: Alert fired
- 14:25: Investigation started
- 14:30: Identified cause (database pool)
- 14:35: Applied fix (increased pool size)
- 14:40: Verified resolution

## Root Cause
Database connection pool exhausted due to slow queries

## Impact
- Duration: 17 minutes
- Error rate: 8%
- Affected users: ~500 requests failed

## Resolution
Increased database pool size, optimized slow queries

## Prevention
- Add database query performance monitoring
- Set up alerts for pool utilization > 80%
- Optimize queries before they become critical
```

### 2. Update Runbook
- Add any new causes discovered
- Update solutions that worked
- Add prevention measures

### 3. Create Follow-Up Tasks
- [ ] Optimize slow database queries
- [ ] Add query performance monitoring
- [ ] Increase test coverage
- [ ] Review similar endpoints for same issue

---

## Related Runbooks
- [High Latency](high-latency.md)
- [Database Errors](database-errors.md)
- [Deployment Failed](deployment-failed.md)

## See Also
- [Prometheus Monitoring](../prometheus.md)
- [Logging Guide](../logging.md)
- [Error Handling](../ERROR_HANDLING.md)
