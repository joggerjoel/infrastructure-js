# AI Orchestration Layer

---
**Status**: 📋 **PLANNED** - AI decision-making layer for node selection  
**Priority**: 🔴 **P0.5** - Required for intelligent navigation  
**Last Updated**: 2026-01-28  
**Owned By**: Frontend Team, AI Team  

---

## Overview

The AI Orchestration Layer decides **WHICH** node to show from the eligible options provided by the Directed Graph Engine. It does **NOT** generate UI, decide business rules, or control flow directly.

**Critical Principle**: AI makes **structured decisions**, not free-form generations.

---

## What AI Should NOT Do

- ❌ Render UI components
- ❌ Decide business rules
- ❌ Control application flow directly
- ❌ Invent new nodes or edges
- ❌ Generate free-text UI instructions

## What AI SHOULD Do

- ✅ Rank eligible nodes by relevance
- ✅ Predict user friction points
- ✅ Optimize for clarity and focus
- ✅ Decide when to interrupt vs. defer
- ✅ Provide structured reasoning

---

## AI Decision Interface

### Input to AI

```typescript
interface AIDecisionRequest {
  // Current user context
  context_snapshot: UserContext
  
  // Nodes that are eligible (from graph engine)
  eligible_nodes: EligibleNode[]
  
  // Human-defined constraints
  constraints: DecisionConstraints
  
  // Objective for this decision
  objective: string
}

interface EligibleNode {
  id: string
  label: string
  type: NodeType
  priority: number  // From graph engine
  uiContract: UIContract
  metadata?: Record<string, any>
}

interface DecisionConstraints {
  // Maximum nodes to show at once
  maxNodes: number
  
  // Can we interrupt the user?
  allowInterruption: boolean
  
  // Minimum confidence threshold
  minConfidence: number
  
  // Attention budget available
  attentionBudget: number
}
```

### Output from AI

```typescript
interface AIDecisionResponse {
  // Selected node ID
  selected_node: string
  
  // Confidence in this decision (0.0 to 1.0)
  confidence: number
  
  // Structured reasoning (tags, not prose)
  reasoning_tags: string[]
  
  // Nodes to defer (show later)
  defer_others: string[]
  
  // Estimated attention cost
  attention_cost?: number
}
```

---

## Example AI Call

### Request

```json
{
  "context_snapshot": {
    "intentVector": [
      {
        "type": "investigate",
        "target": "incident_123",
        "confidence": 0.85,
        "source": "system"
      }
    ],
    "confidence": 0.82,
    "attentionBudget": 65,
    "systemState": {
      "urgentAlerts": [
        {
          "severity": "critical",
          "message": "Database connection pool exhausted"
        }
      ]
    }
  },
  "eligible_nodes": [
    {
      "id": "incident_investigation",
      "label": "Investigate Incident",
      "type": "composite",
      "priority": 10,
      "uiContract": {
        "focusType": "act",
        "maxActions": 3
      }
    },
    {
      "id": "billing_summary",
      "label": "Billing Summary",
      "type": "label",
      "priority": 3,
      "uiContract": {
        "focusType": "read",
        "maxActions": 0
      }
    },
    {
      "id": "system_health",
      "label": "System Health Dashboard",
      "type": "composite",
      "priority": 7,
      "uiContract": {
        "focusType": "read",
        "maxActions": 2
      }
    }
  ],
  "constraints": {
    "maxNodes": 1,
    "allowInterruption": true,
    "minConfidence": 0.7,
    "attentionBudget": 65
  },
  "objective": "maximize clarity, minimize noise"
}
```

### Response

```json
{
  "selected_node": "incident_investigation",
  "confidence": 0.92,
  "reasoning_tags": [
    "high_urgency",
    "user_engaged_recently",
    "matches_intent_vector",
    "critical_alert_active"
  ],
  "defer_others": [
    "billing_summary",
    "system_health"
  ],
  "attention_cost": 25
}
```

---

## AI Orchestration Service

### Implementation

```typescript
class AIOrchestrationService {
  private endpoint: string
  private logger: DecisionLogger
  
  async selectNode(
    eligibleNodes: GraphNode[],
    context: UserContext,
    constraints: DecisionConstraints
  ): Promise<AIDecisionResponse> {
    // Build request
    const request: AIDecisionRequest = {
      context_snapshot: context,
      eligible_nodes: eligibleNodes.map(node => ({
        id: node.id,
        label: node.label,
        type: node.type,
        priority: node.priorityRules?.(context) ?? 0,
        uiContract: node.uiContract
      })),
      constraints: constraints,
      objective: "maximize clarity, minimize noise"
    }
    
    // Call AI service
    const response = await fetch(this.endpoint, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(request)
    })
    
    const decision: AIDecisionResponse = await response.json()
    
    // Validate decision
    this.validateDecision(decision, eligibleNodes)
    
    // Log decision
    this.logger.logDecision(context, eligibleNodes, decision)
    
    return decision
  }
  
  validateDecision(
    decision: AIDecisionResponse,
    eligibleNodes: GraphNode[]
  ): void {
    // Ensure selected node is in eligible list
    const nodeExists = eligibleNodes.some(n => n.id === decision.selected_node)
    if (!nodeExists) {
      throw new Error(
        `AI selected invalid node: ${decision.selected_node}`
      )
    }
    
    // Ensure confidence is in valid range
    if (decision.confidence < 0 || decision.confidence > 1) {
      throw new Error(
        `Invalid confidence: ${decision.confidence}`
      )
    }
    
    // Ensure deferred nodes are in eligible list
    decision.defer_others.forEach(nodeId => {
      const exists = eligibleNodes.some(n => n.id === nodeId)
      if (!exists) {
        throw new Error(
          `AI deferred invalid node: ${nodeId}`
        )
      }
    })
  }
}
```

---

## Fallback Strategy

### When AI Fails or is Unavailable

```typescript
class AIOrchestrationService {
  async selectNode(
    eligibleNodes: GraphNode[],
    context: UserContext,
    constraints: DecisionConstraints
  ): Promise<AIDecisionResponse> {
    try {
      // Try AI decision
      return await this.callAI(eligibleNodes, context, constraints)
    } catch (error) {
      console.warn('AI decision failed, using fallback', error)
      
      // Fallback: Use priority-based selection
      return this.fallbackSelection(eligibleNodes, context)
    }
  }
  
  fallbackSelection(
    eligibleNodes: GraphNode[],
    context: UserContext
  ): AIDecisionResponse {
    // Sort by priority
    const sorted = eligibleNodes
      .map(node => ({
        node,
        priority: node.priorityRules?.(context) ?? 0
      }))
      .sort((a, b) => b.priority - a.priority)
    
    const selected = sorted[0].node
    
    return {
      selected_node: selected.id,
      confidence: 0.6,  // Lower confidence for fallback
      reasoning_tags: ["fallback_priority_based"],
      defer_others: sorted.slice(1).map(s => s.node.id),
      attention_cost: 20
    }
  }
}
```

---

## AI Model Abstraction

### Support Multiple AI Providers

```typescript
interface AIProvider {
  name: string
  selectNode(request: AIDecisionRequest): Promise<AIDecisionResponse>
}

class OpenAIProvider implements AIProvider {
  name = "openai"
  
  async selectNode(request: AIDecisionRequest): Promise<AIDecisionResponse> {
    const prompt = this.buildPrompt(request)
    
    const response = await openai.chat.completions.create({
      model: "gpt-4",
      messages: [
        {
          role: "system",
          content: "You are a navigation decision engine. Select the best node to show the user based on context."
        },
        {
          role: "user",
          content: prompt
        }
      ],
      response_format: { type: "json_object" }
    })
    
    return JSON.parse(response.choices[0].message.content)
  }
  
  buildPrompt(request: AIDecisionRequest): string {
    return `
Context:
- User intent: ${request.context_snapshot.intentVector.map(i => i.type).join(', ')}
- Confidence: ${request.context_snapshot.confidence}
- Attention budget: ${request.context_snapshot.attentionBudget}
- Urgent alerts: ${request.context_snapshot.systemState.urgentAlerts.length}

Eligible nodes:
${request.eligible_nodes.map(n => `- ${n.id} (priority: ${n.priority})`).join('\n')}

Constraints:
- Max nodes: ${request.constraints.maxNodes}
- Allow interruption: ${request.constraints.allowInterruption}

Select the best node and provide reasoning.
    `.trim()
  }
}

class DeepSeekProvider implements AIProvider {
  name = "deepseek"
  
  async selectNode(request: AIDecisionRequest): Promise<AIDecisionResponse> {
    // Similar implementation for DeepSeek
  }
}

class LocalModelProvider implements AIProvider {
  name = "local"
  
  async selectNode(request: AIDecisionRequest): Promise<AIDecisionResponse> {
    // Local model implementation
  }
}
```

### Provider Selection

```typescript
class AIOrchestrationService {
  private providers: Map<string, AIProvider>
  private activeProvider: string
  
  constructor() {
    this.providers = new Map([
      ['openai', new OpenAIProvider()],
      ['deepseek', new DeepSeekProvider()],
      ['local', new LocalModelProvider()]
    ])
    this.activeProvider = 'openai'  // Default
  }
  
  setProvider(name: string): void {
    if (!this.providers.has(name)) {
      throw new Error(`Unknown provider: ${name}`)
    }
    this.activeProvider = name
  }
  
  async selectNode(...args): Promise<AIDecisionResponse> {
    const provider = this.providers.get(this.activeProvider)!
    return provider.selectNode(...args)
  }
}
```

---

## Confidence Calibration

### Adjust AI Confidence Based on Historical Accuracy

```typescript
class ConfidenceCalibrator {
  private history: DecisionOutcome[] = []
  
  calibrate(rawConfidence: number): number {
    // Calculate historical accuracy
    const accuracy = this.calculateAccuracy()
    
    // Adjust confidence based on accuracy
    if (accuracy < 0.7) {
      // AI has been wrong often, reduce confidence
      return rawConfidence * 0.8
    }
    
    if (accuracy > 0.9) {
      // AI has been very accurate, boost confidence slightly
      return Math.min(1.0, rawConfidence * 1.1)
    }
    
    return rawConfidence
  }
  
  calculateAccuracy(): number {
    if (this.history.length === 0) return 0.5
    
    const correct = this.history.filter(h => h.wasCorrect).length
    return correct / this.history.length
  }
  
  recordOutcome(decision: AIDecisionResponse, wasCorrect: boolean): void {
    this.history.push({
      decision: decision,
      wasCorrect: wasCorrect,
      timestamp: new Date()
    })
    
    // Keep only recent history (last 100 decisions)
    if (this.history.length > 100) {
      this.history.shift()
    }
  }
}
```

---

## Testing AI Decisions

### Deterministic Testing

```typescript
class MockAIProvider implements AIProvider {
  name = "mock"
  private responses: Map<string, AIDecisionResponse> = new Map()
  
  setResponse(scenario: string, response: AIDecisionResponse): void {
    this.responses.set(scenario, response)
  }
  
  async selectNode(request: AIDecisionRequest): Promise<AIDecisionResponse> {
    // Determine scenario from request
    const scenario = this.identifyScenario(request)
    
    // Return pre-configured response
    const response = this.responses.get(scenario)
    if (!response) {
      throw new Error(`No mock response for scenario: ${scenario}`)
    }
    
    return response
  }
  
  identifyScenario(request: AIDecisionRequest): string {
    // Simple scenario identification
    if (request.context_snapshot.systemState.urgentAlerts.length > 0) {
      return "urgent_alert"
    }
    if (request.context_snapshot.intentVector.some(i => i.type === "investigate")) {
      return "investigation"
    }
    return "default"
  }
}

// Usage in tests
describe('AI Orchestration', () => {
  it('should select high-priority node during urgent alert', async () => {
    const mockAI = new MockAIProvider()
    mockAI.setResponse("urgent_alert", {
      selected_node: "incident_investigation",
      confidence: 0.95,
      reasoning_tags: ["urgent_alert"],
      defer_others: []
    })
    
    const service = new AIOrchestrationService()
    service.setProvider(mockAI)
    
    const decision = await service.selectNode(eligibleNodes, context, constraints)
    
    expect(decision.selected_node).toBe("incident_investigation")
    expect(decision.confidence).toBeGreaterThan(0.9)
  })
})
```

---

## Monitoring & Observability

### Track AI Decision Quality

```typescript
class AIMetricsCollector {
  private metrics = {
    totalDecisions: 0,
    averageConfidence: 0,
    averageLatency: 0,
    fallbackCount: 0,
    errorCount: 0
  }
  
  recordDecision(
    decision: AIDecisionResponse,
    latency: number,
    wasFallback: boolean
  ): void {
    this.metrics.totalDecisions++
    
    // Update average confidence
    this.metrics.averageConfidence = 
      (this.metrics.averageConfidence * (this.metrics.totalDecisions - 1) + decision.confidence) 
      / this.metrics.totalDecisions
    
    // Update average latency
    this.metrics.averageLatency =
      (this.metrics.averageLatency * (this.metrics.totalDecisions - 1) + latency)
      / this.metrics.totalDecisions
    
    if (wasFallback) {
      this.metrics.fallbackCount++
    }
  }
  
  getMetrics() {
    return {
      ...this.metrics,
      fallbackRate: this.metrics.fallbackCount / this.metrics.totalDecisions,
      errorRate: this.metrics.errorCount / this.metrics.totalDecisions
    }
  }
}
```

---

## Best Practices

1. **Always Validate AI Output** - Never trust AI responses blindly
2. **Have a Fallback** - Priority-based selection when AI fails
3. **Log Everything** - Record all AI decisions for debugging
4. **Calibrate Confidence** - Adjust based on historical accuracy
5. **Make AI Replaceable** - Use provider abstraction
6. **Test Deterministically** - Use mock providers in tests
7. **Monitor Quality** - Track decision quality metrics

---

## Next Steps

1. Implement `AIOrchestrationService` class
2. Create provider abstraction for multiple AI models
3. Build fallback selection logic
4. Add confidence calibration
5. Integrate with Directed Graph Engine
6. Set up monitoring and metrics collection
