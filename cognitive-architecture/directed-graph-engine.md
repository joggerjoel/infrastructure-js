# Directed Graph Engine

---
**Status**: 📋 **PLANNED** - Core navigation engine for cognitive architecture  
**Priority**: 🔴 **P0** - Required for all cognitive navigation features  
**Last Updated**: 2026-01-28  
**Owned By**: Frontend Team  

---

## Overview

The Directed Graph Engine evaluates which nodes are eligible, enforces human-defined constraints, and exposes choices to AI—**never freedom**. This is the heart of the cognitive navigation system.

**Critical Rule**: AI may choose **BETWEEN** nodes, never invent nodes or edges.

---

## Core Principles

1. **Human-Authored Graph** - All nodes and edges are defined by developers in JSON
2. **Deterministic Evaluation** - Same context always produces same eligible nodes
3. **Explainable Decisions** - Every decision can be traced back to rules
4. **Replayable Sessions** - Can reconstruct past navigation from logs

---

## Graph Definition

### GraphNode Interface

```typescript
interface GraphNode {
  // Identity
  id: string
  label: string
  description?: string
  
  // Node Type & Behavior
  type: NodeType
  uiContract: UIContract
  
  // Eligibility Rules (human-defined)
  entryConditions?: (context: UserContext) => boolean
  exitConditions?: (context: UserContext) => boolean
  
  // Priority & Suppression (human-defined)
  priorityRules?: (context: UserContext) => number
  suppressIf?: (context: UserContext) => boolean
  
  // Navigation
  allowedNextNodes?: string[]  // Explicit edges
  branches?: Record<string, GraphNode[]>  // Branching paths
  
  // Metadata
  metadata?: Record<string, any>
}

enum NodeType {
  LABEL = "label",           // Display information
  INPUT = "input",           // Collect user input
  BRANCH = "branch",         // Decision point
  ACTION = "action",         // Execute an action
  COMPOSITE = "composite"    // Multiple sub-nodes
}
```

### UIContract Interface

```typescript
interface UIContract {
  // Focus type determines UI treatment
  focusType: "read" | "decide" | "act"
  
  // Maximum number of actions user can take
  maxActions: number
  
  // Can this node interrupt the user?
  interruptionAllowed: boolean
  
  // Should this node persist across sessions?
  persistence: "sticky" | "ephemeral"
  
  // Estimated time to complete (seconds)
  estimatedDuration?: number
}
```

---

## Graph Loading & Validation

### Load Graph from JSON

```typescript
class GraphLoader {
  async loadGraph(path: string): Promise<TaskGraph> {
    const response = await fetch(path)
    const data = await response.json()
    
    // Validate structure
    this.validateGraph(data)
    
    // Build node registry
    const nodes = new Map<string, GraphNode>()
    data.nodes.forEach(node => {
      nodes.set(node.id, node)
    })
    
    return {
      metadata: data.metadata,
      nodes: nodes,
      entryNode: data.metadata.entryNode
    }
  }
  
  validateGraph(data: any): void {
    // Check required fields
    if (!data.metadata) {
      throw new Error("Graph missing metadata")
    }
    if (!data.nodes || !Array.isArray(data.nodes)) {
      throw new Error("Graph missing nodes array")
    }
    
    // Validate each node
    data.nodes.forEach((node, index) => {
      if (!node.id) {
        throw new Error(`Node at index ${index} missing id`)
      }
      if (!node.type) {
        throw new Error(`Node ${node.id} missing type`)
      }
    })
    
    // Validate edges
    data.nodes.forEach(node => {
      if (node.allowedNextNodes) {
        node.allowedNextNodes.forEach(nextId => {
          const exists = data.nodes.some(n => n.id === nextId)
          if (!exists) {
            throw new Error(`Node ${node.id} references non-existent node ${nextId}`)
          }
        })
      }
    })
  }
}
```

---

## Graph Engine Implementation

### Core Engine Class

```typescript
class DirectedGraphEngine {
  private graph: TaskGraph
  private context: UserContext
  private currentNode: string
  private navigationHistory: NavigationStep[]
  
  constructor(graph: TaskGraph, context: UserContext) {
    this.graph = graph
    this.context = context
    this.currentNode = graph.entryNode
    this.navigationHistory = []
  }
  
  // Get current node
  getCurrentNode(): GraphNode {
    return this.graph.nodes.get(this.currentNode)!
  }
  
  // Evaluate if a node is eligible
  isEligible(node: GraphNode): boolean {
    // Check entry conditions
    if (node.entryConditions && !node.entryConditions(this.context)) {
      return false
    }
    
    // Check suppression
    if (node.suppressIf && node.suppressIf(this.context)) {
      return false
    }
    
    return true
  }
  
  // Get all eligible next nodes
  getEligibleNextNodes(): GraphNode[] {
    const current = this.getCurrentNode()
    
    // If current node has explicit edges, use those
    if (current.allowedNextNodes) {
      return current.allowedNextNodes
        .map(id => this.graph.nodes.get(id)!)
        .filter(node => this.isEligible(node))
    }
    
    // If current node has branches, evaluate branch conditions
    if (current.branches) {
      const eligibleBranches: GraphNode[] = []
      Object.entries(current.branches).forEach(([branchName, nodes]) => {
        // Check if this branch is eligible based on context
        const branchEligible = this.isBranchEligible(branchName, current)
        if (branchEligible) {
          eligibleBranches.push(...nodes.filter(n => this.isEligible(n)))
        }
      })
      return eligibleBranches
    }
    
    // Default: all nodes are potential next nodes (filtered by eligibility)
    return Array.from(this.graph.nodes.values())
      .filter(node => node.id !== this.currentNode)
      .filter(node => this.isEligible(node))
  }
  
  // Calculate priority for a node
  calculatePriority(node: GraphNode): number {
    if (node.priorityRules) {
      return node.priorityRules(this.context)
    }
    return 0  // Default priority
  }
  
  // Navigate to a node
  navigateTo(nodeId: string, reason: string): void {
    const node = this.graph.nodes.get(nodeId)
    if (!node) {
      throw new Error(`Node ${nodeId} not found`)
    }
    
    if (!this.isEligible(node)) {
      throw new Error(`Node ${nodeId} is not eligible`)
    }
    
    // Record navigation step
    this.navigationHistory.push({
      fromNode: this.currentNode,
      toNode: nodeId,
      timestamp: new Date(),
      reason: reason,
      contextSnapshot: structuredClone(this.context)
    })
    
    // Update current node
    this.currentNode = nodeId
  }
}
```

---

## Example Graph Definition

### Incident Investigation Graph

```json
{
  "metadata": {
    "name": "Incident Investigation Flow",
    "version": "1.0.0",
    "entryNode": "incident_triage"
  },
  "nodes": [
    {
      "id": "incident_triage",
      "label": "Incident Triage",
      "type": "branch",
      "uiContract": {
        "focusType": "decide",
        "maxActions": 3,
        "interruptionAllowed": true,
        "persistence": "sticky"
      },
      "branches": {
        "high_severity": [
          {
            "id": "immediate_investigation",
            "label": "Immediate Investigation",
            "type": "composite"
          }
        ],
        "medium_severity": [
          {
            "id": "scheduled_investigation",
            "label": "Scheduled Investigation",
            "type": "composite"
          }
        ],
        "low_severity": [
          {
            "id": "log_and_monitor",
            "label": "Log and Monitor",
            "type": "action"
          }
        ]
      }
    }
  ]
}
```

### With TypeScript Functions

```typescript
const incidentTriageNode: GraphNode = {
  id: "incident_triage",
  label: "Incident Triage",
  type: NodeType.BRANCH,
  
  uiContract: {
    focusType: "decide",
    maxActions: 3,
    interruptionAllowed: true,
    persistence: "sticky"
  },
  
  entryConditions: (ctx) => {
    // Only show if there are active incidents
    return ctx.systemState.activeIncidents.length > 0
  },
  
  priorityRules: (ctx) => {
    // Higher priority for critical incidents
    const hasCritical = ctx.systemState.urgentAlerts.some(
      a => a.severity === "critical"
    )
    return hasCritical ? 10 : 5
  },
  
  suppressIf: (ctx) => {
    // Suppress if user dismissed incident triage
    return ctx.suppressedTopics.some(t => t.id === "incident_triage")
  },
  
  branches: {
    high_severity: [immediateInvestigationNode],
    medium_severity: [scheduledInvestigationNode],
    low_severity: [logAndMonitorNode]
  }
}
```

---

## Branch Evaluation

### Determine Which Branch to Take

```typescript
class BranchEvaluator {
  evaluateBranch(
    branchNode: GraphNode,
    context: UserContext
  ): string | null {
    // Example: Severity-based branching
    if (branchNode.id === "incident_triage") {
      const incidents = context.systemState.activeIncidents
      if (incidents.some(i => i.severity === "critical")) {
        return "high_severity"
      }
      if (incidents.some(i => i.severity === "warning")) {
        return "medium_severity"
      }
      return "low_severity"
    }
    
    return null
  }
}
```

---

## Explainability

### Why Was This Node Shown?

```typescript
class ExplainabilityEngine {
  explainNodeSelection(
    node: GraphNode,
    context: UserContext
  ): Explanation {
    const reasons: string[] = []
    
    // Check entry conditions
    if (node.entryConditions) {
      const eligible = node.entryConditions(context)
      reasons.push(
        eligible
          ? "✅ Entry conditions met"
          : "❌ Entry conditions not met"
      )
    }
    
    // Check priority
    const priority = node.priorityRules?.(context) ?? 0
    reasons.push(`Priority: ${priority}`)
    
    // Check suppression
    const suppressed = node.suppressIf?.(context) ?? false
    if (suppressed) {
      reasons.push("❌ Suppressed by user")
    }
    
    // Check context signals
    if (context.intentVector.length > 0) {
      const topIntent = context.intentVector[0]
      reasons.push(`Top intent: ${topIntent.type}`)
    }
    
    return {
      nodeId: node.id,
      reasons: reasons,
      eligible: this.isEligible(node),
      priority: priority
    }
  }
}
```

---

## Replayability

### Reconstruct Session from History

```typescript
class SessionReplayer {
  async replay(sessionId: string): Promise<void> {
    // Load navigation history
    const history = await this.loadHistory(sessionId)
    
    // Reconstruct initial context
    let context = history[0].contextSnapshot
    
    // Step through each navigation
    for (const step of history) {
      console.log(`[${step.timestamp}] ${step.fromNode} → ${step.toNode}`)
      console.log(`Reason: ${step.reason}`)
      console.log(`Context:`, step.contextSnapshot)
      
      // Update context
      context = step.contextSnapshot
    }
  }
}
```

---

## Performance Optimization

### Node Caching

```typescript
class GraphEngine {
  private eligibilityCache: Map<string, boolean> = new Map()
  
  isEligible(node: GraphNode): boolean {
    // Check cache
    const cacheKey = `${node.id}_${this.context.sessionId}`
    if (this.eligibilityCache.has(cacheKey)) {
      return this.eligibilityCache.get(cacheKey)!
    }
    
    // Evaluate
    const eligible = this.evaluateEligibility(node)
    
    // Cache result (with TTL)
    this.eligibilityCache.set(cacheKey, eligible)
    setTimeout(() => {
      this.eligibilityCache.delete(cacheKey)
    }, 5000)  // Cache for 5 seconds
    
    return eligible
  }
}
```

---

## Testing

### Unit Tests for Graph Engine

```typescript
describe('DirectedGraphEngine', () => {
  it('should return eligible nodes based on context', () => {
    const graph = createTestGraph()
    const context = createTestContext({
      systemState: {
        activeIncidents: [{ severity: 'critical' }]
      }
    })
    
    const engine = new DirectedGraphEngine(graph, context)
    const eligible = engine.getEligibleNextNodes()
    
    expect(eligible).toContainEqual(
      expect.objectContaining({ id: 'incident_triage' })
    )
  })
  
  it('should respect suppression rules', () => {
    const context = createTestContext({
      suppressedTopics: [{ id: 'incident_triage' }]
    })
    
    const engine = new DirectedGraphEngine(graph, context)
    const node = graph.nodes.get('incident_triage')!
    
    expect(engine.isEligible(node)).toBe(false)
  })
})
```

---

## Integration with AI Orchestration

The graph engine provides eligible nodes to the AI orchestration layer:

```typescript
// Graph engine determines WHAT is possible
const eligibleNodes = engine.getEligibleNextNodes()

// AI orchestration decides WHICH to show
const selection = await aiOrchestrator.selectNode(
  eligibleNodes,
  context
)

// Graph engine executes the navigation
engine.navigateTo(selection.selected_node, "ai_selected")
```

See [ai-orchestration.md](ai-orchestration.md) for details.

---

## Best Practices

1. **Keep Rules Pure** - Entry/priority/suppression functions should be pure (no side effects)
2. **Validate Graphs** - Always validate graph structure on load
3. **Log Decisions** - Record every navigation decision for debugging
4. **Cache Sparingly** - Only cache expensive computations, invalidate aggressively
5. **Test Edge Cases** - Test with empty contexts, missing nodes, circular references

---

## Next Steps

1. Implement `DirectedGraphEngine` class
2. Create graph validation utilities
3. Build explainability engine
4. Add session replay functionality
5. Integrate with AI Orchestration Layer
