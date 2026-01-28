# Auditability & Replay

---
**Status**: 📋 **PLANNED** - System for transparency and debugging  
**Priority**: 🟢 **P2** - Critical for trust and tuning  
**Last Updated**: 2026-01-28  
**Owned By**: AI Team, Reliability Engineering  

---

## Overview

If an AI controls navigation, the system must be able to answer: **"Why did the system show this?"** Auditability & Replay ensures that every transition is logged, explainable, and reproducible.

**Key Principle**: Trust is built through transparency, not perfection.

---

## Core Capabilities

1. **Decision Logs**: Structured records of why a specific node was chosen.
2. **Context Snapshots**: Frozen state of the `UserContext` at the moment of decision.
3. **Transition History**: A linear "flight recorder" of the user journey.
4. **AI Reasoning Storage**: Capturing the raw reasoning tags provided by the LLM.

---

## Data Schema

### Decision Registry

```typescript
interface DecisionEntry {
  decisionId: string;
  timestamp: number;
  activeNode: string;
  eligibleNodes: string[];
  selectedNode: string;
  userContext: UserContextSnapshot;
  aiOutput: {
    confidence: number;
    reasoning: string[];
  };
  outcome: "accepted" | "dismissed" | "timed_out";
}
```

---

## Implementation: The "Flight Recorder"

Every navigation event must pass through the Audit Service.

```typescript
class AuditService {
  logTransition(from: string, to: string, context: UserContext, reasoning: string[]) {
    const entry: DecisionEntry = {
      decisionId: uuid(),
      timestamp: Date.now(),
      activeNode: from,
      eligibleNodes: engine.getEligibleNodes(),
      selectedNode: to,
      userContext: this.snapshot(context),
      aiOutput: {
        confidence: ai.lastConfidence,
        reasoning: reasoning
      },
      outcome: "pending"
    };

    api.post("/audit/decision", entry);
  }
}
```

---

## The Replay Engine

For debugging "UI Hallucinations" or narrative drift, developers must be able to inject a historical context and see the same decision occur.

```typescript
async function debugDecision(decisionId: string) {
  const entry = await api.get(`/audit/decision/${decisionId}`);
  
  // Re-hydrate the local engine with the historical context
  engine.hydrate(entry.userContext);
  
  // Re-run the orchestrator with the same eligible nodes
  const result = await orchestrator.reevaluate(entry.eligibleNodes);
  
  console.log("Original Selection:", entry.selectedNode);
  console.log("Current Evaluation:", result.selectedNode);
}
```

---

## Human-in-the-Loop Tuning

Logged decisions are the primary training data for the system. 

1. **Human Review**: Humans tag "Bad Decisions" in the audit log.
2. **Preference Learning**: The system adjusts the `priorityRules` or `entryConditions` based on frequent user dismissals.
3. **Regression Testing**: Use past logs as test cases to ensure new AI models or graph changes don't break established flows.

---

## Visibility: The "Why" UI

In development mode (and potentially for power users), nodes should provide an explanation widget.

```tsx
const DecisionReasoning: React.FC = () => {
  const { auditData } = useDecision();
  
  return (
    <div className="dev-overlay text-xs opacity-50 hover:opacity-100">
      <span>AI Selected this because:</span>
      <ul>
        {auditData.reasoning.map(tag => (
          <li key={tag}>• {tag}</li>
        ))}
      </ul>
      <span>Confidence: {auditData.confidence}%</span>
    </div>
  );
};
```

---

## Next Steps

1. Setup the backend schema for `DecisionEntry` (MariaDB/JSON column).
2. Implement the `AuditService` on the frontend.
3. Create a "Developer Replay" view to step through past interactions.
4. Integrate reasoning tags into the main UI for "Dev/Power" mode.
