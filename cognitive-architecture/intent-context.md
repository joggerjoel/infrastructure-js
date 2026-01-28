# Intent & Context Modeling Layer

---
**Status**: 📋 **PLANNED** - Foundational layer for cognitive navigation  
**Priority**: 🔴 **P0** - Required before AI orchestration can function  
**Last Updated**: 2026-01-28  
**Owned By**: Frontend Team, AI Team  

---

## Overview

The Intent & Context Modeling Layer creates a **first-class representation of user intent and system context** that persists across interactions. This is the foundation that enables AI to make intelligent navigation decisions without guessing.

**Critical Principle**: This layer is **deterministic and auditable**. AI reads it—humans define it.

---

## What This Is NOT

- ❌ Routing (URL state management)
- ❌ Redux alone (generic state management)
- ❌ Analytics events (passive tracking)
- ❌ Session storage (temporary data)

## What This IS

- ✅ A living context object updated continuously
- ✅ First-class intent representation
- ✅ Eligibility signals for graph nodes
- ✅ Priority weights for decision-making
- ✅ Suppression rules for noise reduction

---

## Core Data Structure

### UserContext

```typescript
interface UserContext {
  // Intent Vector: What the user is trying to accomplish
  intentVector: Intent[]
  
  // Confidence: How certain we are about current intent
  confidence: number  // 0.0 to 1.0
  
  // Recent Interactions: What the user has done recently
  recentInteractions: Interaction[]
  
  // Suppressed Topics: What the user has explicitly dismissed
  suppressedTopics: Topic[]
  
  // Attention Budget: How much cognitive load the user can handle
  attentionBudget: number  // 0 to 100
  
  // System State: Current system signals and alerts
  systemState: SystemSignals
  
  // Session Metadata
  sessionId: string
  startTime: Date
  lastActivity: Date
}
```

### Intent

```typescript
interface Intent {
  id: string
  type: IntentType
  target?: string  // What entity this intent relates to
  confidence: number
  timestamp: Date
  source: IntentSource
}

enum IntentType {
  INVESTIGATE = "investigate",
  CONFIGURE = "configure",
  MONITOR = "monitor",
  TROUBLESHOOT = "troubleshoot",
  EXPLORE = "explore",
  REVIEW = "review"
}

enum IntentSource {
  EXPLICIT = "explicit",      // User clicked/navigated
  INFERRED = "inferred",      // System inferred from behavior
  SYSTEM = "system",          // System-initiated (alert, etc.)
  AI = "ai"                   // AI suggested
}
```

### Interaction

```typescript
interface Interaction {
  id: string
  type: InteractionType
  nodeId?: string
  timestamp: Date
  duration?: number  // milliseconds
  outcome?: InteractionOutcome
  metadata?: Record<string, any>
}

enum InteractionType {
  CLICK = "click",
  SCROLL = "scroll",
  INPUT = "input",
  DISMISS = "dismiss",
  ACCEPT = "accept",
  NAVIGATE = "navigate",
  DWELL = "dwell"  // Time spent on a node
}

enum InteractionOutcome {
  COMPLETED = "completed",
  ABANDONED = "abandoned",
  DEFERRED = "deferred",
  REJECTED = "rejected"
}
```

### SystemSignals

```typescript
interface SystemSignals {
  // Active alerts and their severity
  urgentAlerts: Alert[]
  
  // System health indicators
  healthMetrics: HealthMetric[]
  
  // Active incidents
  activeIncidents: Incident[]
  
  // Recent changes
  recentChanges: Change[]
  
  // User-specific flags
  userFlags: Record<string, boolean>
}

interface Alert {
  id: string
  severity: "critical" | "warning" | "info"
  message: string
  timestamp: Date
  relatedEntity?: string
}
```

---

## Context Update Mechanisms

### 1. Explicit User Actions

When a user clicks, navigates, or provides input:

```typescript
class ContextManager {
  updateFromUserAction(action: UserAction): void {
    // Record the interaction
    this.context.recentInteractions.push({
      id: generateId(),
      type: action.type,
      nodeId: action.nodeId,
      timestamp: new Date(),
      metadata: action.metadata
    })
    
    // Update intent vector
    const inferredIntent = this.inferIntent(action)
    if (inferredIntent) {
      this.context.intentVector.push(inferredIntent)
    }
    
    // Update confidence
    this.context.confidence = this.calculateConfidence()
    
    // Update attention budget
    this.context.attentionBudget = Math.max(
      0,
      this.context.attentionBudget - action.attentionCost
    )
    
    // Update last activity
    this.context.lastActivity = new Date()
  }
}
```

### 2. System Events

When alerts fire, incidents occur, or system state changes:

```typescript
class ContextManager {
  updateFromSystemEvent(event: SystemEvent): void {
    // Add to system signals
    if (event.type === "alert") {
      this.context.systemState.urgentAlerts.push(event.alert)
      
      // Create system-initiated intent
      this.context.intentVector.push({
        id: generateId(),
        type: IntentType.INVESTIGATE,
        target: event.alert.relatedEntity,
        confidence: event.alert.severity === "critical" ? 0.95 : 0.7,
        timestamp: new Date(),
        source: IntentSource.SYSTEM
      })
    }
  }
}
```

### 3. AI Suggestions

When AI recommends a course of action:

```typescript
class ContextManager {
  updateFromAISuggestion(suggestion: AISuggestion): void {
    // Add AI-inferred intent
    this.context.intentVector.push({
      id: generateId(),
      type: suggestion.intentType,
      target: suggestion.target,
      confidence: suggestion.confidence,
      timestamp: new Date(),
      source: IntentSource.AI
    })
  }
}
```

### 4. Time-Based Decay

Context decays over time to prevent stale information:

```typescript
class ContextManager {
  tick(): void {
    const now = new Date()
    
    // Decay old intents
    this.context.intentVector = this.context.intentVector.filter(intent => {
      const age = now.getTime() - intent.timestamp.getTime()
      return age < 5 * 60 * 1000  // Keep intents for 5 minutes
    })
    
    // Decay attention budget (recovers over time)
    this.context.attentionBudget = Math.min(
      100,
      this.context.attentionBudget + 1  // Recover 1 point per tick
    )
    
    // Prune old interactions
    this.context.recentInteractions = this.context.recentInteractions.filter(interaction => {
      const age = now.getTime() - interaction.timestamp.getTime()
      return age < 10 * 60 * 1000  // Keep interactions for 10 minutes
    })
  }
}
```

---

## Intent Inference

### Heuristics for Intent Detection

```typescript
class IntentInferenceEngine {
  inferIntent(action: UserAction): Intent | null {
    // Pattern: User clicked on an alert
    if (action.type === InteractionType.CLICK && action.metadata?.alertId) {
      return {
        id: generateId(),
        type: IntentType.INVESTIGATE,
        target: action.metadata.alertId,
        confidence: 0.85,
        timestamp: new Date(),
        source: IntentSource.INFERRED
      }
    }
    
    // Pattern: User spent >30s on a configuration page
    if (action.type === InteractionType.DWELL && action.duration > 30000) {
      return {
        id: generateId(),
        type: IntentType.CONFIGURE,
        target: action.nodeId,
        confidence: 0.7,
        timestamp: new Date(),
        source: IntentSource.INFERRED
      }
    }
    
    // Pattern: User dismissed a suggestion
    if (action.type === InteractionType.DISMISS) {
      // Add to suppressed topics
      this.context.suppressedTopics.push({
        id: action.nodeId,
        suppressedAt: new Date(),
        reason: "user_dismissed"
      })
      return null
    }
    
    return null
  }
}
```

---

## Suppression Rules

### Topic Suppression

```typescript
interface Topic {
  id: string
  suppressedAt: Date
  reason: string
  expiresAt?: Date
}

class SuppressionManager {
  isSuppressed(topicId: string): boolean {
    const topic = this.context.suppressedTopics.find(t => t.id === topicId)
    if (!topic) return false
    
    // Check if suppression has expired
    if (topic.expiresAt && new Date() > topic.expiresAt) {
      this.removeSuppression(topicId)
      return false
    }
    
    return true
  }
  
  suppressTopic(topicId: string, duration?: number): void {
    this.context.suppressedTopics.push({
      id: topicId,
      suppressedAt: new Date(),
      reason: "user_dismissed",
      expiresAt: duration ? new Date(Date.now() + duration) : undefined
    })
  }
}
```

---

## Confidence Calculation

```typescript
class ConfidenceCalculator {
  calculateConfidence(context: UserContext): number {
    let confidence = 0.5  // Start at neutral
    
    // Boost confidence if we have explicit intents
    const explicitIntents = context.intentVector.filter(
      i => i.source === IntentSource.EXPLICIT
    )
    confidence += explicitIntents.length * 0.1
    
    // Boost confidence if recent interactions are consistent
    const recentTypes = context.recentInteractions
      .slice(-5)
      .map(i => i.type)
    const uniqueTypes = new Set(recentTypes).size
    if (uniqueTypes === 1) {
      confidence += 0.2  // Consistent behavior
    }
    
    // Reduce confidence if user is bouncing around
    const navigationCount = context.recentInteractions.filter(
      i => i.type === InteractionType.NAVIGATE
    ).length
    if (navigationCount > 5) {
      confidence -= 0.2  // User seems lost
    }
    
    // Clamp to [0, 1]
    return Math.max(0, Math.min(1, confidence))
  }
}
```

---

## Integration with Graph Engine

The context is used by the Directed Graph Engine to evaluate node eligibility:

```typescript
interface GraphNode {
  id: string
  label: string
  
  // Eligibility based on context
  entryConditions?: (context: UserContext) => boolean
  
  // Priority based on context
  priorityRules?: (context: UserContext) => number
  
  // Suppression based on context
  suppressIf?: (context: UserContext) => boolean
}

// Example node
const incidentInvestigationNode: GraphNode = {
  id: "incident_investigation",
  label: "Investigate Incident",
  
  entryConditions: (ctx) => {
    // Only show if there's an active incident
    return ctx.systemState.activeIncidents.length > 0
  },
  
  priorityRules: (ctx) => {
    // Higher priority if user has investigation intent
    const hasInvestigateIntent = ctx.intentVector.some(
      i => i.type === IntentType.INVESTIGATE
    )
    return hasInvestigateIntent ? 10 : 5
  },
  
  suppressIf: (ctx) => {
    // Suppress if user dismissed this topic
    return ctx.suppressedTopics.some(t => t.id === "incident_investigation")
  }
}
```

---

## Persistence & Hydration

### Save Context to Storage

```typescript
class ContextPersistence {
  saveContext(context: UserContext): void {
    localStorage.setItem('userContext', JSON.stringify({
      sessionId: context.sessionId,
      intentVector: context.intentVector,
      suppressedTopics: context.suppressedTopics,
      attentionBudget: context.attentionBudget,
      lastActivity: context.lastActivity
    }))
  }
  
  loadContext(): UserContext | null {
    const stored = localStorage.getItem('userContext')
    if (!stored) return null
    
    const data = JSON.parse(stored)
    
    // Check if session is still valid (< 1 hour old)
    const lastActivity = new Date(data.lastActivity)
    const age = Date.now() - lastActivity.getTime()
    if (age > 60 * 60 * 1000) {
      return null  // Session expired
    }
    
    return {
      ...data,
      recentInteractions: [],  // Don't persist interactions
      systemState: this.fetchCurrentSystemState()  // Refresh system state
    }
  }
}
```

---

## Testing & Debugging

### Context Inspector

```typescript
class ContextInspector {
  printContext(context: UserContext): void {
    console.group('User Context')
    console.log('Session ID:', context.sessionId)
    console.log('Confidence:', context.confidence.toFixed(2))
    console.log('Attention Budget:', context.attentionBudget)
    
    console.group('Intent Vector')
    context.intentVector.forEach(intent => {
      console.log(`- ${intent.type} (${intent.confidence.toFixed(2)}) [${intent.source}]`)
    })
    console.groupEnd()
    
    console.group('Recent Interactions')
    context.recentInteractions.slice(-5).forEach(interaction => {
      console.log(`- ${interaction.type} on ${interaction.nodeId}`)
    })
    console.groupEnd()
    
    console.group('Suppressed Topics')
    context.suppressedTopics.forEach(topic => {
      console.log(`- ${topic.id} (${topic.reason})`)
    })
    console.groupEnd()
    
    console.groupEnd()
  }
}
```

---

## Best Practices

1. **Keep Context Lean**: Don't store everything—only what's needed for decision-making
2. **Decay Old Data**: Prune stale intents and interactions regularly
3. **Validate Confidence**: Always calculate confidence based on observable signals
4. **Respect Suppression**: Once a user dismisses something, honor that choice
5. **Log Context Changes**: Every context update should be logged for debugging

---

## Next Steps

1. Implement `UserContext` interface in TypeScript
2. Create `ContextManager` class with update methods
3. Build `IntentInferenceEngine` with heuristics
4. Integrate with Directed Graph Engine (see `directed-graph-engine.md`)
5. Add context inspector to dev tools
