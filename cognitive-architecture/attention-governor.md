# Attention & Noise Governance

---
**Status**: 📋 **PLANNED** - System-level governor for cognitive load  
**Priority**: 🟡 **P1** - Highly important for user experience  
**Last Updated**: 2026-01-28  
**Owned By**: Frontend Team, AI Team  

---

## Overview

The Attention Governor prevents the cognitive system from overwhelming the user. It manages interruptions, alerts, and suggestions based on a real-time "Attention Budget."

**Key Principle**: The system learns restraint. Intelligent behavior requires knowing when to **not** speak.

---

## The Attention Budget

A dynamic metric representing the user's current capacity for new information or interruptions.

### Configuration

```typescript
interface AttentionConfig {
  initialBudget: 100;
  interruptThreshold: 30; // Min budget required for a non-urgent interrupt
  alertThreshold: 10;     // Min budget required for any non-critical alert
  decayRate: number;      // Budget lost per minute of active interaction
  recoveryRate: number;   // Budget gained per minute of dwell/idle time
}
```

### State Persistence

```typescript
interface AttentionGovernor {
  budget: number;          // Current points (0-100)
  lastUpdate: number;      // Timestamp
  activeCooldowns: Map<string, number>; // nodeID -> expiresAt
}
```

---

## Governance Rules

### 1. Interruption Priority

The AI Orchestrator must consult the Governor before proposing a node switch.

```typescript
function canSystemInterrupt(nodePriority: number, intentConfidence: number): boolean {
  // Critical system alerts always bypass
  if (nodePriority >= 10) return true;

  // Otherwise, check budget
  if (governor.budget < config.interruptThreshold) {
    return false;
  }

  // Ensure high confidence if budget is low
  if (governor.budget < 50 && intentConfidence < 0.8) {
    return false;
  }

  return true;
}
```

### 2. Delay & Decay

| User State | Budget Change | Logic |
|------------|---------------|-------|
| **Active Typing** | -10 per min | High concentration, no interruptions. |
| **Rapid Clicking** | -20 per min | Frantic state, minimize noise. |
| **Dwell (Reading)** | +10 per min | Calm state, budget recovers. |
| **Idle** | +15 per min | System resets to base attentiveness. |

---

## Implementation Layer

The Governor acts as a middleware between the **Directed Graph Engine** and the **Story Rendering Layer**.

```typescript
class AttentionGovernorService {
  private budget: number = 100;

  evaluate(proposal: NodeProposal): boolean {
    const cost = proposal.node.uiContract.attentionCost || 10;
    
    if (this.budget - cost < 0) {
      this.defer(proposal);
      return false;
    }

    this.consume(cost);
    return true;
  }

  private defer(proposal: NodeProposal) {
    // Add to a "queued suggestions" tray rather than disrupting the flow
    trayService.queue(proposal);
  }
}
```

---

## Noise Reduction Strategies

1. **Alert Aggregation**: Instead of 5 alerts, show one "Alert Pulse" that the user can dive into when ready.
2. **Contextual Cooldowns**: If a user dismisses a node type, put that specific intent on long-term cooldown (24h+).
3. **Muted Transitions**: If the system changes state without user input, use extremely subtle transitions (opacity only) to avoid drawing the eye aggressively.

---

## Why This Matters

Without this layer, an AI-driven UI becomes "needy" (like a paperclip assistant). With governance, the system feels "intelligent" because it understands the value of the user's focus. 

The goal is to move from **Notification-Driven Development** to **Attention-Aware Architecture**.

---

## Next Steps

1. Implement the `AttentionGovernor` class in the frontend state manager.
2. Hook interaction events (scroll, click, keydown) into the budget decay logic.
3. Define "Urgency levels" for all automated alerts to ensure they interact correctly with the governor.
