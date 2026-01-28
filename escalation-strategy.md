# Escalation & Alerting Strategy
## Multi-Channel Notification and Escalation Framework

---
**Status**: 💡 **GUIDANCE** - Strategic recommendations for incident escalation  
**Priority**: 🔴 **P0** - Critical for production operations  
**Last Updated**: 2026-01-28  
**Owned By**: Infrastructure Team, DevOps  
---

## Table of Contents

1. [Overview](#overview)
2. [Escalation Channels](#escalation-channels)
3. [Alert Detection Sources](#alert-detection-sources)
4. [Escalation Levels & Timing](#escalation-levels--timing)
5. [Channel Selection Matrix](#channel-selection-matrix)
6. [Implementation Patterns](#implementation-patterns)
7. [Integration Examples](#integration-examples)
8. [Best Practices](#best-practices)

---

## Overview

### Escalation Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    Alert Detection                           │
│  [Logs]  [Metrics]  [Health Checks]  [External Monitors]   │
└─────────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────────┐
│                  Alert Aggregation                           │
│              [Prometheus Alertmanager]                       │
└─────────────────────────────────────────────────────────────┘
                        ↓
        ┌───────────────┴───────────────┐
        ↓                               ↓
┌───────────────┐              ┌───────────────┐
│  GUI Grid     │              │  Notifications │
│  (Dashboard)  │              │  (Multi-Channel)│
└───────────────┘              └───────────────┘
        │                               │
        │                       ┌───────┴───────┐
        │                       ↓               ↓
        │              ┌──────────┐      ┌──────────┐
        │              │  Email   │      │ SMS/WA   │
        │              └──────────┘      └──────────┘
        │
        └──────────────→ [On-Call Engineer]
                                ↓
                        [Escalation Path]
```

### Key Principles

1. **Fail-Safe**: If one channel fails, others should still work
2. **Redundancy**: Critical alerts use multiple channels
3. **Progressive Escalation**: Start with less intrusive channels
4. **Context-Rich**: Each alert includes actionable information
5. **Rate Limiting**: Prevent alert fatigue

---

## Escalation Channels

### 1. Logs (Detection & Investigation)

**Purpose**: Primary detection source and investigation tool

**Characteristics**:
- **Real-time**: Stream logs for immediate detection
- **Searchable**: Full-text search across all log entries
- **Structured**: JSON logs with trace IDs for correlation
- **Retention**: Hot logs (7 days), cold storage (90 days)

**Use Cases**:
- Error pattern detection
- Performance anomaly detection
- Security event detection
- Debugging and root cause analysis

**Integration**:
```typescript
// Log-based alert detection
import { logger } from './logger';

// Pattern-based alerting
logger.on('error', (logEntry) => {
  if (logEntry.errorRate > 0.05) {
    alertManager.trigger({
      severity: 'critical',
      source: 'logs',
      message: `Error rate spike detected: ${logEntry.errorRate}`,
      traceId: logEntry.traceId
    });
  }
});
```

**Tools**:
- **File-based**: Direct log file monitoring (`tail -f`)
- **Aggregated**: Elasticsearch + Kibana for search
- **Streaming**: Kafka → Elasticsearch for real-time analysis

---

### 2. GUI Grid (Dashboard Visualization)

**Purpose**: Visual monitoring and alert acknowledgment

**Characteristics**:
- **Real-time**: Live dashboards with auto-refresh
- **Visual**: Graphs, charts, heatmaps
- **Interactive**: Click-through to detailed views
- **Customizable**: Role-based dashboards

**Use Cases**:
- **Operational Dashboard**: Current system health
- **Alert Dashboard**: Active alerts and their status
- **Performance Dashboard**: Latency, throughput, errors
- **Business Dashboard**: User metrics, revenue, conversions

**Dashboard Components**:

```yaml
# Grafana Dashboard Configuration
dashboard:
  title: "Production Operations"
  
  panels:
    - title: "Error Rate"
      type: graph
      query: |
        rate(http_errors_total[5m]) / 
        rate(http_requests_total[5m]) * 100
      alert:
        condition: "> 5%"
        severity: critical
        
    - title: "Active Alerts"
      type: alertlist
      sources:
        - prometheus
        - elasticsearch
        
    - title: "System Health"
      type: stat
      query: up{job="api"}
      thresholds:
        - value: 0
          color: red
          severity: critical
```

**Alert Acknowledgment**:
- Engineers acknowledge alerts directly in GUI
- Acknowledgment prevents duplicate notifications
- Status visible to entire team

**Integration**:
- **Prometheus + Grafana**: Metrics visualization
- **Elasticsearch + Kibana**: Log visualization
- **Custom Dashboards**: Business-specific views

---

### 3. Email (Non-Urgent Notifications)

**Purpose**: Detailed notifications for non-critical alerts

**Characteristics**:
- **Rich Content**: HTML emails with graphs, links
- **Asynchronous**: Doesn't require immediate attention
- **Persistent**: Email trail for audit
- **Searchable**: Easy to find historical alerts

**Use Cases**:
- **Warning Alerts**: Degraded performance, capacity warnings
- **Daily Summaries**: Alert summary reports
- **Escalation Notifications**: When alerts escalate to next level
- **Post-Incident Reports**: Incident summaries

**Email Template**:

```typescript
// Email notification service
class EmailNotificationService {
  async sendAlert(alert: Alert): Promise<void> {
    const email = {
      to: alert.recipients,
      subject: `[${alert.severity.toUpperCase()}] ${alert.title}`,
      html: this.renderAlertEmail(alert),
      attachments: [
        {
          filename: 'metrics.png',
          content: await this.generateMetricsGraph(alert)
        }
      ]
    };
    
    await emailQueue.add(email, {
      priority: alert.severity === 'critical' ? 1 : 5
    });
  }
  
  private renderAlertEmail(alert: Alert): string {
    return `
      <h2>${alert.title}</h2>
      <p><strong>Severity:</strong> ${alert.severity}</p>
      <p><strong>Time:</strong> ${alert.timestamp}</p>
      <p><strong>Description:</strong> ${alert.description}</p>
      
      <h3>Metrics</h3>
      <img src="cid:metrics.png" alt="Metrics Graph" />
      
      <h3>Actions</h3>
      <ul>
        <li><a href="${alert.dashboardUrl}">View Dashboard</a></li>
        <li><a href="${alert.runbookUrl}">View Runbook</a></li>
        <li><a href="${alert.acknowledgeUrl}">Acknowledge Alert</a></li>
      </ul>
      
      <h3>Recent Logs</h3>
      <pre>${alert.recentLogs}</pre>
    `;
  }
}
```

**When to Use Email**:
- ✅ Warning-level alerts
- ✅ Daily/weekly summaries
- ✅ Escalation notifications (after SMS/WhatsApp)
- ✅ Post-incident reports
- ❌ Critical alerts (too slow)
- ❌ Time-sensitive incidents

---

### 4. WhatsApp/SMS (Urgent Alerts)

**Purpose**: Immediate notification for critical incidents

**Characteristics**:
- **Instant**: Delivered within seconds
- **Mobile**: Always with the on-call engineer
- **Reliable**: Works even when email is down
- **Concise**: Short, actionable messages

**Use Cases**:
- **Critical Alerts**: Service down, data loss, security breach
- **Escalation**: When email/Slack not acknowledged
- **On-Call Rotation**: Shift handoff notifications
- **Emergency**: Manual escalation requests

**SMS/WhatsApp Template**:

```typescript
// SMS/WhatsApp notification service
class SMSNotificationService {
  async sendCriticalAlert(alert: Alert): Promise<void> {
    const message = this.formatAlertMessage(alert);
    
    // Try WhatsApp first (free, more reliable)
    try {
      await this.sendWhatsApp(alert.onCallPhone, message);
    } catch (error) {
      // Fallback to SMS
      await this.sendSMS(alert.onCallPhone, message);
    }
  }
  
  private formatAlertMessage(alert: Alert): string {
    // Keep under 160 characters for SMS
    const emoji = {
      critical: '🚨',
      warning: '⚠️',
      info: 'ℹ️'
    }[alert.severity];
    
    return `${emoji} ${alert.title}
${alert.severity.toUpperCase()}

${alert.shortDescription}

Dashboard: ${alert.shortUrl}
Ack: ${alert.acknowledgeUrl}`;
  }
  
  private async sendWhatsApp(phone: string, message: string): Promise<void> {
    // Using Twilio WhatsApp API or similar
    await twilio.messages.create({
      from: 'whatsapp:+14155238886',
      to: `whatsapp:${phone}`,
      body: message
    });
  }
  
  private async sendSMS(phone: string, message: string): Promise<void> {
    await twilio.messages.create({
      from: '+1234567890',
      to: phone,
      body: message
    });
  }
}
```

**When to Use SMS/WhatsApp**:
- ✅ Critical alerts (P0/P1)
- ✅ Service down alerts
- ✅ Security incidents
- ✅ Data loss alerts
- ✅ Escalation after no response to email
- ❌ Warning alerts (too noisy)
- ❌ Informational alerts

**Rate Limiting**:
- Max 3 SMS/WhatsApp per hour per alert
- Group related alerts to prevent spam
- Escalate to next level if no acknowledgment

---

## Escalation Levels & Timing

### Severity Levels

| Level | Description | Response Time | Channels |
|-------|-------------|--------------|----------|
| **P0 - Critical** | Service down, data loss, security breach | Immediate (< 5 min) | SMS/WA → Email → GUI |
| **P1 - High** | Major degradation, partial outage | 15 minutes | Email → SMS/WA → GUI |
| **P2 - Medium** | Performance issues, non-critical errors | 1 hour | Email → GUI |
| **P3 - Low** | Minor issues, capacity warnings | 4 hours | Email only |
| **P4 - Info** | Informational, maintenance notices | Next business day | Email only |

### Escalation Timeline

```
T+0:00  Alert Detected
        ↓
T+0:01  [P0] SMS/WhatsApp sent to on-call
        [P1] Email sent to on-call
        [P2+] Email sent, logged in GUI
        ↓
T+0:05  [P0] If not acknowledged → Escalate to L2
        ↓
T+0:15  [P1] If not acknowledged → SMS/WhatsApp to on-call
        [P0] If not resolved → Escalate to L2
        ↓
T+0:30  [P0] If not resolved → Escalate to L3 (Manager)
        ↓
T+1:00  [P1] If not resolved → Escalate to L2
        [P0] If not resolved → Escalate to L4 (CTO)
```

### Escalation Path

```yaml
escalation_path:
  level_1:
    role: "On-Call Engineer"
    contact: "Primary on-call rotation"
    channels: ["SMS/WhatsApp", "Email", "Slack"]
    response_time: "5 minutes"
    
  level_2:
    role: "Tech Lead"
    contact: "@tech-lead"
    channels: ["SMS/WhatsApp", "Email"]
    escalation_trigger: "No response after 15 minutes (P0) or 1 hour (P1)"
    
  level_3:
    role: "Engineering Manager"
    contact: "@eng-manager"
    channels: ["SMS/WhatsApp", "Email"]
    escalation_trigger: "No resolution after 30 minutes (P0)"
    
  level_4:
    role: "CTO"
    contact: "@cto"
    channels: ["SMS/WhatsApp"]
    escalation_trigger: "Critical incident unresolved after 1 hour"
```

---

## Channel Selection Matrix

### When to Use Each Channel

| Alert Type | Logs | GUI Grid | Email | SMS/WhatsApp |
|------------|------|----------|-------|--------------|
| **P0 - Critical** | ✅ Detection | ✅ Dashboard | ✅ Details | ✅ Immediate |
| **P1 - High** | ✅ Detection | ✅ Dashboard | ✅ Primary | ⚠️ Escalation |
| **P2 - Medium** | ✅ Detection | ✅ Dashboard | ✅ Primary | ❌ |
| **P3 - Low** | ✅ Detection | ✅ Dashboard | ✅ Primary | ❌ |
| **Daily Summary** | ✅ | ✅ | ✅ | ❌ |
| **Investigation** | ✅ Primary | ✅ Visualization | ⚠️ Reports | ❌ |

### Channel Priority by Severity

**P0 - Critical**:
1. **SMS/WhatsApp** (immediate)
2. **GUI Grid** (acknowledgment)
3. **Email** (detailed context)
4. **Logs** (investigation)

**P1 - High**:
1. **Email** (primary notification)
2. **GUI Grid** (monitoring)
3. **Logs** (investigation)
4. **SMS/WhatsApp** (if not acknowledged)

**P2/P3 - Medium/Low**:
1. **Email** (notification)
2. **GUI Grid** (monitoring)
3. **Logs** (investigation)

---

## Implementation Patterns

### Pattern 1: Multi-Channel Alert Router

```typescript
// Alert routing service
class AlertRouter {
  constructor(
    private emailService: EmailNotificationService,
    private smsService: SMSNotificationService,
    private dashboardService: DashboardService,
    private logService: LogService
  ) {}
  
  async routeAlert(alert: Alert): Promise<void> {
    // Always log the alert
    await this.logService.logAlert(alert);
    
    // Always show in dashboard
    await this.dashboardService.addAlert(alert);
    
    // Route based on severity
    switch (alert.severity) {
      case 'critical':
        // P0: All channels immediately
        await Promise.all([
          this.smsService.sendCriticalAlert(alert),
          this.emailService.sendAlert(alert),
          this.dashboardService.highlightAlert(alert)
        ]);
        break;
        
      case 'warning':
        // P1: Email first, SMS if not acknowledged
        await this.emailService.sendAlert(alert);
        // Schedule SMS escalation if not acknowledged in 15 min
        setTimeout(async () => {
          if (!await this.dashboardService.isAcknowledged(alert.id)) {
            await this.smsService.sendEscalation(alert);
          }
        }, 15 * 60 * 1000);
        break;
        
      case 'info':
        // P2/P3: Email only
        await this.emailService.sendAlert(alert);
        break;
    }
  }
}
```

### Pattern 2: Alert Aggregation

```typescript
// Prevent alert spam by grouping similar alerts
class AlertAggregator {
  private alertGroups: Map<string, Alert[]> = new Map();
  
  async aggregateAlert(alert: Alert): Promise<void> {
    const groupKey = `${alert.service}-${alert.type}`;
    const group = this.alertGroups.get(groupKey) || [];
    
    group.push(alert);
    this.alertGroups.set(groupKey, group);
    
    // Send aggregated alert if threshold reached
    if (group.length >= 5) {
      await this.sendAggregatedAlert(group);
      this.alertGroups.delete(groupKey);
    }
  }
  
  private async sendAggregatedAlert(alerts: Alert[]): Promise<void> {
    const summary = {
      count: alerts.length,
      firstOccurrence: alerts[0].timestamp,
      lastOccurrence: alerts[alerts.length - 1].timestamp,
      severity: alerts[0].severity,
      service: alerts[0].service
    };
    
    await this.router.routeAlert({
      ...alerts[0],
      title: `${summary.count} ${alerts[0].title} alerts`,
      description: `Multiple alerts detected. First: ${summary.firstOccurrence}, Last: ${summary.lastOccurrence}`
    });
  }
}
```

### Pattern 3: Escalation Timer

```typescript
// Automatic escalation if no response
class EscalationTimer {
  private timers: Map<string, NodeJS.Timeout> = new Map();
  
  startEscalationTimer(alert: Alert): void {
    const escalationTime = this.getEscalationTime(alert.severity);
    
    const timer = setTimeout(async () => {
      const isAcknowledged = await dashboardService.isAcknowledged(alert.id);
      const isResolved = await dashboardService.isResolved(alert.id);
      
      if (!isAcknowledged && !isResolved) {
        await this.escalate(alert);
      }
      
      this.timers.delete(alert.id);
    }, escalationTime);
    
    this.timers.set(alert.id, timer);
  }
  
  cancelEscalationTimer(alertId: string): void {
    const timer = this.timers.get(alertId);
    if (timer) {
      clearTimeout(timer);
      this.timers.delete(alertId);
    }
  }
  
  private getEscalationTime(severity: string): number {
    const times = {
      critical: 5 * 60 * 1000,    // 5 minutes
      warning: 15 * 60 * 1000,    // 15 minutes
      info: 60 * 60 * 1000        // 1 hour
    };
    return times[severity] || 60 * 60 * 1000;
  }
  
  private async escalate(alert: Alert): Promise<void> {
    const nextLevel = this.getNextEscalationLevel(alert.currentLevel);
    await this.router.routeAlert({
      ...alert,
      currentLevel: nextLevel,
      escalated: true,
      escalationReason: 'No response from previous level'
    });
  }
}
```

---

## Integration Examples

### Prometheus Alertmanager Configuration

```yaml
# alertmanager.yml
route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  receiver: 'default'
  
  routes:
    # Critical alerts → SMS/WhatsApp + Email + Dashboard
    - match:
        severity: critical
      receiver: 'critical-alerts'
      continue: true
      
    # Warning alerts → Email + Dashboard
    - match:
        severity: warning
      receiver: 'warning-alerts'
      
    # Info alerts → Email only
    - match:
        severity: info
      receiver: 'info-alerts'

receivers:
  - name: 'critical-alerts'
    # SMS/WhatsApp (immediate)
    webhook_configs:
      - url: 'http://notification-service:3000/webhook/sms'
        send_resolved: false
        
    # Email (detailed)
    email_configs:
      - to: 'oncall@company.com'
        headers:
          Subject: '🚨 CRITICAL: {{ .GroupLabels.alertname }}'
        html: |
          <h2>{{ .GroupLabels.alertname }}</h2>
          <p><strong>Severity:</strong> {{ .CommonLabels.severity }}</p>
          <p><strong>Time:</strong> {{ .StartsAt }}</p>
          <p><strong>Description:</strong> {{ .CommonAnnotations.description }}</p>
          <p><a href="{{ .ExternalURL }}">View in Dashboard</a></p>
          
    # Dashboard (via webhook)
    webhook_configs:
      - url: 'http://dashboard-service:3000/api/alerts'
        
  - name: 'warning-alerts'
    email_configs:
      - to: 'oncall@company.com'
        headers:
          Subject: '⚠️ WARNING: {{ .GroupLabels.alertname }}'
          
    webhook_configs:
      - url: 'http://dashboard-service:3000/api/alerts'
        
  - name: 'info-alerts'
    email_configs:
      - to: 'alerts@company.com'
        headers:
          Subject: 'ℹ️ INFO: {{ .GroupLabels.alertname }}'
```

### Log-Based Alert Detection

```typescript
// Log pattern detection
import { logger } from './logger';

class LogAlertDetector {
  private errorCounts: Map<string, number> = new Map();
  private readonly ERROR_THRESHOLD = 10; // errors per minute
  
  constructor() {
    // Monitor log stream
    logger.on('log', (logEntry) => {
      if (logEntry.level === 'error') {
        this.trackError(logEntry);
      }
    });
  }
  
  private trackError(logEntry: LogEntry): void {
    const key = `${logEntry.service}-${logEntry.endpoint}`;
    const count = (this.errorCounts.get(key) || 0) + 1;
    this.errorCounts.set(key, count);
    
    // Check threshold
    if (count >= this.ERROR_THRESHOLD) {
      this.triggerAlert({
        severity: 'critical',
        source: 'logs',
        title: `High error rate: ${logEntry.endpoint}`,
        description: `${count} errors in the last minute`,
        service: logEntry.service,
        endpoint: logEntry.endpoint,
        traceId: logEntry.traceId
      });
      
      // Reset counter
      this.errorCounts.delete(key);
    }
  }
  
  private async triggerAlert(alert: Alert): Promise<void> {
    await alertRouter.routeAlert(alert);
  }
}
```

### Dashboard Integration

```typescript
// Dashboard alert display
class DashboardAlertService {
  async displayAlert(alert: Alert): Promise<void> {
    // Add to alert list
    await this.addToAlertList(alert);
    
    // Highlight if critical
    if (alert.severity === 'critical') {
      await this.highlightAlert(alert);
      await this.playSound('critical-alert.mp3');
    }
    
    // Show notification banner
    await this.showNotificationBanner(alert);
  }
  
  async acknowledgeAlert(alertId: string, userId: string): Promise<void> {
    await this.updateAlertStatus(alertId, {
      acknowledged: true,
      acknowledgedBy: userId,
      acknowledgedAt: new Date()
    });
    
    // Cancel escalation timer
    escalationTimer.cancelEscalationTimer(alertId);
  }
}
```

---

## Best Practices

### 1. Alert Fatigue Prevention

- **Group similar alerts**: Don't send 100 emails for the same issue
- **Rate limiting**: Max 1 SMS per hour per alert type
- **Quiet hours**: Reduce SMS/WhatsApp during off-hours (use email)
- **Alert suppression**: Suppress alerts during maintenance windows

### 2. Channel Reliability

- **Redundancy**: Critical alerts use multiple channels
- **Fallback**: If SMS fails, try WhatsApp, then email
- **Health checks**: Monitor notification service health
- **Testing**: Regularly test all notification channels

### 3. Context-Rich Alerts

- **Include links**: Dashboard URL, runbook URL, log search URL
- **Attach graphs**: Include relevant metrics graphs
- **Recent logs**: Show last 5-10 relevant log entries
- **Action items**: Clear next steps in alert message

### 4. Acknowledgment Workflow

- **Easy acknowledgment**: One-click in dashboard or reply to SMS
- **Status visibility**: Team can see who's handling what
- **Escalation prevention**: Acknowledgment stops escalation timer
- **Handoff**: Easy to transfer alert to another engineer

### 5. Post-Incident

- **Alert review**: Review all alerts that fired during incident
- **Tune thresholds**: Adjust if too sensitive or not sensitive enough
- **Update runbooks**: Add lessons learned to runbooks
- **Channel optimization**: Remove channels that didn't help

---

## See Also

- [DevOps Overview](devops.md) - On-call responsibilities and escalation paths
- [Prometheus](prometheus.md) - Metrics and alerting setup
- [Logging](logging.md) - Log-based alert detection
- [Runbooks](runbooks/) - Incident response procedures
- [Architecture Practices](architecture-practices.md) - Service ownership and escalation

---

**Last Reviewed**: 2026-01-28  
**Next Review**: 2026-04-28 (Quarterly)
