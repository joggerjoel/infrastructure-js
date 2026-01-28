# Architecture Practices & Operational Excellence

---
**Status**: 💡 **GUIDANCE** - Best practices for long-lived systems  
**Priority**: 🟡 **P1** - Essential for maintainability and reliability  
**Last Updated**: 2026-01-28  
**Owned By**: Architecture Team, SRE Team  
**Sections**: 23 comprehensive practice areas covering architecture, operations, and engineering excellence

---

## Table of Contents

1. [Architecture Decision Records (ADR)](#architecture-decision-records-adr)
2. [System Boundaries & Ownership](#system-boundaries--ownership)
3. [Dependency Management](#dependency-management)
4. [Reliability Engineering (SRE)](#reliability-engineering-sre)
5. [Data Discipline](#data-discipline)
6. [Development Standards](#development-standards)
7. [Security Practices](#security-practices)
8. [Operational Excellence](#operational-excellence)
9. [Documentation Meta-Standards](#documentation-meta-standards)
10. [AI-Specific Practices](#ai-specific-practices)
11. [Performance Engineering](#performance-engineering)
12. [Observability Standards](#observability-standards)
13. [API Design & Versioning](#api-design--versioning)
14. [Testing Strategy](#testing-strategy)
15. [Configuration Management](#configuration-management)
16. [Service Mesh / Networking Standards](#service-mesh--networking-standards)
17. [Cost Engineering](#cost-engineering)
18. [Compliance & Audit](#compliance--audit)
19. [Disaster Recovery & Business Continuity](#disaster-recovery--business-continuity)
20. [Team Practices & Knowledge Management](#team-practices--knowledge-management)
21. [Mobile-Specific Practices](#mobile-specific-practices)
22. [Environmental / Sustainability](#environmental--sustainability)
23. [Standards Evolution & Compliance](#standards-evolution--compliance)

---

## Architecture Decision Records (ADR)

### Why ADRs Matter

**Prevents tribal knowledge and "why the hell did we do this?" six months later.**

ADRs capture the **why** behind architectural decisions, not just the **what**. They document:
- What decision was made
- What alternatives were considered
- What tradeoffs were evaluated
- Who made the decision and when

### ADR Template

```markdown
# ADR-001: [Short Title]

**Status**: Proposed | Accepted | Deprecated | Superseded  
**Date**: YYYY-MM-DD  
**Deciders**: [Names/Teams]  
**Tags**: [architecture, database, api, etc.]

## Context

What is the issue we're trying to address? What constraints exist?

## Decision

What is the change we're proposing or have agreed to?

## Alternatives Considered

### Alternative 1: [Name]
- **Pros**: ...
- **Cons**: ...
- **Why not chosen**: ...

### Alternative 2: [Name]
- **Pros**: ...
- **Cons**: ...
- **Why not chosen**: ...

## Tradeoffs

- **Pros**: What are the benefits?
- **Cons**: What are the drawbacks?
- **Risks**: What could go wrong?
- **Mitigations**: How do we address risks?

## Consequences

- **Positive**: What good things will happen?
- **Negative**: What bad things might happen?
- **Neutral**: What are the neutral consequences?

## Implementation Notes

- Key implementation details
- Migration path (if applicable)
- Rollback plan

## References

- Related ADRs
- External documentation
- RFCs or standards
```

### ADR Directory Structure

```
docs/
  adr/
    0001-use-postgresql-over-mongodb.md
    0002-implement-event-sourcing.md
    0003-adopt-openapi-for-api-docs.md
    0004-use-redis-for-caching.md
    0005-deprecate-adr-0002.md  # Supersedes ADR-0002
```

### Example ADR

```markdown
# ADR-001: Use PostgreSQL Over MongoDB

**Status**: Accepted  
**Date**: 2026-01-15  
**Deciders**: Backend Team, Data Team  
**Tags**: database, architecture

## Context

We need to choose a primary database for the application. Requirements:
- ACID transactions
- Complex relational queries
- JSON support for flexible schemas
- Strong consistency guarantees

## Decision

Use PostgreSQL as the primary database.

## Alternatives Considered

### Alternative 1: MongoDB
- **Pros**: 
  - Flexible schema
  - Horizontal scaling
  - Document model matches some use cases
- **Cons**: 
  - Weaker consistency guarantees
  - No native joins
  - Complex transactions (recent addition)
- **Why not chosen**: ACID requirements and relational queries are critical

### Alternative 2: MySQL
- **Pros**: 
  - Mature, widely used
  - Good performance
- **Cons**: 
  - Limited JSON support
  - Less flexible than PostgreSQL
- **Why not chosen**: PostgreSQL's JSON support and extensibility are valuable

## Tradeoffs

- **Pros**: 
  - ACID compliance
  - Rich query capabilities
  - JSON support for flexibility
  - Strong ecosystem
- **Cons**: 
  - Vertical scaling limitations
  - More complex than NoSQL for simple use cases
- **Risks**: 
  - Scaling challenges at very high scale
- **Mitigations**: 
  - Use read replicas
  - Consider partitioning strategies
  - Monitor performance metrics

## Consequences

- **Positive**: 
  - Strong data integrity
  - Complex queries possible
  - Team familiar with SQL
- **Negative**: 
  - May need optimization for high-scale scenarios
- **Neutral**: 
  - Migration from existing system required

## Implementation Notes

- Use connection pooling (pgBouncer)
- Set up read replicas for scaling
- Use JSONB columns for flexible schemas
- Implement proper indexing strategy
```

### ADR Workflow

1. **Propose**: Create ADR with status "Proposed"
2. **Review**: Team reviews, discusses alternatives
3. **Decide**: Update status to "Accepted" or "Rejected"
4. **Implement**: Reference ADR in implementation PRs
5. **Deprecate**: If superseded, mark as "Deprecated" and link to new ADR

---

## System Boundaries & Ownership

### Why Ownership Matters

**Every incident question starts with: "Who owns this?"**

Clear ownership prevents:
- Blame games
- Unclear escalation paths
- Knowledge gaps
- Unmaintained systems

### Service Ownership Model

```yaml
services:
  api-server:
    owner: backend-team
    primary_contact: "@backend-lead"
    escalation_path:
      - "@backend-lead"
      - "@engineering-manager"
      - "@cto"
    on_call: true
    runbooks: ["docs/runbooks/api-server/"]
    
  scraper-service:
    owner: data-team
    primary_contact: "@data-lead"
    escalation_path:
      - "@data-lead"
      - "@backend-lead"
    on_call: true
    runbooks: ["docs/runbooks/scraper/"]
    
  reconciliation-service:
    owner: data-team
    primary_contact: "@data-lead"
    escalation_path:
      - "@data-lead"
      - "@backend-lead"
    on_call: false  # Batch job, not real-time
```

### Data Ownership

```yaml
data_domains:
  events:
    owner: data-team
    schema: stubhub_events
    retention: 2_years
    pii: false
    
  users:
    owner: backend-team
    schema: users
    retention: indefinite
    pii: true
    gdpr_applies: true
    
  reconciliation:
    owner: data-team
    schema: event_reconciliation_review
    retention: 1_year
    pii: false
```

### Domain Boundaries (DDD-lite)

```typescript
// Domain: Event Management
// Owner: Data Team
// Bounded Context: Event scraping, storage, reconciliation

namespace EventDomain {
  // Core entities
  export interface Event { }
  export interface Venue { }
  
  // Services
  export class EventScraper { }
  export class EventReconciler { }
  
  // Repositories
  export interface EventRepository { }
}

// Domain: User Management
// Owner: Backend Team
// Bounded Context: Authentication, authorization, user data

namespace UserDomain {
  export interface User { }
  export class AuthService { }
  export interface UserRepository { }
}
```

### Ownership Documentation Template

```markdown
# Service: [Service Name]

## Ownership

- **Team**: [Team Name]
- **Primary Contact**: [@slack-handle]
- **On-Call Rotation**: [Link to PagerDuty/Opsgenie]

## Escalation Path

1. **L1**: [@primary-contact] - First responder
2. **L2**: [@team-lead] - Escalation after 15 minutes
3. **L3**: [@engineering-manager] - Escalation after 1 hour
4. **L4**: [@cto] - Critical incidents only

## Responsibilities

- [ ] Monitoring and alerting
- [ ] Incident response
- [ ] Capacity planning
- [ ] Security updates
- [ ] Documentation maintenance

## Dependencies

- **Depends On**: [List services]
- **Depended On By**: [List services]
- **External Dependencies**: [APIs, databases, etc.]

## Runbooks

- [Runbook 1](./runbooks/service/incident-1.md)
- [Runbook 2](./runbooks/service/incident-2.md)
```

---

## Dependency Management

### Dependency Update Strategy

```yaml
# .github/dependabot.yml
version: 2
updates:
  # Production dependencies - security only
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 5
    versioning-strategy: increase
    groups:
      production-dependencies:
        dependency-type: "production"
        update-types:
          - "security"
    
  # Development dependencies - all updates
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
    groups:
      dev-dependencies:
        dependency-type: "development"
        update-types:
          - "version-update:semver-major"
          - "version-update:semver-minor"
          - "version-update:semver-patch"
```

### Breaking-Change Policy

```markdown
## Dependency Breaking Changes

### Process

1. **Assessment**: Evaluate impact of breaking change
2. **Communication**: Notify team via Slack #engineering channel
3. **Planning**: Create migration plan
4. **Testing**: Test in staging environment
5. **Deployment**: Deploy during low-traffic window

### Major Version Upgrades

- **Node.js**: Requires team review, 2-week notice
- **Framework updates** (Express, etc.): Requires ADR
- **Database drivers**: Requires migration testing
- **Security patches**: Expedited, can skip some steps

### Rollback Plan

- Keep previous version in package-lock.json
- Document rollback steps in runbook
- Test rollback procedure quarterly
```

### Vulnerability Scanning Cadence

```yaml
# .github/workflows/security-scan.yml
name: Security Scan

on:
  schedule:
    - cron: '0 0 * * 1'  # Weekly on Monday
  push:
    branches: [main]
  pull_request:

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Run npm audit
        run: npm audit --audit-level=moderate
      
      - name: Run Snyk scan
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
      
      - name: Check for known vulnerabilities
        run: |
          npm audit --json > audit.json
          HIGH=$(jq '.metadata.vulnerabilities.high' audit.json)
          CRITICAL=$(jq '.metadata.vulnerabilities.critical' audit.json)
          if [ "$HIGH" -gt 0 ] || [ "$CRITICAL" -gt 0 ]; then
            echo "❌ Found vulnerabilities"
            exit 1
          fi
```

### Lockfile Discipline

```markdown
## Lockfile Policy

### Rules

1. **Always commit** `package-lock.json` (or `yarn.lock`)
2. **Never** edit lockfile manually
3. **Regenerate** lockfile when updating dependencies
4. **Review** lockfile changes in PRs

### CI/CD Checks

```yaml
# .github/workflows/check-lockfile.yml
- name: Verify lockfile is up to date
  run: |
    npm install --package-lock-only
    git diff --exit-code package-lock.json || \
      (echo "❌ package-lock.json is out of sync" && exit 1)
```

### Dependency Audit

- **Weekly**: Automated security scans
- **Monthly**: Review dependency tree for unused packages
- **Quarterly**: Major version upgrade planning
```

---

## Reliability Engineering (SRE)

### SLIs / SLOs / Error Budgets

**This is the difference between observed and managed systems.**

#### Service Level Indicators (SLIs)

```yaml
sli_definitions:
  availability:
    description: "Percentage of successful requests"
    measurement: "successful_requests / total_requests"
    window: "1 minute"
    
  latency:
    description: "Request latency at p99"
    measurement: "histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))"
    window: "5 minutes"
    
  freshness:
    description: "Data freshness for scraper"
    measurement: "time_since_last_successful_scrape"
    window: "1 hour"
    
  correctness:
    description: "Data validation pass rate"
    measurement: "validated_records / total_records"
    window: "1 hour"
```

#### Service Level Objectives (SLOs)

```yaml
slo_definitions:
  api_availability:
    sli: availability
    target: 99.9%  # 99.9% uptime = ~43 minutes downtime/month
    window: "30 days"
    error_budget: 0.1%  # 43 minutes/month
    
  api_latency:
    sli: latency
    target: "p99 < 500ms"
    window: "30 days"
    error_budget: 0.1%
    
  scraper_freshness:
    sli: freshness
    target: "max_age < 24 hours"
    window: "7 days"
    error_budget: 1%  # More lenient for batch jobs
    
  data_correctness:
    sli: correctness
    target: "> 99.5%"
    window: "30 days"
    error_budget: 0.5%
```

#### Error Budget Policy

```markdown
## Error Budget Management

### What Happens When We Burn Budget?

#### 0-50% Budget Remaining (Warning)
- **Action**: Review recent incidents
- **Communication**: Post-mortem scheduled
- **Prevention**: Increase monitoring, review runbooks

#### 50-100% Budget Remaining (Critical)
- **Action**: Freeze feature releases
- **Communication**: All-hands announcement
- **Focus**: Stability work only
- **Review**: Weekly SLO review meetings

#### Budget Exhausted (Emergency)
- **Action**: Stop all non-critical work
- **Communication**: Executive escalation
- **Focus**: Reliability work exclusively
- **Review**: Daily SLO review until recovered

### Budget Recovery

- **Target**: 75% budget remaining before resuming features
- **Process**: 
  1. Identify root causes
  2. Implement fixes
  3. Monitor for 7 days
  4. Resume feature work gradually
```

#### SLO Monitoring Dashboard

```typescript
// Prometheus queries for SLO dashboards

// Availability SLO
const availabilitySLO = `
  sum(rate(http_requests_total{status=~"2.."}[5m])) 
  / 
  sum(rate(http_requests_total[5m]))
`;

// Latency SLO (p99)
const latencySLO = `
  histogram_quantile(0.99, 
    sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
  )
`;

// Error Budget Remaining
const errorBudgetRemaining = `
  (1 - (1 - ${availabilitySLO}) / 0.001) * 100
`;
```

### Capacity Planning

#### Traffic Modeling

```typescript
// Capacity planning model

interface CapacityModel {
  // Current metrics
  currentQPS: number;
  currentLatency: number;
  currentUtilization: number;
  
  // Projections
  projectedQPS: {
    nextMonth: number;
    nextQuarter: number;
    nextYear: number;
  };
  
  // Headroom targets
  headroomTarget: number;  // e.g., 50% = 2x current capacity
  
  // Scaling thresholds
  scaleUpThreshold: number;   // e.g., 70% utilization
  scaleDownThreshold: number;  // e.g., 30% utilization
}

// Example calculation
const capacityPlan: CapacityModel = {
  currentQPS: 100,
  currentLatency: 200,  // ms
  currentUtilization: 60,  // %
  
  projectedQPS: {
    nextMonth: 120,    // +20% growth
    nextQuarter: 150,  // +50% growth
    nextYear: 250      // +150% growth
  },
  
  headroomTarget: 50,  // 50% headroom = 2x capacity
  
  scaleUpThreshold: 70,
  scaleDownThreshold: 30
};

// Calculate required capacity
const requiredCapacity = (qps: number, headroom: number) => 
  qps * (1 + headroom / 100);

const nextMonthCapacity = requiredCapacity(
  capacityPlan.projectedQPS.nextMonth,
  capacityPlan.headroomTarget
);  // 120 * 1.5 = 180 QPS capacity needed
```

#### Headroom Targets

```yaml
headroom_targets:
  api_servers:
    target: 50%  # 2x current capacity available
    measurement: "cpu_utilization < 50%"
    alert_threshold: 70%
    
  database:
    target: 30%  # More conservative for DB
    measurement: "connection_pool_utilization < 70%"
    alert_threshold: 80%
    
  cache:
    target: 40%
    measurement: "memory_utilization < 60%"
    alert_threshold: 75%
```

#### Peak vs Sustained Load

```markdown
## Load Characteristics

### Sustained Load
- **Definition**: Average load over 24 hours
- **Planning**: Use for capacity planning
- **Example**: 100 QPS average

### Peak Load
- **Definition**: Maximum load during peak hours
- **Planning**: Use for scaling thresholds
- **Example**: 300 QPS peak (3x sustained)

### Planning Strategy

1. **Base capacity**: Handle sustained load with 50% headroom
2. **Auto-scaling**: Scale to handle 2x peak load
3. **Manual scaling**: Pre-scale for known events (e.g., Black Friday)
```

#### Cost vs Performance Tradeoffs

```markdown
## Cost-Performance Analysis

### Scenario: Database Read Replicas

**Option 1**: Single database
- **Cost**: $X/month
- **Performance**: 1000 QPS max
- **Risk**: Single point of failure

**Option 2**: Primary + 2 Read Replicas
- **Cost**: $3X/month
- **Performance**: 3000 QPS max
- **Risk**: Lower (redundancy)

**Decision Matrix**:
- If QPS < 800: Option 1 (cost-effective)
- If QPS > 800: Option 2 (performance-critical)
- If budget-constrained: Option 1 + caching layer
```

### Chaos / Fault Injection (Lightweight)

**If you never break it intentionally, production will.**

#### Chaos Engineering Principles

```markdown
## Chaos Engineering

### Goals

1. **Build confidence** in system resilience
2. **Find weaknesses** before production incidents
3. **Validate** runbooks and incident response
4. **Test** monitoring and alerting

### Chaos Experiments

#### Level 1: Safe Experiments (Start Here)

1. **Kill a pod**
   - Impact: Low
   - Expected: Auto-recovery
   - Validation: Service continues operating

2. **Restart a service**
   - Impact: Low
   - Expected: Graceful shutdown, restart
   - Validation: No data loss

#### Level 2: Moderate Experiments

3. **Kill DB connections**
   - Impact: Medium
   - Expected: Connection pool recovery
   - Validation: Requests retry successfully

4. **Simulate Redis outage**
   - Impact: Medium
   - Expected: Fallback to database
   - Validation: Service degrades gracefully

#### Level 3: Advanced Experiments

5. **Slow downstream APIs**
   - Impact: Medium-High
   - Expected: Timeout handling, circuit breakers
   - Validation: No cascading failures

6. **Network partition**
   - Impact: High
   - Expected: Service isolation
   - Validation: Each partition operates independently
```

#### Chaos Testing Script

```typescript
// scripts/chaos-test.ts

interface ChaosExperiment {
  name: string;
  description: string;
  execute: () => Promise<void>;
  rollback: () => Promise<void>;
  expectedBehavior: string;
}

const experiments: ChaosExperiment[] = [
  {
    name: "kill-pod",
    description: "Kill a random API pod",
    execute: async () => {
      const pod = await getRandomPod("api-server");
      await kubectl.deletePod(pod);
    },
    rollback: async () => {
      // Auto-recovery via deployment
    },
    expectedBehavior: "Pod restarts automatically, service continues"
  },
  
  {
    name: "kill-db-connections",
    description: "Terminate database connections",
    execute: async () => {
      await db.terminateConnections(0.5); // Kill 50% of connections
    },
    rollback: async () => {
      // Connections will reconnect automatically
    },
    expectedBehavior: "Connection pool recovers, requests retry"
  },
  
  {
    name: "simulate-redis-outage",
    description: "Block Redis access",
    execute: async () => {
      await kubectl.networkPolicy.block("redis-service");
    },
    rollback: async () => {
      await kubectl.networkPolicy.allow("redis-service");
    },
    expectedBehavior: "Fallback to database, performance degrades gracefully"
  }
];

// Run experiment
async function runChaosExperiment(experiment: ChaosExperiment) {
  console.log(`🧪 Running: ${experiment.name}`);
  console.log(`Expected: ${experiment.expectedBehavior}`);
  
  // Monitor before
  const metricsBefore = await getMetrics();
  
  // Execute
  await experiment.execute();
  
  // Wait and observe
  await sleep(30000); // 30 seconds
  
  // Check metrics
  const metricsAfter = await getMetrics();
  const impact = analyzeImpact(metricsBefore, metricsAfter);
  
  // Rollback
  await experiment.rollback();
  
  // Report
  console.log(`Impact: ${impact}`);
  return impact;
}
```

#### Chaos Testing Schedule

```yaml
chaos_schedule:
  weekly:
    - experiment: "kill-pod"
      day: "Monday"
      time: "02:00"  # Low traffic
      
  monthly:
    - experiment: "kill-db-connections"
      day: "First Saturday"
      time: "03:00"
      
  quarterly:
    - experiment: "network-partition"
      day: "Quarter end Saturday"
      time: "04:00"
      
  game_days:
    - experiment: "full-chaos"
      frequency: "Quarterly"
      participants: "On-call team"
      purpose: "Validate incident response"
```

---

## Data Discipline

### Data Contracts / Schemas

**Especially important with LLMs + APIs.**

#### Schema Versioning

```typescript
// Data contract with versioning

interface EventSchema {
  version: "1.0.0" | "1.1.0" | "2.0.0";
  event: {
    id: string;
    name: string;
    venue: string;
    date: string;  // ISO 8601
    // v1.1.0 additions
    type?: "event" | "parking";  // Added in v1.1.0
    // v2.0.0 changes
    eventDate?: string;  // Renamed from 'date' in v2.0.0
  };
}

// Schema registry
class SchemaRegistry {
  private schemas: Map<string, any> = new Map();
  
  register(name: string, version: string, schema: any) {
    const key = `${name}:${version}`;
    this.schemas.set(key, schema);
  }
  
  validate(data: any, name: string, version: string): boolean {
    const key = `${name}:${version}`;
    const schema = this.schemas.get(key);
    return validateAgainstSchema(data, schema);
  }
  
  getLatestVersion(name: string): string {
    // Return latest version for name
  }
}
```

#### Backward Compatibility Rules

```markdown
## Backward Compatibility Policy

### Rules

1. **Additive changes only** in minor versions
   - ✅ Adding optional fields
   - ✅ Adding new endpoints
   - ❌ Removing fields
   - ❌ Changing field types

2. **Breaking changes** require major version
   - Field removals
   - Type changes
   - Required field additions

3. **Deprecation period**: 6 months minimum
   - Mark as deprecated
   - Document migration path
   - Remove in next major version

### Example

```typescript
// v1.0.0
interface Event {
  id: string;
  name: string;
  date: string;
}

// v1.1.0 - Backward compatible (additive)
interface Event {
  id: string;
  name: string;
  date: string;
  type?: "event" | "parking";  // Optional, backward compatible
}

// v2.0.0 - Breaking change
interface Event {
  id: string;
  name: string;
  eventDate: string;  // Renamed from 'date' - BREAKING
  type: "event" | "parking";  // Now required - BREAKING
}
```
```

#### Consumer-Driven Contracts

```typescript
// Contract testing with Pact

import { Pact } from '@pact-foundation/pact';

describe('Event API Contract', () => {
  const provider = new Pact({
    consumer: 'frontend-app',
    provider: 'api-server',
    port: 1234,
    log: './logs/pact.log',
    dir: './pacts',
    spec: 2
  });

  beforeAll(() => provider.setup());
  afterAll(() => provider.finalize());

  describe('GET /events/:id', () => {
    it('returns event details', () => {
      return provider.addInteraction({
        state: 'event exists',
        uponReceiving: 'a request for event details',
        withRequest: {
          method: 'GET',
          path: '/events/123'
        },
        willRespondWith: {
          status: 200,
          headers: { 'Content-Type': 'application/json' },
          body: {
            id: '123',
            name: 'Concert',
            venue: 'Venue Name',
            date: '2026-02-01T20:00:00Z'
          }
        }
      });
    });
  });
});
```

### Data Retention & Deletion

#### Retention Periods

```yaml
retention_policies:
  events:
    table: stubhub_events
    retention: 2_years
    archive_after: 1_year  # Move to cold storage
    delete_after: 2_years
    
  reconciliation_reviews:
    table: event_reconciliation_review
    retention: 1_year
    archive_after: 6_months
    delete_after: 1_year
    
  logs:
    table: application_logs
    retention: 90_days
    archive_after: 30_days
    delete_after: 90_days
    
  audit_logs:
    table: audit_logs
    retention: 7_years  # Compliance requirement
    archive_after: 1_year
    delete_after: 7_years
```

#### GDPR / Right-to-Forget

```typescript
// GDPR compliance implementation

class GDPRService {
  async deleteUserData(userId: string): Promise<void> {
    // 1. Identify all user data
    const userData = await this.findUserData(userId);
    
    // 2. Anonymize or delete
    await Promise.all([
      this.anonymizeUser(userId),           // Users table
      this.deleteUserEvents(userId),        // User-specific events
      this.deleteUserPreferences(userId),   // Preferences
      this.deleteAuditTrail(userId)         // Audit logs (with exceptions)
    ]);
    
    // 3. Log deletion
    await this.logDeletion(userId, {
      timestamp: new Date(),
      reason: 'GDPR request',
      deletedTables: userData.tables
    });
  }
  
  async exportUserData(userId: string): Promise<UserDataExport> {
    // Export all user data in machine-readable format
    return {
      user: await this.getUser(userId),
      events: await this.getUserEvents(userId),
      preferences: await this.getUserPreferences(userId),
      // ... other data
    };
  }
}
```

#### Archival Strategy

```typescript
// Data archival process

class ArchivalService {
  async archiveOldData(table: string, retentionDays: number): Promise<void> {
    const cutoffDate = new Date();
    cutoffDate.setDate(cutoffDate.getDate() - retentionDays);
    
    // 1. Query data to archive
    const dataToArchive = await db.query(`
      SELECT * FROM ${table}
      WHERE created_at < ?
    `, [cutoffDate]);
    
    // 2. Upload to cold storage (S3 Glacier, etc.)
    await this.uploadToColdStorage(table, dataToArchive);
    
    // 3. Delete from primary database
    await db.query(`
      DELETE FROM ${table}
      WHERE created_at < ?
    `, [cutoffDate]);
    
    // 4. Update metadata
    await this.updateArchivalMetadata(table, {
      archivedAt: new Date(),
      recordCount: dataToArchive.length,
      storageLocation: `s3://archive/${table}/${cutoffDate.toISOString()}.json`
    });
  }
  
  async restoreFromArchive(table: string, dateRange: DateRange): Promise<any[]> {
    // Restore data from cold storage if needed
    const archiveLocation = await this.findArchiveLocation(table, dateRange);
    return await this.downloadFromColdStorage(archiveLocation);
  }
}
```

### Idempotency & Exactly-Once Semantics

**Critical for: Payments, Reprocessing, Retries, Event systems**

#### Idempotency Keys

```typescript
// Idempotent API design

interface IdempotentRequest {
  idempotencyKey: string;  // Client-provided unique key
  // ... request data
}

class IdempotencyService {
  private cache: Map<string, { response: any; timestamp: Date }> = new Map();
  
  async executeWithIdempotency<T>(
    key: string,
    operation: () => Promise<T>,
    ttl: number = 3600000  // 1 hour default
  ): Promise<T> {
    // Check if we've seen this key
    const cached = this.cache.get(key);
    if (cached) {
      // Return cached response
      return cached.response as T;
    }
    
    // Execute operation
    const result = await operation();
    
    // Cache result
    this.cache.set(key, {
      response: result,
      timestamp: new Date()
    });
    
    // Cleanup after TTL
    setTimeout(() => {
      this.cache.delete(key);
    }, ttl);
    
    return result;
  }
}

// Usage
app.post('/events', async (req, res) => {
  const { idempotencyKey, ...eventData } = req.body;
  
  const event = await idempotencyService.executeWithIdempotency(
    idempotencyKey,
    async () => {
      return await eventService.createEvent(eventData);
    }
  );
  
  res.json(event);
});
```

#### Exactly-Once Processing

```typescript
// Exactly-once event processing

class EventProcessor {
  async processEvent(eventId: string): Promise<void> {
    // 1. Check if already processed
    const processed = await db.query(`
      SELECT * FROM processed_events
      WHERE event_id = ?
    `, [eventId]);
    
    if (processed.length > 0) {
      // Already processed, skip
      return;
    }
    
    // 2. Mark as processing (with transaction)
    await db.transaction(async (tx) => {
      // Insert processing record
      await tx.query(`
        INSERT INTO processed_events (event_id, status, started_at)
        VALUES (?, 'processing', NOW())
      `, [eventId]);
      
      // Process event
      await this.doProcessEvent(eventId);
      
      // Mark as completed
      await tx.query(`
        UPDATE processed_events
        SET status = 'completed', completed_at = NOW()
        WHERE event_id = ?
      `, [eventId]);
    });
  }
}
```

---

## Development Standards

### Code Review Standards

**Not just "do PRs" - define what reviewers check for.**

#### Code Review Checklist

```markdown
## Code Review Checklist

### Functionality
- [ ] Does the code do what it's supposed to do?
- [ ] Are edge cases handled?
- [ ] Are error cases handled?
- [ ] Are there tests? Do they pass?

### Performance
- [ ] Are there any obvious performance issues?
- [ ] Are database queries optimized?
- [ ] Is caching used appropriately?
- [ ] Are there N+1 query problems?

### Security
- [ ] Is input validated?
- [ ] Are SQL injections prevented?
- [ ] Are secrets handled securely?
- [ ] Are permissions checked?

### Observability
- [ ] Are important operations logged?
- [ ] Are metrics emitted?
- [ ] Are errors logged with context?
- [ ] Can we trace this in production?

### Code Quality
- [ ] Is the code readable?
- [ ] Are naming conventions followed?
- [ ] Is there duplicate code?
- [ ] Are comments helpful (not obvious)?

### Documentation
- [ ] Is the code self-documenting?
- [ ] Are complex algorithms explained?
- [ ] Is API documentation updated?
- [ ] Are runbooks updated (if needed)?
```

#### Review Process

```yaml
review_process:
  required_reviewers: 2
  required_approvals: 1
  blocking_labels:
    - "security-review"
    - "architecture-review"
    
  review_timeout: 48_hours
  escalation: "@team-lead"
  
  auto_approve:
    - "dependencies"
    - "documentation"
    - "typos"
    
  require_approval:
    - "security"
    - "database-migrations"
    - "api-changes"
```

### Static Analysis & Linters

#### Linting Rules

```json
// .eslintrc.json
{
  "extends": [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended",
    "plugin:@typescript-eslint/recommended-requiring-type-checking"
  ],
  "rules": {
    "@typescript-eslint/no-explicit-any": "error",
    "@typescript-eslint/no-unused-vars": "error",
    "@typescript-eslint/explicit-function-return-type": "warn",
    "complexity": ["error", 10],  // Max cyclomatic complexity
    "max-lines-per-function": ["warn", 50],
    "max-depth": ["error", 4]
  }
}
```

#### Strict Mode

```json
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "strictPropertyInitialization": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true
  }
}
```

#### Dead Code Detection

```bash
# Use ts-prune to find unused exports
npm install -D ts-prune
npx ts-prune

# Use unimported to find unused files
npm install -D unimported
npx unimported
```

#### Cyclomatic Complexity Limits

```typescript
// Example: Refactor complex function

// ❌ Bad: High complexity (15+)
function processEvent(event: Event): Result {
  if (event.type === 'A') {
    if (event.status === 'active') {
      if (event.priority === 'high') {
        // ... nested conditions
      }
    }
  }
  // ... many more nested conditions
}

// ✅ Good: Low complexity (< 10)
function processEvent(event: Event): Result {
  if (event.type === 'A' && event.status === 'active' && event.priority === 'high') {
    return processHighPriorityActiveEvent(event);
  }
  
  if (event.type === 'A' && event.status === 'active') {
    return processActiveEvent(event);
  }
  
  return processDefaultEvent(event);
}
```

### Feature Flags & Progressive Rollouts

**Feature flags are a safety system, not just product tooling.**

#### Feature Flag Implementation

```typescript
// Feature flag service

interface FeatureFlag {
  name: string;
  enabled: boolean;
  rolloutPercentage: number;  // 0-100
  targetUsers?: string[];     // Specific user IDs
  targetTenants?: string[];   // Specific tenant IDs
  killSwitch: boolean;        // Emergency disable
}

class FeatureFlagService {
  async isEnabled(
    flagName: string,
    userId?: string,
    tenantId?: string
  ): Promise<boolean> {
    const flag = await this.getFlag(flagName);
    
    // Kill switch - always disabled
    if (flag.killSwitch) {
      return false;
    }
    
    // Global disable
    if (!flag.enabled) {
      return false;
    }
    
    // User targeting
    if (flag.targetUsers && userId) {
      return flag.targetUsers.includes(userId);
    }
    
    // Tenant targeting
    if (flag.targetTenants && tenantId) {
      return flag.targetTenants.includes(tenantId);
    }
    
    // Percentage rollout
    if (flag.rolloutPercentage < 100) {
      const hash = this.hashUserId(userId || 'anonymous');
      return (hash % 100) < flag.rolloutPercentage;
    }
    
    return true;
  }
  
  // Kill switch - emergency disable
  async killSwitch(flagName: string): Promise<void> {
    await this.updateFlag(flagName, { killSwitch: true });
    // Alert on-call
    await this.alertOnCall(`Feature flag ${flagName} killed`);
  }
}

// Usage
const useNewScraper = await featureFlags.isEnabled(
  'new-scraper-v2',
  userId,
  tenantId
);

if (useNewScraper) {
  await newScraper.scrape();
} else {
  await oldScraper.scrape();
}
```

#### Progressive Rollout Strategy

```yaml
rollout_strategy:
  phase_1_canary:
    percentage: 1%
    duration: 24_hours
    metrics_to_watch:
      - error_rate
      - latency_p99
      - success_rate
    rollback_threshold:
      error_rate: "> 1%"
      latency_p99: "> 2x baseline"
      
  phase_2_gradual:
    percentage: 10%
    duration: 48_hours
    metrics_to_watch: [same as phase 1]
    
  phase_3_broad:
    percentage: 50%
    duration: 72_hours
    
  phase_4_full:
    percentage: 100%
    after_validation: true
```

---

## Security Practices

### Secrets Management

**Explicitly document secrets handling.**

#### Secrets Rotation

```yaml
secrets_rotation:
  database_passwords:
    frequency: 90_days
    process:
      1. Generate new password
      2. Update in secret manager
      3. Update application config (blue-green deployment)
      4. Verify connection
      5. Deprecate old password after 7 days
      
  api_keys:
    frequency: 180_days
    process:
      1. Generate new key
      2. Add to secret manager (both old and new)
      3. Update application
      4. Monitor for errors
      5. Remove old key after 30 days
      
  jwt_secrets:
    frequency: 365_days
    process:
      1. Generate new secret
      2. Support both secrets during rotation
      3. Update application
      4. Remove old secret after token expiry period
```

#### Auditability

```typescript
// Secrets access auditing

class SecretsAuditor {
  async logSecretAccess(
    secretName: string,
    userId: string,
    action: 'read' | 'write' | 'delete'
  ): Promise<void> {
    await auditLog.create({
      type: 'secret_access',
      secretName,
      userId,
      action,
      timestamp: new Date(),
      ipAddress: req.ip,
      userAgent: req.headers['user-agent']
    });
    
    // Alert on suspicious access
    if (action === 'delete' || this.isOffHours()) {
      await this.alertSecurityTeam({
        secretName,
        userId,
        action,
        severity: 'high'
      });
    }
  }
}
```

#### No Secrets in Logs

```typescript
// Sanitize logs

class LogSanitizer {
  private secretPatterns = [
    /password=["']([^"']+)["']/gi,
    /api[_-]?key=["']([^"']+)["']/gi,
    /secret=["']([^"']+)["']/gi,
    /token=["']([^"']+)["']/gi
  ];
  
  sanitize(log: string): string {
    let sanitized = log;
    
    this.secretPatterns.forEach(pattern => {
      sanitized = sanitized.replace(pattern, (match, value) => {
        return match.replace(value, '***REDACTED***');
      });
    });
    
    return sanitized;
  }
}

// Usage
logger.info(this.logSanitizer.sanitize(
  `Connecting with password=${dbPassword}`
));
// Logs: "Connecting with password=***REDACTED***"
```

### Threat Modeling

**Lightweight but documented.**

#### Threat Model Template

```markdown
# Threat Model: [System/Feature Name]

## System Overview

- **Purpose**: [What does this system do?]
- **Entry Points**: [How is it accessed?]
- **Trust Boundaries**: [Where do we trust vs verify?]
- **High-Value Assets**: [What needs protection?]

## Threat Analysis

### STRIDE Analysis

| Threat | Description | Mitigation |
|--------|-------------|------------|
| **S**poofing | [Identity spoofing threats] | [Mitigations] |
| **T**ampering | [Data tampering threats] | [Mitigations] |
| **R**epudiation | [Non-repudiation threats] | [Mitigations] |
| **I**nformation Disclosure | [Data exposure threats] | [Mitigations] |
| **D**enial of Service | [DoS threats] | [Mitigations] |
| **E**levation of Privilege | [Privilege escalation] | [Mitigations] |

## Attack Surface

### Entry Points
1. **API Endpoints**
   - Threats: SQL injection, XSS, CSRF
   - Mitigations: Input validation, parameterized queries, CSRF tokens

2. **Authentication**
   - Threats: Brute force, credential stuffing
   - Mitigations: Rate limiting, MFA, account lockout

### Trust Boundaries
- **External → API**: Untrusted, verify everything
- **API → Database**: Trusted network, still validate
- **API → External Service**: Untrusted, timeout and retry

## High-Value Assets

1. **User Data**
   - Protection: Encryption at rest, encryption in transit
   - Access: Role-based access control

2. **Payment Information**
   - Protection: PCI-DSS compliance, tokenization
   - Access: Minimal access, audit logging

## Risk Assessment

| Risk | Likelihood | Impact | Severity | Mitigation Priority |
|------|------------|--------|----------|---------------------|
| SQL Injection | Low | High | High | P0 |
| DDoS | Medium | Medium | Medium | P1 |
| Data Breach | Low | Critical | Critical | P0 |
```

### Supply Chain Security

**Modern must-have.**

#### Dependency Provenance

```bash
# Verify package integrity
npm audit
npm audit --audit-level=moderate

# Use lockfiles
npm ci  # Install from lockfile only

# Verify package signatures (when available)
npm install --package-lock-only
```

#### CI Pipeline Trust

```yaml
# .github/workflows/secure-build.yml
name: Secure Build

on: [push, pull_request]

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      # Verify dependencies
      - name: Verify lockfile
        run: |
          npm ci --package-lock-only
          git diff --exit-code package-lock.json
      
      # Security scanning
      - name: Run Snyk
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
      
      # Check for known vulnerabilities
      - name: npm audit
        run: npm audit --audit-level=moderate
      
      # Verify build artifacts
      - name: Build
        run: npm run build
      
      # Sign artifacts (if applicable)
      - name: Sign artifacts
        run: |
          # Sign with GPG or similar
          gpg --sign build/artifact.tar.gz
```

#### Build Artifact Signing

```bash
# Sign build artifacts
gpg --detach-sign --armor build/app.tar.gz

# Verify signature
gpg --verify build/app.tar.gz.asc build/app.tar.gz
```

---

## Operational Excellence

### Incident Postmortems

**Blameless, structured.**

#### Postmortem Template

```markdown
# Postmortem: [Incident Title]

**Date**: YYYY-MM-DD  
**Duration**: [Start time] - [End time]  
**Severity**: P0 | P1 | P2 | P3  
**Status**: Resolved  

## Summary

[One paragraph summary of what happened]

## Impact

- **Users Affected**: [Number/percentage]
- **Duration**: [How long]
- **Services Affected**: [List services]
- **Revenue Impact**: [If applicable]

## Timeline

| Time | Event |
|------|-------|
| HH:MM | [Event description] |
| HH:MM | [Event description] |

## Root Cause

[What actually caused the incident?]

## Contributing Factors

- [Factor 1]
- [Factor 2]
- [Factor 3]

## What Went Well

- [Positive thing 1]
- [Positive thing 2]

## What Went Wrong

- [Issue 1]
- [Issue 2]

## Action Items

| Action | Owner | Due Date | Status |
|--------|-------|----------|--------|
| [Action 1] | [@owner] | YYYY-MM-DD | [ ] |
| [Action 2] | [@owner] | YYYY-MM-DD | [ ] |

## Lessons Learned

- [Lesson 1]
- [Lesson 2]

## References

- [Links to monitoring, logs, etc.]
```

### Runbook Testing

**Runbooks that aren't exercised will fail.**

#### Game Days

```markdown
## Game Day Schedule

### Quarterly Game Days

**Purpose**: Validate runbooks and incident response

**Format**:
1. **Scenario**: Simulate realistic incident
2. **Participants**: On-call team
3. **Duration**: 2-4 hours
4. **Debrief**: Post-mortem on game day itself

### Example Scenarios

1. **Database Connection Pool Exhaustion**
   - Simulate: Block database connections
   - Expected: Team follows runbook, escalates appropriately
   - Validation: Runbook works, monitoring alerts

2. **API Latency Spike**
   - Simulate: Slow downstream service
   - Expected: Circuit breaker activates, fallback works
   - Validation: System degrades gracefully

3. **Data Corruption**
   - Simulate: Corrupted data in database
   - Expected: Team identifies issue, restores from backup
   - Validation: Backup restoration process works
```

#### Fire Drills

```yaml
fire_drills:
  frequency: monthly
  duration: 30_minutes
  scenarios:
    - name: "High Error Rate"
      simulate: "Inject errors into API"
      runbook: "docs/runbooks/high-error-rate.md"
      
    - name: "Database Slowdown"
      simulate: "Slow database queries"
      runbook: "docs/runbooks/database-slow.md"
      
    - name: "Cache Outage"
      simulate: "Disable Redis"
      runbook: "docs/runbooks/cache-outage.md"
```

#### On-Call Onboarding

```markdown
## On-Call Onboarding Checklist

### Before First On-Call Shift

- [ ] Read all runbooks
- [ ] Complete fire drill scenarios
- [ ] Shadow experienced on-call engineer for 1 week
- [ ] Know escalation path
- [ ] Have access to all systems
- [ ] Know how to access logs and metrics
- [ ] Complete incident response training

### First Week On-Call

- [ ] Pair with experienced engineer
- [ ] Handle at least 1 real incident (with support)
- [ ] Document any runbook gaps found
- [ ] Review postmortems from past incidents

### Ongoing

- [ ] Participate in monthly fire drills
- [ ] Review and update runbooks quarterly
- [ ] Attend postmortem meetings
```

---

## Documentation Meta-Standards

### Documentation Freshness Rules

#### Ownership Per Doc

```yaml
documentation_ownership:
  api-documentation.md:
    owner: "@backend-team"
    review_cadence: "monthly"
    last_validated: "2026-01-15"
    
  security.md:
    owner: "@security-team"
    review_cadence: "quarterly"
    last_validated: "2026-01-01"
    
  runbooks/:
    owner: "@sre-team"
    review_cadence: "monthly"
    last_validated: "2026-01-20"
```

#### Review Cadence

```markdown
## Documentation Review Schedule

- **Weekly**: Runbooks (active incidents)
- **Monthly**: API docs, operational docs
- **Quarterly**: Architecture docs, security docs
- **Annually**: High-level overview docs

### Review Process

1. Owner reviews doc for accuracy
2. Updates "Last Validated" date
3. Flags outdated sections
4. Updates or deprecates as needed
```

#### "Last Validated" Dates

```markdown
---
**Last Updated**: 2026-01-28  
**Last Validated**: 2026-01-28  
**Next Review**: 2026-02-28  
**Owner**: @team-name  
---
```

### Glossary / Ubiquitous Language

**Especially important with LLM systems.**

#### Domain Glossary

```markdown
# Glossary

## Terms

### Event
- **Definition**: A ticketed occurrence at a venue (concert, sports game, etc.)
- **Context**: Used in `stubhub_events` table, API endpoints
- **Synonyms**: None
- **Related**: Venue, Ticket, Date

### Confidence
- **Definition**: Match confidence score for reconciliation (low, medium, high, very_high)
- **Context**: `event_reconciliation_review.match_confidence`
- **Synonyms**: Match Score
- **Related**: Reconciliation, Match Status

### Review
- **Definition**: Human review of reconciliation match
- **Context**: `event_reconciliation_review.review_status` (pending, approved, rejected, bot)
- **Synonyms**: Manual Review, Human Review
- **Related**: Reconciliation, Bot Decision

### Decision
- **Definition**: Final determination on reconciliation match
- **Context**: Used in reconciliation workflow
- **Synonyms**: Match Decision, Resolution
- **Related**: Review, Confidence, Match Status

### Run
- **Definition**: A single execution of the scraper across all venues
- **Context**: `stubhub_venue_run.run_id`
- **Synonyms**: Scraping Run, Execution
- **Related**: Venue, Progress, Events

### Venue
- **Definition**: A physical location where events occur
- **Context**: Scraped from StubHub, stored in venues table
- **Synonyms**: Location, Site
- **Related**: Event, Run
```

#### Shared Vocabulary

```markdown
## Vocabulary Standards

### Code
- Use domain terms consistently
- Match database column names
- Match API parameter names

### UI
- Use same terms as code
- Tooltips link to glossary
- Consistent labeling

### Documentation
- Glossary as source of truth
- Link to glossary from docs
- Update glossary when adding terms
```

---

## AI-Specific Practices

### Model Governance

#### Versioning

```typescript
// Model versioning

interface ModelVersion {
  id: string;
  version: string;  // e.g., "1.2.3"
  modelName: string;
  trainedAt: Date;
  performance: {
    accuracy: number;
    precision: number;
    recall: number;
  };
  deployedAt?: Date;
  deprecatedAt?: Date;
}

class ModelRegistry {
  async deployModel(version: ModelVersion): Promise<void> {
    // 1. Validate performance meets thresholds
    if (version.performance.accuracy < 0.95) {
      throw new Error('Model accuracy below threshold');
    }
    
    // 2. Register version
    await this.registerVersion(version);
    
    // 3. Deploy with feature flag
    await featureFlags.create({
      name: `model-${version.modelName}-v${version.version}`,
      enabled: false,
      rolloutPercentage: 0
    });
  }
  
  async rollbackModel(modelName: string): Promise<void> {
    const currentVersion = await this.getCurrentVersion(modelName);
    const previousVersion = await this.getPreviousVersion(modelName);
    
    // Disable current
    await featureFlags.disable(`model-${modelName}-v${currentVersion.version}`);
    
    // Enable previous
    await featureFlags.enable(`model-${modelName}-v${previousVersion.version}`);
  }
}
```

#### Evaluation Criteria

```yaml
model_evaluation:
  thresholds:
    accuracy: ">= 0.95"
    precision: ">= 0.90"
    recall: ">= 0.90"
    f1_score: ">= 0.90"
    
  test_sets:
    - name: "validation"
      size: 1000
    - name: "test"
      size: 5000
    - name: "edge_cases"
      size: 100
      
  evaluation_process:
    1. Train model
    2. Evaluate on validation set
    3. If passes thresholds, evaluate on test set
    4. Evaluate on edge cases
    5. Deploy with feature flag (0% rollout)
    6. Gradual rollout with monitoring
```

### Human-in-the-Loop Standards

#### When Humans Intervene

```typescript
// Human-in-the-loop decision points

interface AIDecision {
  confidence: number;  // 0-1
  decision: string;
  reasoning: string;
}

class HumanInTheLoop {
  async shouldRequireHumanReview(decision: AIDecision): Promise<boolean> {
    // Low confidence requires review
    if (decision.confidence < 0.7) {
      return true;
    }
    
    // High-value decisions require review
    if (this.isHighValueDecision(decision)) {
      return true;
    }
    
    // Edge cases require review
    if (this.isEdgeCase(decision)) {
      return true;
    }
    
    return false;
  }
  
  async requestHumanReview(decision: AIDecision): Promise<HumanReview> {
    // Create review task
    const review = await reviewService.create({
      decision,
      status: 'pending',
      createdAt: new Date()
    });
    
    // Notify reviewers
    await notificationService.notify({
      channel: '#ai-reviews',
      message: `Human review required: ${review.id}`
    });
    
    return review;
  }
}
```

#### Confidence Thresholds

```yaml
confidence_thresholds:
  auto_approve:
    threshold: ">= 0.95"
    action: "Automatically approve"
    
  auto_reject:
    threshold: "< 0.5"
    action: "Automatically reject"
    
  human_review:
    threshold: "0.5 - 0.95"
    action: "Require human review"
    
  high_confidence_bot:
    threshold: ">= 0.9"
    action: "Bot decision, log for audit"
```

#### Audit Trail of Overrides

```typescript
// Audit trail for human overrides

interface OverrideAudit {
  decisionId: string;
  aiDecision: AIDecision;
  humanDecision: string;
  humanUserId: string;
  reason: string;
  timestamp: Date;
}

class OverrideAuditor {
  async logOverride(
    decisionId: string,
    aiDecision: AIDecision,
    humanDecision: string,
    userId: string,
    reason: string
  ): Promise<void> {
    await auditLog.create({
      type: 'ai_override',
      decisionId,
      aiDecision,
      humanDecision,
      humanUserId: userId,
      reason,
      timestamp: new Date()
    });
    
    // Analyze override patterns
    await this.analyzeOverridePattern(decisionId, aiDecision, humanDecision);
  }
  
  async analyzeOverridePattern(
    decisionId: string,
    aiDecision: AIDecision,
    humanDecision: string
  ): Promise<void> {
    // If humans consistently override AI in certain cases,
    // flag for model retraining
    if (this.isConsistentOverride(decisionId)) {
      await this.flagForRetraining(decisionId);
    }
  }
}
```

### Prompt & Output Contracts

#### Structured Outputs

```typescript
// Structured output schema

interface StructuredOutput {
  event: {
    name: string;
    venue: string;
    date: string;  // ISO 8601
  };
  confidence: number;  // 0-1
  reasoning: string;
  sources: string[];  // URLs or references
}

// Validate against schema
function validateStructuredOutput(output: any): output is StructuredOutput {
  return (
    typeof output.event === 'object' &&
    typeof output.event.name === 'string' &&
    typeof output.event.venue === 'string' &&
    typeof output.event.date === 'string' &&
    typeof output.confidence === 'number' &&
    output.confidence >= 0 &&
    output.confidence <= 1 &&
    typeof output.reasoning === 'string' &&
    Array.isArray(output.sources)
  );
}
```

#### Confidence Scores

```typescript
// Confidence score calculation

interface ConfidenceScore {
  overall: number;  // 0-1
  components: {
    nameMatch: number;
    dateMatch: number;
    venueMatch: number;
  };
  reasoning: string;
}

class ConfidenceCalculator {
  calculateConfidence(match: Match): ConfidenceScore {
    const nameMatch = this.calculateNameSimilarity(match.name1, match.name2);
    const dateMatch = this.calculateDateMatch(match.date1, match.date2);
    const venueMatch = this.calculateVenueMatch(match.venue1, match.venue2);
    
    // Weighted average
    const overall = (
      nameMatch * 0.4 +
      dateMatch * 0.4 +
      venueMatch * 0.2
    );
    
    return {
      overall,
      components: {
        nameMatch,
        dateMatch,
        venueMatch
      },
      reasoning: `Name: ${nameMatch}, Date: ${dateMatch}, Venue: ${venueMatch}`
    };
  }
}
```

#### Schema Validation

```typescript
// Schema validation for AI outputs

import { z } from 'zod';

const EventSchema = z.object({
  name: z.string().min(1),
  venue: z.string().min(1),
  date: z.string().datetime(),
  confidence: z.number().min(0).max(1),
  reasoning: z.string(),
  sources: z.array(z.string().url())
});

function validateAIOutput(output: unknown): StructuredOutput {
  try {
    return EventSchema.parse(output);
  } catch (error) {
    // Log validation error
    logger.error('AI output validation failed', { error, output });
    
    // Return safe default or throw
    throw new Error('Invalid AI output format');
  }
}
```

---

## Performance Engineering

### Performance Budgets

**What happens when budgets are exceeded? Block deployment or require performance review.**

#### Page Load Time Budgets

```yaml
performance_budgets:
  web_vitals:
    lcp: "< 2.5s"  # Largest Contentful Paint
    fid: "< 100ms"  # First Input Delay
    cls: "< 0.1"  # Cumulative Layout Shift
    fcp: "< 1.8s"  # First Contentful Paint
    
  api_response_times:
    critical_endpoints:
      "/api/events": "< 200ms p95"
      "/api/users": "< 150ms p95"
    standard_endpoints:
      "/api/venues": "< 500ms p95"
      "/api/reconciliation": "< 1000ms p95"
      
  bundle_sizes:
    initial_js: "< 200KB gzipped"
    total_js: "< 500KB gzipped"
    css: "< 50KB gzipped"
    images: "< 500KB per image"
    
  database_queries:
    simple_queries: "< 50ms p95"
    complex_queries: "< 200ms p95"
    report_queries: "< 2000ms p95"
```

#### Budget Enforcement

```typescript
// Performance budget enforcement

class PerformanceBudget {
  async checkBudget(metric: string, value: number): Promise<BudgetResult> {
    const budget = this.getBudget(metric);
    
    if (value > budget.threshold) {
      // Budget exceeded
      await this.reportViolation(metric, value, budget.threshold);
      
      // Block deployment if critical
      if (budget.critical) {
        throw new Error(`Performance budget exceeded: ${metric}`);
      }
      
      // Require performance review
      await this.requireReview(metric, value, budget.threshold);
    }
    
    return { passed: true, metric, value, threshold: budget.threshold };
  }
}

// CI/CD integration
// .github/workflows/performance-check.yml
async function checkPerformanceBudgets() {
  const budgets = new PerformanceBudget();
  
  // Check bundle size
  const bundleSize = await getBundleSize();
  await budgets.checkBudget('initial_js', bundleSize);
  
  // Check API response times
  const apiMetrics = await getAPIMetrics();
  await budgets.checkBudget('api_events', apiMetrics.p95);
  
  // Check database query times
  const dbMetrics = await getDBMetrics();
  await budgets.checkBudget('simple_queries', dbMetrics.p95);
}
```

### Load Testing Strategy

#### Baseline Performance Tests in CI

```yaml
# .github/workflows/performance-baseline.yml
name: Performance Baseline

on: [pull_request]

jobs:
  performance:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Run baseline performance tests
        run: |
          npm run test:performance:baseline
          
      - name: Compare against baseline
        run: |
          npm run test:performance:compare
          
      - name: Fail if regression detected
        run: |
          if [ $? -ne 0 ]; then
            echo "Performance regression detected"
            exit 1
          fi
```

#### Pre-Production Load Testing

```typescript
// Load testing with k6

import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 100 },  // Ramp up
    { duration: '5m', target: 100 },   // Sustained load
    { duration: '2m', target: 200 },  // Spike
    { duration: '5m', target: 200 },   // Sustained spike
    { duration: '2m', target: 0 },    // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],  // 95% of requests < 500ms
    http_req_failed: ['rate<0.01'],     // Error rate < 1%
  },
};

export default function () {
  const response = http.get('https://api.example.com/events');
  
  check(response, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });
  
  sleep(1);
}
```

#### Traffic Replay from Production

```typescript
// Traffic replay system

class TrafficReplay {
  async replayTraffic(
    startTime: Date,
    endTime: Date,
    speedMultiplier: number = 1.0
  ): Promise<void> {
    // Load production traffic logs
    const trafficLogs = await this.loadTrafficLogs(startTime, endTime);
    
    // Replay at specified speed
    for (const log of trafficLogs) {
      const delay = log.timestamp - previousLog.timestamp;
      await sleep(delay / speedMultiplier);
      
      // Replay request
      await this.replayRequest(log.request);
    }
  }
  
  async replayRequest(request: RequestLog): Promise<void> {
    // Replay with same parameters
    const response = await fetch(request.url, {
      method: request.method,
      headers: request.headers,
      body: request.body
    });
    
    // Compare response
    this.compareResponse(request.response, response);
  }
}
```

#### Sustained Load Tests

```yaml
sustained_load_testing:
  schedule: "weekly"
  duration: "24_hours"
  target_load: "80% of peak capacity"
  
  scenarios:
    - name: "normal_load"
      duration: "12_hours"
      target_qps: 100
      
    - name: "peak_load"
      duration: "4_hours"
      target_qps: 200
      
    - name: "stress_test"
      duration: "2_hours"
      target_qps: 300
      
  metrics_to_monitor:
    - response_time_p95
    - error_rate
    - memory_usage
    - cpu_utilization
    - database_connection_pool
```

#### Performance Regression Detection

```typescript
// Performance regression detection

class PerformanceRegressionDetector {
  async detectRegression(
    currentMetrics: Metrics,
    baselineMetrics: Metrics
  ): Promise<RegressionReport> {
    const regressions: Regression[] = [];
    
    // Check response time
    if (currentMetrics.p95 > baselineMetrics.p95 * 1.2) {
      regressions.push({
        metric: 'response_time_p95',
        current: currentMetrics.p95,
        baseline: baselineMetrics.p95,
        degradation: (currentMetrics.p95 / baselineMetrics.p95 - 1) * 100
      });
    }
    
    // Check error rate
    if (currentMetrics.errorRate > baselineMetrics.errorRate * 1.5) {
      regressions.push({
        metric: 'error_rate',
        current: currentMetrics.errorRate,
        baseline: baselineMetrics.errorRate,
        degradation: (currentMetrics.errorRate / baselineMetrics.errorRate - 1) * 100
      });
    }
    
    return { regressions, severity: this.calculateSeverity(regressions) };
  }
}
```

### Resource Efficiency

#### Cost Per Transaction Metrics

```typescript
// Cost per transaction tracking

class CostTracker {
  async calculateCostPerTransaction(
    service: string,
    period: DateRange
  ): Promise<CostMetrics> {
    const costs = await this.getServiceCosts(service, period);
    const transactions = await this.getTransactionCount(service, period);
    
    return {
      totalCost: costs.total,
      transactionCount: transactions.count,
      costPerTransaction: costs.total / transactions.count,
      breakdown: {
        compute: costs.compute / transactions.count,
        database: costs.database / transactions.count,
        cache: costs.cache / transactions.count,
        external: costs.external / transactions.count
      }
    };
  }
  
  async trackCostTrends(service: string): Promise<CostTrend> {
    const metrics = await Promise.all([
      this.calculateCostPerTransaction(service, this.getLastWeek()),
      this.calculateCostPerTransaction(service, this.getThisWeek())
    ]);
    
    return {
      change: metrics[1].costPerTransaction - metrics[0].costPerTransaction,
      percentageChange: ((metrics[1].costPerTransaction / metrics[0].costPerTransaction) - 1) * 100,
      trend: metrics[1].costPerTransaction > metrics[0].costPerTransaction ? 'increasing' : 'decreasing'
    };
  }
}
```

#### Memory Leak Detection

```typescript
// Memory leak detection

class MemoryLeakDetector {
  private memorySnapshots: MemorySnapshot[] = [];
  
  async detectLeaks(): Promise<LeakReport> {
    // Take memory snapshot
    const snapshot = await this.takeSnapshot();
    this.memorySnapshots.push(snapshot);
    
    // Keep last 10 snapshots
    if (this.memorySnapshots.length > 10) {
      this.memorySnapshots.shift();
    }
    
    // Detect trend
    if (this.memorySnapshots.length >= 5) {
      const trend = this.analyzeTrend(this.memorySnapshots);
      
      if (trend.isIncreasing && trend.rate > 0.1) {
        return {
          detected: true,
          rate: trend.rate,
          severity: this.calculateSeverity(trend.rate),
          recommendations: this.getRecommendations()
        };
      }
    }
    
    return { detected: false };
  }
  
  private analyzeTrend(snapshots: MemorySnapshot[]): Trend {
    const values = snapshots.map(s => s.heapUsed);
    const slope = this.calculateSlope(values);
    
    return {
      isIncreasing: slope > 0,
      rate: slope / values[0]  // Percentage increase per snapshot
    };
  }
}
```

#### Database Connection Efficiency

```typescript
// Database connection pool monitoring

class ConnectionPoolMonitor {
  async monitorPool(pool: Pool): Promise<PoolMetrics> {
    return {
      total: pool.totalCount,
      idle: pool.idleCount,
      waiting: pool.waitingCount,
      utilization: (pool.totalCount - pool.idleCount) / pool.totalCount,
      
      // Alert if utilization > 80%
      alert: (pool.totalCount - pool.idleCount) / pool.totalCount > 0.8
    };
  }
  
  async optimizePool(pool: Pool, metrics: PoolMetrics): Promise<void> {
    if (metrics.utilization > 0.8) {
      // Increase pool size
      await pool.resize(metrics.total * 1.5);
    } else if (metrics.utilization < 0.3) {
      // Decrease pool size
      await pool.resize(metrics.total * 0.8);
    }
  }
}
```

#### Cache Hit Rate Targets

```yaml
cache_hit_rate_targets:
  redis_cache:
    target: "> 80%"
    alert_threshold: "< 70%"
    critical_threshold: "< 50%"
    
  cdn_cache:
    target: "> 90%"
    alert_threshold: "< 85%"
    
  application_cache:
    target: "> 60%"
    alert_threshold: "< 50%"
```

```typescript
// Cache hit rate monitoring

class CacheMonitor {
  async trackHitRate(cache: Cache): Promise<HitRateMetrics> {
    const stats = await cache.getStats();
    
    const hitRate = stats.hits / (stats.hits + stats.misses);
    
    return {
      hitRate,
      hits: stats.hits,
      misses: stats.misses,
      target: 0.8,
      status: hitRate >= 0.8 ? 'healthy' : 'degraded'
    };
  }
  
  async alertIfLow(hitRate: HitRateMetrics): Promise<void> {
    if (hitRate.hitRate < hitRate.target) {
      await this.sendAlert({
        severity: 'warning',
        message: `Cache hit rate below target: ${hitRate.hitRate}`,
        recommendations: [
          'Review cache TTL settings',
          'Check cache key patterns',
          'Consider cache warming'
        ]
      });
    }
  }
}
```

---

## Observability Standards

### Distributed Tracing Standards

#### Trace ID Propagation

```typescript
// Trace ID propagation across services

import { trace, context } from '@opentelemetry/api';

class TracePropagation {
  // Extract trace context from incoming request
  extractTraceContext(req: Request): TraceContext {
    const traceParent = req.headers['traceparent'];
    const traceState = req.headers['tracestate'];
    
    if (traceParent) {
      return this.parseTraceParent(traceParent);
    }
    
    // Create new trace if none exists
    return this.createNewTrace();
  }
  
  // Inject trace context into outgoing request
  injectTraceContext(context: TraceContext, req: Request): void {
    req.headers['traceparent'] = this.formatTraceParent(context);
    req.headers['tracestate'] = this.formatTraceState(context);
  }
  
  // Propagate across async boundaries
  async withTrace<T>(traceContext: TraceContext, fn: () => Promise<T>): Promise<T> {
    return trace.setSpanContext(context.active(), traceContext.spanContext, () => {
      return fn();
    });
  }
}
```

#### Span Naming Conventions

```typescript
// Span naming standards

class SpanNaming {
  // Format: {operation} {resource}
  // Examples:
  // - "GET /api/events"
  // - "POST /api/users"
  // - "SELECT events"
  // - "INSERT event"
  
  nameHTTPRequest(method: string, path: string): string {
    return `${method} ${path}`;
  }
  
  nameDatabaseQuery(operation: string, table: string): string {
    return `${operation} ${table}`;
  }
  
  nameExternalCall(service: string, operation: string): string {
    return `${service}.${operation}`;
  }
  
  nameInternalFunction(module: string, functionName: string): string {
    return `${module}.${functionName}`;
  }
}
```

#### Required Span Attributes

```typescript
// Required span attributes

interface RequiredSpanAttributes {
  // Request identification
  'http.method': string;
  'http.route': string;
  'http.status_code': number;
  
  // User context
  'user.id': string;
  'user.email'?: string;
  
  // Tenant context
  'tenant.id': string;
  
  // Request identification
  'request.id': string;
  
  // Service identification
  'service.name': string;
  'service.version': string;
  
  // Error context
  'error.type'?: string;
  'error.message'?: string;
}

class SpanAttributes {
  addRequiredAttributes(span: Span, req: Request, user?: User): void {
    // HTTP attributes
    span.setAttribute('http.method', req.method);
    span.setAttribute('http.route', req.route);
    
    // User attributes
    if (user) {
      span.setAttribute('user.id', user.id);
      span.setAttribute('user.email', user.email);
    }
    
    // Tenant attributes
    span.setAttribute('tenant.id', req.tenant.id);
    
    // Request ID
    span.setAttribute('request.id', req.id);
    
    // Service attributes
    span.setAttribute('service.name', process.env.SERVICE_NAME);
    span.setAttribute('service.version', process.env.SERVICE_VERSION);
  }
}
```

#### Sampling Strategy

```typescript
// Sampling strategy: 100% for errors, 1% for success

class SamplingStrategy {
  shouldSample(span: Span): boolean {
    // Always sample errors
    if (span.status?.code === SpanStatusCode.ERROR) {
      return true;
    }
    
    // Sample 1% of successful requests
    const sampleRate = 0.01;
    return Math.random() < sampleRate;
  }
  
  // Adaptive sampling based on load
  adaptiveSampling(currentLoad: number): number {
    if (currentLoad > 0.9) {
      return 0.001;  // 0.1% under high load
    } else if (currentLoad > 0.7) {
      return 0.005;  // 0.5% under medium load
    } else {
      return 0.01;   // 1% under normal load
    }
  }
}
```

### Structured Logging Requirements

#### JSON Logging Format

```typescript
// Structured logging with required fields

interface LogEntry {
  timestamp: string;      // ISO 8601
  level: 'DEBUG' | 'INFO' | 'WARN' | 'ERROR';
  service: string;
  trace_id: string;
  span_id?: string;
  message: string;
  [key: string]: any;     // Additional context
}

class StructuredLogger {
  log(level: LogLevel, message: string, context: Record<string, any>): void {
    const entry: LogEntry = {
      timestamp: new Date().toISOString(),
      level,
      service: process.env.SERVICE_NAME || 'unknown',
      trace_id: this.getTraceId(),
      span_id: this.getSpanId(),
      message,
      ...this.sanitizeContext(context)
    };
    
    console.log(JSON.stringify(entry));
  }
  
  private sanitizeContext(context: Record<string, any>): Record<string, any> {
    const sanitized = { ...context };
    
    // Remove sensitive data
    delete sanitized.password;
    delete sanitized.token;
    delete sanitized.secret;
    delete sanitized.apiKey;
    
    // Remove PII
    delete sanitized.ssn;
    delete sanitized.creditCard;
    
    return sanitized;
  }
}
```

#### Log Level Guidelines

```markdown
## Log Level Guidelines

### DEBUG
- Detailed information for diagnosing problems
- Only enabled in development/staging
- Examples: Function entry/exit, variable values

### INFO
- General informational messages
- Normal application flow
- Examples: Request received, operation completed

### WARN
- Warning messages for potentially harmful situations
- Application continues to function
- Examples: Deprecated API usage, retry attempts

### ERROR
- Error events that might still allow the application to continue
- Examples: Failed API call, validation error

### FATAL
- Very severe error events that might cause the application to abort
- Examples: Database connection lost, out of memory
```

### Metrics Cardinality Management

#### Label Cardinality Limits

```yaml
metrics_cardinality:
  limits:
    max_labels_per_metric: 10
    max_cardinality_per_metric: 10000
    
  required_labels:
    - service_name
    - environment
    
  optional_labels:
    - user_id      # High cardinality - use sparingly
    - tenant_id    # Medium cardinality
    - endpoint     # Low cardinality - safe to use
    
  prohibited_labels:
    - request_id   # Too high cardinality
    - session_id   # Too high cardinality
    - timestamp    # Use time series, not labels
```

#### Metric Naming Conventions

```typescript
// Metric naming standards

class MetricNaming {
  // Format: {namespace}_{subsystem}_{name}_{unit}
  // Examples:
  // - http_request_duration_seconds
  // - database_query_duration_seconds
  // - cache_hit_rate_ratio
  // - memory_usage_bytes
  
  nameHTTPMetric(metric: string): string {
    return `http_${metric}_seconds`;
  }
  
  nameDatabaseMetric(metric: string): string {
    return `database_${metric}_seconds`;
  }
  
  nameCacheMetric(metric: string): string {
    return `cache_${metric}_ratio`;
  }
  
  nameMemoryMetric(metric: string): string {
    return `memory_${metric}_bytes`;
  }
}
```

---

## API Design & Versioning

### API Design Standards

#### RESTful Conventions

```typescript
// RESTful API design standards

// Resource naming: Use nouns, plural
// GET /api/events
// GET /api/events/:id
// POST /api/events
// PUT /api/events/:id
// DELETE /api/events/:id

// Nested resources
// GET /api/venues/:venueId/events
// POST /api/venues/:venueId/events

// Query parameters for filtering/sorting
// GET /api/events?status=active&sort=date&limit=20&offset=0
```

#### Error Response Format (RFC 7807 Problem Details)

```typescript
// RFC 7807 Problem Details for HTTP APIs

interface ProblemDetails {
  type: string;           // URI identifying problem type
  title: string;          // Short summary
  status: number;         // HTTP status code
  detail: string;        // Human-readable explanation
  instance: string;       // URI identifying specific occurrence
  [key: string]: any;    // Additional properties
}

// Example error response
{
  "type": "https://api.example.com/problems/validation-error",
  "title": "Validation Failed",
  "status": 400,
  "detail": "The request body failed validation",
  "instance": "/api/events/123",
  "errors": [
    {
      "field": "date",
      "message": "Date must be in the future"
    }
  ]
}
```

#### Pagination Strategy

```typescript
// Cursor-based pagination (preferred for large datasets)

interface CursorPagination {
  limit: number;         // Max items per page
  cursor?: string;       // Opaque cursor token
  hasMore: boolean;      // Whether more pages exist
}

interface PaginatedResponse<T> {
  data: T[];
  pagination: {
    limit: number;
    cursor?: string;
    hasMore: boolean;
    nextCursor?: string;
  };
}

// Offset-based pagination (for small datasets)
interface OffsetPagination {
  page: number;          // Page number (1-indexed)
  limit: number;         // Items per page
  total: number;         // Total items
  totalPages: number;    // Total pages
}
```

#### Rate Limiting Headers

```typescript
// Rate limiting headers

interface RateLimitHeaders {
  'X-RateLimit-Limit': string;      // Max requests per window
  'X-RateLimit-Remaining': string; // Remaining requests
  'X-RateLimit-Reset': string;     // Unix timestamp when limit resets
  'X-RateLimit-Used': string;      // Requests used in current window
}

// Example
// X-RateLimit-Limit: 100
// X-RateLimit-Remaining: 95
// X-RateLimit-Reset: 1640995200
// X-RateLimit-Used: 5
```

#### Request ID Tracking

```typescript
// Request ID propagation

class RequestIDTracker {
  generateRequestID(): string {
    return `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }
  
  extractRequestID(req: Request): string {
    return req.headers['x-request-id'] || this.generateRequestID();
  }
  
  addRequestIDHeader(req: Request, res: Response): void {
    const requestId = this.extractRequestID(req);
    res.setHeader('X-Request-ID', requestId);
    req.id = requestId;
  }
}
```

### API Versioning Strategy

#### URL Versioning vs Header Versioning

```yaml
versioning_strategy:
  method: "url_versioning"  # /api/v1/events
  
  # Alternative: header_versioning
  # Accept: application/vnd.api+json;version=1
  
  url_format: "/api/v{version}/{resource}"
  
  examples:
    - "/api/v1/events"
    - "/api/v2/events"
```

#### Deprecation Timeline

```markdown
## Deprecation Timeline

1. **Announce** (6 months before removal)
   - Add `Deprecation` header to responses
   - Update documentation
   - Notify API consumers

2. **Warn** (3 months before removal)
   - Increase deprecation warnings
   - Provide migration guide
   - Set sunset date

3. **Remove** (after sunset date)
   - Endpoint returns 410 Gone
   - Redirect to new endpoint (if applicable)
```

#### Sunset Header

```typescript
// Sunset header for deprecated endpoints

interface SunsetHeader {
  'Sunset': string;  // RFC 8594 - Date when endpoint will be removed
  'Deprecation': string;  // RFC 8594 - Date when endpoint was deprecated
  'Link': string;  // Link to replacement endpoint
}

// Example
// Sunset: Sat, 31 Dec 2026 23:59:59 GMT
// Deprecation: Sat, 30 Jun 2026 23:59:59 GMT
// Link: </api/v2/events>; rel="successor-version"
```

#### Breaking Change Definition

```markdown
## Breaking Changes

### What Constitutes a Breaking Change?

- Removing an endpoint
- Removing a required field
- Changing field type
- Making optional field required
- Changing authentication/authorization requirements
- Changing response format

### Non-Breaking Changes

- Adding new endpoints
- Adding optional fields
- Adding new response fields
- Improving error messages
- Performance improvements
```

#### Backward Compatibility Requirements

```typescript
// Backward compatibility enforcement

class APIVersionManager {
  async validateBackwardCompatibility(
    oldVersion: APIVersion,
    newVersion: APIVersion
  ): Promise<CompatibilityReport> {
    const breakingChanges = this.detectBreakingChanges(oldVersion, newVersion);
    
    if (breakingChanges.length > 0) {
      return {
        compatible: false,
        breakingChanges,
        recommendation: 'Create new major version'
      };
    }
    
    return {
      compatible: true,
      breakingChanges: []
    };
  }
}
```

### API Documentation

#### OpenAPI/Swagger Specs Required

```yaml
# See api-documentation.md for full details
# Requirements:
# - All endpoints must have OpenAPI spec
# - Specs must be kept in sync with code
# - Auto-generated from code preferred
# - Validated in CI/CD
```

#### Interactive Documentation

```markdown
## Interactive Documentation Requirements

- **Swagger UI**: Available at `/api-docs/swagger`
- **ReDoc**: Available at `/api-docs/redoc`
- **Postman Collection**: Exportable from OpenAPI spec
- **Example Requests**: All endpoints must have examples
- **Example Responses**: Success and error responses documented
```

---

## Testing Strategy

### Test Pyramid Enforcement

```yaml
test_pyramid:
  unit_tests:
    percentage: 70%
    description: "Fast, isolated, test individual functions"
    target_duration: "< 1 minute"
    
  integration_tests:
    percentage: 20%
    description: "Test database, external services"
    target_duration: "< 5 minutes"
    
  e2e_tests:
    percentage: 10%
    description: "Critical user paths only"
    target_duration: "< 10 minutes"
    
  total_suite_duration: "< 10 minutes"
```

#### Test Quality Standards

```typescript
// Deterministic tests - no flaky tests

describe('Event Service', () => {
  // ❌ Bad: Non-deterministic
  it('should create event', async () => {
    const event = await createEvent({ date: new Date() }); // Uses current time
  });
  
  // ✅ Good: Deterministic
  it('should create event', async () => {
    const fixedDate = new Date('2026-01-01T12:00:00Z');
    const event = await createEvent({ date: fixedDate });
  });
});

// Test data factories
class EventFactory {
  static create(overrides?: Partial<Event>): Event {
    return {
      id: faker.string.uuid(),
      name: faker.lorem.words(3),
      venue: faker.company.name(),
      date: faker.date.future().toISOString(),
      ...overrides
    };
  }
}
```

#### Contract Testing

```typescript
// Contract testing for service boundaries

// Consumer contract (frontend)
describe('Events API Contract', () => {
  it('should return events list', async () => {
    const response = await pact.mockService
      .given('events exist')
      .uponReceiving('a request for events')
      .withRequest({
        method: 'GET',
        path: '/api/events'
      })
      .willRespondWith({
        status: 200,
        body: {
          events: [
            { id: '123', name: 'Concert', venue: 'Venue' }
          ]
        }
      });
  });
});
```

#### Mutation Testing

```yaml
# Mutation testing for critical code paths

mutation_testing:
  enabled: true
  paths:
    - "src/services/payment/"
    - "src/services/auth/"
    - "src/services/reconciliation/"
    
  threshold: 80%  # Minimum mutation score
  tools:
    - "Stryker"
```

### Testing in Production

#### Synthetic Monitoring

```typescript
// Synthetic monitoring - test critical flows every 5 minutes

class SyntheticMonitor {
  async runSyntheticTests(): Promise<TestResults> {
    const tests = [
      this.testUserLogin(),
      this.testEventCreation(),
      this.testPaymentProcessing(),
      this.testDataReconciliation()
    ];
    
    const results = await Promise.all(tests);
    
    // Alert if any test fails
    const failures = results.filter(r => !r.passed);
    if (failures.length > 0) {
      await this.alertOnCall(failures);
    }
    
    return { results, failures: failures.length };
  }
  
  // Schedule: Every 5 minutes
  schedule('*/5 * * * *', () => {
    this.runSyntheticTests();
  });
}
```

#### Smoke Tests After Deployment

```yaml
# Smoke tests after deployment

smoke_tests:
  triggers:
    - "after_deployment"
    - "after_config_change"
    
  tests:
    - name: "Health check"
      endpoint: "/health"
      expected_status: 200
      
    - name: "Database connectivity"
      test: "SELECT 1"
      expected_result: "success"
      
    - name: "Cache connectivity"
      test: "PING redis"
      expected_result: "PONG"
      
  failure_action: "rollback_deployment"
```

#### Canary Analysis Automation

```typescript
// Canary analysis automation

class CanaryAnalyzer {
  async analyzeCanary(
    canaryMetrics: Metrics,
    baselineMetrics: Metrics
  ): Promise<CanaryResult> {
    // Compare error rates
    const errorRateDiff = canaryMetrics.errorRate - baselineMetrics.errorRate;
    
    // Compare latency
    const latencyDiff = canaryMetrics.p95 - baselineMetrics.p95;
    
    // Compare throughput
    const throughputDiff = canaryMetrics.throughput - baselineMetrics.throughput;
    
    // Decision logic
    if (errorRateDiff > 0.01 || latencyDiff > 100) {
      return {
        passed: false,
        reason: 'Error rate or latency degradation detected',
        rollback: true
      };
    }
    
    return {
      passed: true,
      reason: 'Canary metrics within acceptable range',
      rollback: false
    };
  }
}
```

---

## Configuration Management

### Configuration as Code

```typescript
// All config in version control

// config/base.yaml
database:
  host: ${DB_HOST}
  port: ${DB_PORT}
  name: ${DB_NAME}

// config/development.yaml
database:
  host: localhost
  port: 5432

// config/production.yaml
database:
  host: ${DB_HOST}  # From environment
  port: ${DB_PORT}
  
// Environment-specific overrides (not separate files)
// Use environment variables for overrides
```

#### Configuration Validation

```typescript
// Configuration validation on startup

import { z } from 'zod';

const ConfigSchema = z.object({
  database: z.object({
    host: z.string().min(1),
    port: z.number().int().positive(),
    name: z.string().min(1)
  }),
  api: z.object({
    port: z.number().int().positive(),
    timeout: z.number().positive()
  })
});

class ConfigValidator {
  validate(config: unknown): Config {
    try {
      return ConfigSchema.parse(config);
    } catch (error) {
      logger.error('Configuration validation failed', { error });
      process.exit(1);  // Fail fast on invalid config
    }
  }
}
```

### Feature Configuration

#### Separate Feature Flags from Infrastructure Config

```typescript
// Feature flags separate from infrastructure

// Feature flags service
class FeatureFlagService {
  async getFlag(name: string): Promise<FeatureFlag> {
    // Load from feature flag service (LaunchDarkly, etc.)
    return await featureFlagClient.getFlag(name);
  }
}

// Infrastructure config
const config = {
  database: { host: '...' },
  cache: { host: '...' }
};
```

#### Configuration Change Audit Trail

```typescript
// Configuration change audit trail

class ConfigAuditor {
  async logConfigChange(
    key: string,
    oldValue: any,
    newValue: any,
    userId: string
  ): Promise<void> {
    await auditLog.create({
      type: 'config_change',
      key,
      oldValue,
      newValue,
      userId,
      timestamp: new Date(),
      ipAddress: req.ip
    });
  }
}
```

#### Gradual Config Rollout

```yaml
# Gradual configuration rollout

config_rollout:
  strategy: "canary"
  
  phases:
    - percentage: 10%
      duration: 1_hour
      monitor: true
      
    - percentage: 50%
      duration: 2_hours
      monitor: true
      
    - percentage: 100%
      after_validation: true
```

### Dynamic Configuration

#### Hot-Reload for Non-Critical Config

```typescript
// Hot-reload for non-critical config

class ConfigHotReload {
  async reloadConfig(): Promise<void> {
    // Reload non-critical config
    const newConfig = await this.loadConfig();
    
    // Validate
    this.validateConfig(newConfig);
    
    // Apply
    this.applyConfig(newConfig);
    
    // Notify
    await this.notifyConfigChange(newConfig);
  }
  
  // Critical config (database, auth) requires restart
  isCritical(key: string): boolean {
    return ['database', 'auth', 'secrets'].includes(key);
  }
}
```

#### Configuration Change Notifications

```typescript
// Configuration change notifications

class ConfigNotifier {
  async notifyChange(change: ConfigChange): Promise<void> {
    await Promise.all([
      this.notifySlack(change),
      this.notifyOnCall(change),
      this.logChange(change)
    ]);
  }
}
```

#### Config Drift Detection

```typescript
// Config drift detection between environments

class ConfigDriftDetector {
  async detectDrift(
    env1: Environment,
    env2: Environment
  ): Promise<DriftReport> {
    const config1 = await this.getConfig(env1);
    const config2 = await this.getConfig(env2);
    
    const differences = this.compareConfigs(config1, config2);
    
    return {
      hasDrift: differences.length > 0,
      differences,
      recommendations: this.getRecommendations(differences)
    };
  }
}
```

---

## Service Mesh / Networking Standards

### Service-to-Service Communication

#### mTLS Everywhere

```yaml
# mTLS configuration

mtls:
  enabled: true
  required: true
  
  exceptions:
    - "legacy_service"  # Requires justification
    - "external_api"    # Not in service mesh
    
  certificate_rotation:
    frequency: "90_days"
    auto_rotation: true
```

#### Circuit Breaker Configuration

```typescript
// Circuit breaker configuration

interface CircuitBreakerConfig {
  failureThreshold: number;    // Open after N failures
  timeout: number;             // Time before retry (ms)
  resetTimeout: number;        // Time before half-open (ms)
  monitoringPeriod: number;    // Time window for failures (ms)
}

const defaultCircuitBreaker: CircuitBreakerConfig = {
  failureThreshold: 5,         // Open after 5 failures
  timeout: 60000,              // 1 minute before retry
  resetTimeout: 300000,        // 5 minutes before half-open
  monitoringPeriod: 60000      // 1 minute window
};
```

#### Retry Policy

```typescript
// Retry policy with exponential backoff

interface RetryPolicy {
  maxRetries: number;
  initialDelay: number;
  maxDelay: number;
  backoffMultiplier: number;
  retryableErrors: number[];  // HTTP status codes
}

const defaultRetryPolicy: RetryPolicy = {
  maxRetries: 3,
  initialDelay: 1000,         // 1 second
  maxDelay: 30000,            // 30 seconds
  backoffMultiplier: 2,
  retryableErrors: [500, 502, 503, 504]  // Retry on server errors
};

class RetryHandler {
  async executeWithRetry<T>(
    fn: () => Promise<T>,
    policy: RetryPolicy = defaultRetryPolicy
  ): Promise<T> {
    let attempt = 0;
    let delay = policy.initialDelay;
    
    while (attempt < policy.maxRetries) {
      try {
        return await fn();
      } catch (error) {
        attempt++;
        
        if (attempt >= policy.maxRetries) {
          throw error;
        }
        
        // Check if error is retryable
        if (!this.isRetryable(error, policy)) {
          throw error;
        }
        
        // Exponential backoff
        await sleep(delay);
        delay = Math.min(delay * policy.backoffMultiplier, policy.maxDelay);
      }
    }
    
    throw new Error('Max retries exceeded');
  }
}
```

#### Timeout Standards

```yaml
timeout_standards:
  connection_timeout: "5s"
  read_timeout: "30s"
  write_timeout: "30s"
  
  service_specific:
    database:
      connection_timeout: "10s"
      query_timeout: "30s"
      
    external_api:
      connection_timeout: "5s"
      read_timeout: "60s"
      
    internal_service:
      connection_timeout: "2s"
      read_timeout: "10s"
```

### Service Discovery

#### Health Check Requirements

```typescript
// Liveness vs Readiness probes

// Liveness: Is the service running?
// Readiness: Is the service ready to accept traffic?

class HealthChecker {
  async livenessCheck(): Promise<HealthStatus> {
    // Simple check - is process alive?
    return {
      status: 'healthy',
      timestamp: new Date()
    };
  }
  
  async readinessCheck(): Promise<HealthStatus> {
    // Check dependencies
    const checks = await Promise.all([
      this.checkDatabase(),
      this.checkCache(),
      this.checkExternalServices()
    ]);
    
    const allHealthy = checks.every(c => c.healthy);
    
    return {
      status: allHealthy ? 'ready' : 'not_ready',
      checks,
      timestamp: new Date()
    };
  }
}
```

#### Graceful Degradation

```typescript
// Graceful degradation when dependencies unavailable

class ServiceWithFallback {
  async getData(id: string): Promise<Data> {
    try {
      // Try primary service
      return await this.primaryService.getData(id);
    } catch (error) {
      // Fallback to cache
      const cached = await this.cache.get(id);
      if (cached) {
        return cached;
      }
      
      // Fallback to default
      return this.getDefaultData(id);
    }
  }
}
```

---

## Cost Engineering

### Cost Visibility

#### Cost Per Service/Feature Tracking

```typescript
// Cost per service/feature tracking

class CostTracker {
  async getServiceCosts(service: string, period: DateRange): Promise<ServiceCosts> {
    const costs = await this.cloudProvider.getCosts({
      service,
      period,
      groupBy: ['resource_type', 'environment']
    });
    
    return {
      total: costs.total,
      breakdown: {
        compute: costs.compute,
        storage: costs.storage,
        network: costs.network,
        database: costs.database
      },
      perTransaction: costs.total / this.getTransactionCount(service, period)
    };
  }
}
```

#### Resource Tagging Strategy

```yaml
# Resource tagging for cost allocation

resource_tags:
  required:
    - "service"        # Service name
    - "environment"    # dev, staging, prod
    - "team"           # Team owner
    - "cost_center"    # Cost center code
    
  optional:
    - "project"        # Project name
    - "feature"        # Feature flag
    - "version"        # Service version
```

#### Monthly Cost Review

```markdown
## Monthly Cost Review Process

1. **Generate Report**: Cost breakdown by service/team
2. **Review Anomalies**: Investigate cost spikes
3. **Identify Optimization**: Find waste, unused resources
4. **Set Targets**: Cost targets for next month
5. **Track Trends**: Compare month-over-month
```

#### Cost Anomaly Detection

```typescript
// Cost anomaly detection

class CostAnomalyDetector {
  async detectAnomalies(service: string): Promise<Anomaly[]> {
    const currentCost = await this.getCurrentCost(service);
    const historicalCosts = await this.getHistoricalCosts(service, 30);
    
    const average = this.calculateAverage(historicalCosts);
    const stdDev = this.calculateStdDev(historicalCosts);
    
    // Alert if cost > 2 standard deviations from mean
    if (currentCost > average + 2 * stdDev) {
      return [{
        service,
        currentCost,
        expectedCost: average,
        deviation: currentCost - average,
        severity: 'high'
      }];
    }
    
    return [];
  }
}
```

### Cost Optimization

#### Right-Sizing Recommendations

```typescript
// Right-sizing recommendations

class RightSizingAnalyzer {
  async analyzeResource(resource: Resource): Promise<Recommendation> {
    const utilization = await this.getUtilization(resource, 30);
    
    if (utilization.average < 0.3) {
      return {
        action: 'downsize',
        current: resource.size,
        recommended: this.calculateOptimalSize(utilization),
        savings: this.calculateSavings(resource, utilization)
      };
    }
    
    if (utilization.average > 0.9) {
      return {
        action: 'upsize',
        current: resource.size,
        recommended: this.calculateOptimalSize(utilization),
        risk: 'performance_degradation'
      };
    }
    
    return { action: 'no_change' };
  }
}
```

#### Unused Resource Cleanup

```yaml
# Unused resource cleanup automation

unused_resource_cleanup:
  schedule: "weekly"
  
  resources_to_check:
    - "unattached_volumes"
    - "unused_load_balancers"
    - "idle_instances"
    - "orphaned_snapshots"
    
  retention_periods:
    unattached_volumes: "30_days"
    idle_instances: "7_days"
    orphaned_snapshots: "90_days"
    
  notification:
    before_deletion: "7_days"
    after_deletion: "immediate"
```

#### Reserved Capacity Strategy

```yaml
# Reserved capacity vs on-demand

capacity_strategy:
  reserved_instances:
    use_for: "baseline_load"
    commitment: "1_year"
    savings: "~40%"
    
  on_demand:
    use_for: "variable_load"
    flexibility: "high"
    
  spot_instances:
    use_for: "batch_jobs"
    savings: "~70%"
    risk: "interruption"
```

---

## Compliance & Audit

### Audit Logging

#### Who Did What, When, To Which Resource

```typescript
// Comprehensive audit logging

interface AuditLog {
  timestamp: Date;
  userId: string;
  userEmail: string;
  action: string;
  resourceType: string;
  resourceId: string;
  resourceName?: string;
  changes?: Record<string, { old: any; new: any }>;
  ipAddress: string;
  userAgent: string;
  result: 'success' | 'failure';
  errorMessage?: string;
}

class AuditLogger {
  async log(action: AuditLog): Promise<void> {
    // Immutable, append-only log
    await this.appendToAuditLog(action);
    
    // Also send to SIEM
    await this.sendToSIEM(action);
  }
}
```

#### Immutable Audit Logs

```typescript
// Immutable audit logs (append-only)

class ImmutableAuditLog {
  async append(log: AuditLog): Promise<void> {
    // Write to append-only storage (S3, WAL)
    await this.appendOnlyStorage.append(log);
    
    // Verify integrity
    await this.verifyIntegrity();
  }
  
  async verifyIntegrity(): Promise<boolean> {
    // Check hash chain
    const hash = await this.calculateHash();
    return await this.verifyHashChain(hash);
  }
}
```

#### Audit Log Retention

```yaml
audit_log_retention:
  financial_transactions: "7_years"
  access_logs: "3_years"
  security_events: "7_years"
  general_audit: "1_year"
  
  archival:
    after: "90_days"
    storage: "cold_storage"
    format: "compressed_json"
```

#### Audit Log Review Cadence

```markdown
## Audit Log Review Schedule

- **Daily**: Automated anomaly detection
- **Weekly**: High-privilege access review
- **Monthly**: Full audit log review
- **Quarterly**: Compliance audit
- **Annually**: External audit
```

### Compliance Automation

#### Automated Compliance Checks

```typescript
// Automated compliance checks (SOC2, GDPR, HIPAA)

class ComplianceChecker {
  async checkSOC2(): Promise<ComplianceReport> {
    const checks = await Promise.all([
      this.checkAccessControls(),
      this.checkEncryption(),
      this.checkAuditLogging(),
      this.checkIncidentResponse()
    ]);
    
    return {
      compliant: checks.every(c => c.passed),
      checks,
      score: this.calculateScore(checks)
    };
  }
  
  async checkGDPR(): Promise<ComplianceReport> {
    const checks = await Promise.all([
      this.checkDataMinimization(),
      this.checkRightToErasure(),
      this.checkDataPortability(),
      this.checkConsentManagement()
    ]);
    
    return {
      compliant: checks.every(c => c.passed),
      checks
    };
  }
}
```

#### Policy-as-Code

```typescript
// Policy-as-code with OPA (Open Policy Agent)

// policies/compliance.rego
package compliance

default allow = false

allow {
  input.user.role == "admin"
  input.action == "read"
}

allow {
  input.user.role == "user"
  input.action == "read"
  input.resource.owner == input.user.id
}

// Usage
const policy = await opa.loadPolicy('compliance.rego');
const result = await policy.evaluate(request);
if (!result.allow) {
  throw new Error('Access denied');
}
```

#### Compliance Drift Detection

```typescript
// Compliance drift detection

class ComplianceDriftDetector {
  async detectDrift(): Promise<DriftReport> {
    const currentState = await this.getCurrentComplianceState();
    const expectedState = await this.getExpectedComplianceState();
    
    const drift = this.compareStates(currentState, expectedState);
    
    return {
      hasDrift: drift.length > 0,
      drift,
      severity: this.calculateSeverity(drift)
    };
  }
}
```

### Access Control

#### Principle of Least Privilege

```typescript
// Least privilege enforcement

class AccessController {
  async grantAccess(userId: string, resource: string, permission: string): Promise<void> {
    // Check if permission is minimal
    const minimalPermission = this.getMinimalPermission(resource, permission);
    
    if (permission !== minimalPermission) {
      throw new Error(`Use minimal permission: ${minimalPermission}`);
    }
    
    await this.grant(userId, resource, permission);
  }
}
```

#### Regular Access Reviews

```yaml
# Access review schedule

access_reviews:
  frequency: "quarterly"
  
  review_scope:
    - "admin_access"
    - "database_access"
    - "production_access"
    - "secrets_access"
    
  process:
    1. "Generate access report"
    2. "Send to managers for review"
    3. "Revoke unused access"
    4. "Document decisions"
```

#### Automated Access Expiration

```typescript
// Automated access expiration

class AccessExpiration {
  async expireTemporaryAccess(): Promise<void> {
    const expiredAccess = await this.findExpiredAccess();
    
    for (const access of expiredAccess) {
      await this.revokeAccess(access);
      await this.notifyUser(access);
      await this.logRevocation(access);
    }
  }
  
  async createTemporaryAccess(
    userId: string,
    resource: string,
    duration: number
  ): Promise<Access> {
    const expiresAt = new Date(Date.now() + duration);
    
    return await this.grantAccess(userId, resource, expiresAt);
  }
}
```

---

## Disaster Recovery & Business Continuity

### Backup Strategy

#### RPO and RTO Definitions

```yaml
# Recovery Point Objective (RPO) and Recovery Time Objective (RTO)

backup_strategy:
  critical_databases:
    rpo: "15_minutes"  # Max data loss: 15 minutes
    rto: "1_hour"      # Max downtime: 1 hour
    backup_frequency: "every_15_minutes"
    
  standard_databases:
    rpo: "1_hour"
    rto: "4_hours"
    backup_frequency: "hourly"
    
  application_data:
    rpo: "24_hours"
    rto: "8_hours"
    backup_frequency: "daily"
```

#### Backup Testing Schedule

```yaml
# Backup testing schedule

backup_testing:
  restore_drills:
    frequency: "monthly"
    scope: "random_backup"
    validation: "data_integrity_check"
    
  full_restore:
    frequency: "quarterly"
    scope: "full_database"
    environment: "disaster_recovery_environment"
    
  documentation:
    - "Restore procedure"
    - "Time to restore"
    - "Issues encountered"
    - "Improvements needed"
```

#### Cross-Region Backup Replication

```yaml
# Cross-region backup replication

backup_replication:
  primary_region: "us-east-1"
  backup_regions:
    - "us-west-2"
    - "eu-west-1"
    
  replication_strategy: "synchronous"
  replication_lag: "< 1_minute"
  
  failover:
    automatic: false
    manual_trigger: true
    rto: "15_minutes"
```

### Disaster Recovery Scenarios

#### Region Failure Recovery

```markdown
## Region Failure Recovery Procedure

1. **Detection**: Monitor for region-wide failures
2. **Assessment**: Evaluate impact and scope
3. **Failover**: Switch to backup region
4. **Validation**: Verify services operational
5. **Communication**: Notify stakeholders
6. **Recovery**: Restore primary region when available
```

#### Data Center Evacuation

```markdown
## Data Center Evacuation Plan

1. **Trigger**: Physical threat to data center
2. **Failover**: Route traffic to backup data center
3. **Data Sync**: Ensure data consistency
4. **Communication**: Update status page
5. **Recovery**: Return to primary when safe
```

#### Third-Party Service Failure

```markdown
## Third-Party Service Failure Contingency

1. **Detection**: Monitor third-party service health
2. **Fallback**: Switch to backup provider (if available)
3. **Degradation**: Operate in degraded mode
4. **Communication**: Notify users of limitations
5. **Recovery**: Resume normal operation when available
```

### Business Continuity Testing

#### Annual DR Exercises

```yaml
# Annual disaster recovery exercises

dr_exercises:
  frequency: "annually"
  duration: "4_hours"
  
  scenarios:
    - "region_failure"
    - "database_corruption"
    - "ransomware_attack"
    - "data_center_fire"
    
  participants:
    - "engineering_team"
    - "operations_team"
    - "management"
    
  deliverables:
    - "DR_exercise_report"
    - "lessons_learned"
    - "improvement_plan"
```

#### Tabletop Exercises

```markdown
## Tabletop Exercise Process

1. **Scenario**: Present disaster scenario
2. **Discussion**: Team discusses response
3. **Documentation**: Record decisions and actions
4. **Review**: Compare with actual procedures
5. **Improvement**: Update procedures based on findings
```

---

## Team Practices & Knowledge Management

### On-Call Health

#### On-Call Rotation

```yaml
# On-call rotation policy

on_call:
  rotation:
    minimum_duration: "1_week"
    maximum_duration: "2_weeks"
    handoff_day: "Monday"
    
  compensation:
    hourly_rate: "1.5x_normal_rate"
    minimum_payment: "4_hours"
    
  time_off:
    after_on_call: "1_day_off"
    maximum_pages_per_shift: 5
```

#### Post-On-Call Retrospectives

```markdown
## Post-On-Call Retrospective

### Questions to Answer

1. How many pages did you receive?
2. Were pages actionable?
3. Were runbooks helpful?
4. What improvements are needed?
5. What went well?
```

### Knowledge Sharing

#### Bi-Weekly Tech Talks

```yaml
# Tech talk schedule

tech_talks:
  frequency: "bi_weekly"
  duration: "30_minutes"
  format: "presentation"
  
  topics:
    - "New technologies"
    - "Architecture decisions"
    - "Incident learnings"
    - "Best practices"
    
  rotation: "all_engineers"
```

#### Architecture Review Board

```markdown
## Architecture Review Board

### Purpose
Review cross-team architecture changes

### Process
1. Submit architecture proposal
2. Review by board members
3. Discussion and feedback
4. Approval or request changes
5. Implementation tracking
```

### Technical Debt Management

#### Technical Debt Register

```typescript
// Technical debt tracking

interface TechnicalDebt {
  id: string;
  description: string;
  impact: 'low' | 'medium' | 'high' | 'critical';
  effort: 'small' | 'medium' | 'large' | 'xlarge';
  created: Date;
  dueDate?: Date;
  owner: string;
  status: 'open' | 'in_progress' | 'resolved';
}

class TechnicalDebtRegister {
  async addDebt(debt: TechnicalDebt): Promise<void> {
    await this.debtTracker.create(debt);
  }
  
  async prioritizeDebt(): Promise<TechnicalDebt[]> {
    // Prioritize by impact and effort
    return this.debtTracker.findAll()
      .sort((a, b) => this.calculatePriority(b) - this.calculatePriority(a));
  }
}
```

#### Tech Debt Allocation

```yaml
# Technical debt allocation

tech_debt_allocation:
  percentage: "20%"
  description: "20% of sprint capacity for tech debt"
  
  quarterly_review:
    frequency: "quarterly"
    actions:
      - "Review debt register"
      - "Prioritize items"
      - "Allocate resources"
      - "Track progress"
```

---

## Mobile-Specific Practices

### Mobile Release Management

#### Phased Rollouts

```yaml
# Mobile phased rollout

mobile_rollout:
  phases:
    - percentage: 5%
      duration: "24_hours"
      monitor: true
      
    - percentage: 25%
      duration: "48_hours"
      monitor: true
      
    - percentage: 50%
      duration: "72_hours"
      monitor: true
      
    - percentage: 100%
      after_validation: true
```

#### Force Update Strategy

```typescript
// Force update for critical bugs

class ForceUpdateManager {
  async checkUpdateRequired(version: string): Promise<UpdateStatus> {
    const minVersion = await this.getMinRequiredVersion();
    
    if (this.compareVersions(version, minVersion) < 0) {
      return {
        updateRequired: true,
        forceUpdate: true,
        message: 'Critical security update required'
      };
    }
    
    return { updateRequired: false };
  }
}
```

#### Backward Compatibility

```yaml
# Backward compatibility policy

backward_compatibility:
  server_versions: "N-2"
  description: "Mobile app supports last 2 server versions"
  
  deprecation_timeline:
    announce: "3_months_before"
    deprecate: "1_month_before"
    remove: "after_deprecation"
```

### Mobile Performance

#### App Startup Time Budgets

```yaml
# Mobile performance budgets

mobile_performance:
  startup_time:
    cold_start: "< 3_seconds"
    warm_start: "< 1_second"
    
  bundle_size:
    ios: "< 50MB"
    android: "< 100MB"
    
  memory_usage:
    target: "< 150MB"
    critical: "> 300MB"
```

#### Battery Usage Monitoring

```typescript
// Battery usage monitoring

class BatteryMonitor {
  async trackBatteryUsage(feature: string): Promise<BatteryMetrics> {
    const usage = await this.getBatteryUsage(feature);
    
    if (usage > this.getThreshold(feature)) {
      await this.alertHighUsage(feature, usage);
    }
    
    return {
      feature,
      usage,
      threshold: this.getThreshold(feature)
    };
  }
}
```

---

## Environmental / Sustainability

### Carbon Footprint

#### Energy-Efficient Architecture

```yaml
# Energy-efficient architecture choices

sustainability:
  region_selection:
    preference: "renewable_energy_regions"
    regions:
      - "us-west-2"  # AWS - high renewable energy
      - "eu-west-1"  # AWS - high renewable energy
      
  resource_optimization:
    - "Right-size instances"
    - "Use spot instances for batch jobs"
    - "Optimize database queries"
    - "Implement caching"
    
  carbon_footprint:
    tracking: true
    reporting: "quarterly"
    target: "reduce_by_20%_yearly"
```

#### Carbon Footprint Reporting

```typescript
// Carbon footprint reporting

class CarbonFootprintTracker {
  async calculateFootprint(service: string): Promise<CarbonFootprint> {
    const resources = await this.getServiceResources(service);
    
    const footprint = resources.reduce((total, resource) => {
      return total + this.calculateResourceFootprint(resource);
    }, 0);
    
    return {
      service,
      footprint,  // kg CO2 equivalent
      breakdown: this.getBreakdown(resources)
    };
  }
}
```

---

## Standards Evolution & Compliance

### Standards Evolution Process

#### How Standards Are Proposed

```markdown
## Standards Proposal Process

1. **Proposal**: Create RFC (Request for Comments)
2. **Review**: Architecture review board reviews
3. **Discussion**: Team discussion and feedback
4. **Approval**: Board approves or requests changes
5. **Adoption**: Standard is adopted and documented
6. **Review**: Annual review of standards
```

#### RFC Process

```markdown
## RFC Template

# RFC-XXX: [Standard Name]

**Status**: Proposed | Under Review | Approved | Rejected  
**Author**: [Name]  
**Date**: YYYY-MM-DD  

## Summary
[One paragraph summary]

## Motivation
[Why is this standard needed?]

## Proposal
[Detailed proposal]

## Alternatives Considered
[Other options considered]

## Impact
[Who/what is affected?]

## Implementation Plan
[How will this be implemented?]
```

#### Variance/Exception Request Process

```markdown
## Exception Request Process

1. **Request**: Submit exception request
2. **Justification**: Explain why exception is needed
3. **Review**: Architecture board reviews
4. **Decision**: Approve or deny
5. **Documentation**: Document exception and rationale
6. **Review**: Periodic review of exceptions
```

### Standards Compliance Measurement

#### Automated Compliance Checking

```typescript
// Automated standards compliance checking

class StandardsComplianceChecker {
  async checkCompliance(service: string): Promise<ComplianceReport> {
    const checks = await Promise.all([
      this.checkCodeStandards(service),
      this.checkAPIDesign(service),
      this.checkTestingStandards(service),
      this.checkDocumentation(service),
      this.checkSecurityStandards(service)
    ]);
    
    return {
      service,
      score: this.calculateScore(checks),
      checks,
      compliant: checks.every(c => c.passed)
    };
  }
}
```

#### Team Scorecards

```markdown
## Team Scorecards

### Purpose
Track standards adoption (not for punishment, for improvement)

### Metrics
- Code quality score
- Test coverage
- Documentation completeness
- Security compliance
- Performance budgets

### Review
- Monthly team review
- Quarterly cross-team comparison
- Focus on improvement, not blame
```

### Onboarding & Training

#### Standards Onboarding

```markdown
## Standards Onboarding Checklist

- [ ] Read architecture practices guide
- [ ] Complete coding standards training
- [ ] Review API design standards
- [ ] Understand testing requirements
- [ ] Learn security practices
- [ ] Complete certification/sign-off
```

#### Standards Champion Role

```markdown
## Standards Champion

### Responsibilities
- Advocate for standards
- Answer questions
- Review compliance
- Suggest improvements
- Onboard new team members

### Per Team
- One standards champion per team
- Rotates quarterly
- Attends architecture review board
```

---

## Summary

### The Big Missing Buckets

If you had to prioritize, focus on:

1. **Decision capture (ADR)** - Prevents tribal knowledge loss
2. **Reliability contracts (SLOs)** - Difference between observed and managed
3. **Ownership & accountability** - Every incident starts with "who owns this?"
4. **Data lifecycle discipline** - Retention, deletion, contracts
5. **Feature flags & safe rollout** - Safety system for changes
6. **Incident learning loops** - Postmortems and runbook testing
7. **Performance budgets** - Prevent performance degradation
8. **API design standards** - Consistent, maintainable APIs
9. **Testing strategy** - Quality at scale
10. **Configuration management** - Infrastructure as code

### Implementation Priority

#### Tier 1: Implement Now (Critical)

**API Versioning & Design Standards**
- [ ] API design standards (RESTful conventions, error format)
- [ ] API versioning strategy
- [ ] OpenAPI/Swagger specs for all endpoints
- [ ] Request ID tracking

**Test Strategy Enforcement**
- [ ] Test pyramid (70% unit, 20% integration, 10% E2E)
- [ ] Deterministic tests (no flaky tests)
- [ ] Test data factories
- [ ] Contract testing for service boundaries

**Configuration Management**
- [ ] Configuration as code (all config in version control)
- [ ] Configuration validation on startup
- [ ] Feature flags separate from infrastructure config
- [ ] Configuration change audit trail

**Service Mesh Communication Standards**
- [ ] mTLS everywhere (or justification for exceptions)
- [ ] Circuit breaker configuration
- [ ] Retry policy with exponential backoff
- [ ] Timeout standards

**Audit Logging**
- [ ] Comprehensive audit logs (who, what, when, where)
- [ ] Immutable audit logs (append-only)
- [ ] Audit log retention policies
- [ ] Regular audit log reviews

#### Tier 2: Next Quarter (Important)

**Performance Budgets**
- [ ] Page load time budgets (LCP, FID, CLS)
- [ ] API response time budgets per endpoint
- [ ] Bundle size budgets
- [ ] Database query time budgets
- [ ] Budget enforcement in CI/CD

**Cost Engineering**
- [ ] Cost per service/feature tracking
- [ ] Resource tagging strategy
- [ ] Monthly cost reviews
- [ ] Cost anomaly detection
- [ ] Right-sizing recommendations

**Disaster Recovery Procedures**
- [ ] RPO/RTO definitions per service
- [ ] Backup testing schedule (monthly restore drills)
- [ ] Cross-region backup replication
- [ ] DR scenario runbooks
- [ ] Annual DR exercises

**Technical Debt Management**
- [ ] Technical debt register
- [ ] 20% time allocation for tech debt
- [ ] Quarterly tech debt review
- [ ] Deprecation tracking

**Distributed Tracing Standards**
- [ ] Trace ID propagation across services
- [ ] Span naming conventions
- [ ] Required span attributes (user_id, tenant_id, request_id)
- [ ] Sampling strategy (100% errors, 1% success)

#### Tier 3: Medium Term (Valuable)

**Compliance Automation**
- [ ] Automated compliance checks (SOC2, GDPR)
- [ ] Policy-as-code (OPA, Cedar)
- [ ] Compliance drift detection
- [ ] Remediation runbooks

**Knowledge Sharing Practices**
- [ ] Bi-weekly tech talks
- [ ] Incident review sharing
- [ ] Architecture review board
- [ ] Engineering blog

**Standards Evolution Process**
- [ ] RFC process for standards changes
- [ ] Standards review cadence (annual)
- [ ] Variance/exception request process
- [ ] Standards adoption tracking

**On-Call Health Metrics**
- [ ] On-call rotation (1-2 weeks)
- [ ] Maximum 5 pages per shift
- [ ] Post-on-call retrospectives
- [ ] On-call compensation/time-off policy

**Mobile-Specific Standards** (If Applicable)
- [ ] Phased rollouts (5% → 25% → 50% → 100%)
- [ ] Force update strategy
- [ ] Backward compatibility (N-2 server versions)
- [ ] App startup time budgets
- [ ] Battery usage monitoring

#### Tier 4: Advanced (Long Term)

**Full SRE Practices**
- [ ] Complete SLO/SLI definitions
- [ ] Error budget management
- [ ] Capacity planning automation
- [ ] Advanced chaos engineering

**Observability Standards**
- [ ] Structured logging requirements (JSON, required fields)
- [ ] Log level guidelines
- [ ] Metrics cardinality management
- [ ] Distributed tracing implementation

**Cost Optimization**
- [ ] Unused resource cleanup automation
- [ ] Reserved capacity strategy
- [ ] Cost/performance tradeoff documentation
- [ ] Carbon footprint tracking

**Environmental Sustainability**
- [ ] Energy-efficient architecture choices
- [ ] Region selection (renewable energy)
- [ ] Carbon footprint reporting
- [ ] Resource utilization efficiency

**Standards Compliance Measurement**
- [ ] Automated compliance checking
- [ ] Team scorecards (improvement-focused)
- [ ] Standards adoption metrics
- [ ] Improvement tracking over time

**Onboarding & Training**
- [ ] Standards onboarding checklist
- [ ] Certification/sign-off requirements
- [ ] Standards reference documentation
- [ ] "Standards champion" role per team

---

**Last Updated**: 2026-01-28  
**Owner**: Architecture Team, SRE Team  
**Review**: Quarterly
