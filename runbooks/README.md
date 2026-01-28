# Operational Runbooks

## Purpose

Runbooks are step-by-step guides for responding to common incidents and operational tasks. They help on-call engineers quickly diagnose and resolve issues.

---

## Available Runbooks

### Critical Incidents
- **[High Error Rate](high-error-rate.md)** - Error rate > 5%
- **[High Latency](high-latency.md)** - P95 latency > 1s
- **[Application Down](application-down.md)** - Service not responding
- **[Database Errors](database-errors.md)** - Database connection issues

### Performance Issues
- **[High Resource Usage](high-resource-usage.md)** - CPU/Memory > 80%
- **[Slow Database Queries](slow-queries.md)** - Database performance degradation

### Deployment Issues
- **[Deployment Failed](deployment-failed.md)** - CI/CD deployment failure
- **[Rollback Procedure](rollback.md)** - How to rollback a deployment

---

## Runbook Template

When creating a new runbook, use this structure:

```markdown
# [Incident Name]

## Symptoms
- What the user/system experiences
- Alert that fired

## Impact
- Who is affected
- Severity level

## Triage Steps
1. Check [specific metric/dashboard]
2. Review [specific logs]
3. Verify [specific condition]

## Common Causes
- Cause 1 → Solution
- Cause 2 → Solution
- Cause 3 → Solution

## Resolution Steps
1. Step-by-step instructions
2. Commands to run
3. What to verify

## Prevention
- How to prevent this in the future
- Monitoring to add
- Code changes needed

## Escalation
- When to escalate
- Who to contact
```

---

## Quick Reference

| Alert | Severity | Response Time | Runbook |
|-------|----------|---------------|---------|
| Application Down | 🔴 Critical | Immediate | [application-down.md](application-down.md) |
| High Error Rate | 🔴 Critical | 5 minutes | [high-error-rate.md](high-error-rate.md) |
| Database Errors | 🔴 Critical | 5 minutes | [database-errors.md](database-errors.md) |
| High Latency | 🟡 Warning | 15 minutes | [high-latency.md](high-latency.md) |
| High CPU/Memory | 🟡 Warning | 15 minutes | [high-resource-usage.md](high-resource-usage.md) |
| Deployment Failed | 🟡 Warning | 30 minutes | [deployment-failed.md](deployment-failed.md) |

---

## On-Call Workflow

1. **Alert fires** → Check alert details
2. **Find runbook** → Use this index
3. **Follow triage steps** → Diagnose issue
4. **Apply resolution** → Fix the problem
5. **Verify fix** → Check metrics/logs
6. **Document** → Update runbook if needed
7. **Post-mortem** → For critical incidents

---

## Contributing

To add a new runbook:

1. Copy the template above
2. Fill in all sections
3. Add link to this index
4. Test the runbook during next incident
5. Update based on lessons learned

---

## See Also

- [DevOps Overview](../devops.md)
- [Prometheus Monitoring](../prometheus.md)
- [Logging Guide](../LOGGING.md)
- [Implementation Status](../IMPLEMENTATION_STATUS.md)
