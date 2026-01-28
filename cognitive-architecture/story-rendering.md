# Story Rendering Layer

---
**Status**: 📋 **PLANNED** - The presentation layer for cognitive navigation  
**Priority**: 🔴 **P0** - Core UI architecture for the vision  
**Last Updated**: 2026-01-28  
**Owned By**: Frontend Team, UX Design  

---

## Overview

The Story Rendering Layer replaces traditional routing, tabs, and menus with a **cognitive-driven presentation system**. It operates on the "SPA Without Navigation" principle: the UI morphs based on state transitions rather than jumping between routes.

**Key Principle**: One route, one shell, one active node.

---

## Core Principles

1. **State Transitions, Not Navigation**: The UI is a pure function of the active node.
2. **Minimalist Shell**: A consistent container that provides global context without clutter.
3. **UI Contracts**: Each node declares its presentation requirements (focus type, actions, persistence).
4. **Motion-Authored Transitions**: Smooth micro-animations convey relationship and context changes.

---

## The SPA Architecture

Traditional apps use routes like `/incidents` or `/settings`. The Story Rendering Layer uses a single route with a dynamic shell:

```tsx
<App>
  <StoryShell>
    <ContextHeader />
    <NodeRenderer node={activeNode} />
    <MinimalActions />
  </StoryShell>
</App>
```

### StoryShell Implementation

```tsx
const StoryShell: React.FC = ({ children }) => {
  return (
    <div className="story-shell h-screen flex flex-col overflow-hidden bg-slate-950 text-slate-100">
      <ContextHeader />
      <main className="flex-1 overflow-y-auto p-6 transition-all duration-500 ease-in-out">
        {children}
      </main>
      <GlobalStatusBar />
    </div>
  );
};
```

---

## UI Contracts

The `uiContract` defined in the graph node determines how the renderer treats the component.

```typescript
interface UIContract {
  focusType: "read" | "decide" | "act";
  maxActions: number;
  interruptionAllowed: boolean;
  persistence: "sticky" | "ephemeral";
}
```

### focusType Treatments

| Focus Type | Visual Treatment | UX Goal |
|------------|------------------|----------|
| **read** | High whitespace, serif typography option, minimized sidebars. | Passive absorption of information. |
| **decide** | Side-by-side comparisons, highlighted choices, prominent buttons. | Clear decision making. |
| **act** | Condensed inputs, focus on primary action, confirmation steps. | Execution of specific tasks. |

---

## Node Rendering

### NodeRenderer Factory

```tsx
const NodeRenderer: React.FC<{ node: GraphNode }> = ({ node }) => {
  // Map node types to functional components
  switch (node.type) {
    case "label":
      return <LabelStory node={node} />;
    case "branch":
      return <DecisionStory node={node} />;
    case "input":
      return <ActionStory node={node} />;
    case "composite":
      return <CompositeStory node={node} />;
    default:
      return <UnknownStory node={node} />;
  }
};
```

### Focus Wrapper

Each story component is wrapped in a FocusWrapper that applies styles dictated by the `uiContract`.

```tsx
const FocusWrapper: React.FC<{ contract: UIContract }> = ({ contract, children }) => {
  const focusClasses = {
    read: "max-w-3xl mx-auto text-lg leading-relaxed",
    decide: "grid grid-cols-1 md:grid-cols-2 gap-8",
    act: "max-w-xl mx-auto bg-slate-900 rounded-xl p-8 shadow-2xl"
  };

  return (
    <div className={`transition-all duration-700 ${focusClasses[contract.focusType]}`}>
      {children}
    </div>
  );
};
```

---

## Transitions & Animations

Transitions are managed via a central coordinator to ensure the user doesn't lose context.

```tsx
import { AnimatePresence, motion } from "framer-motion";

const NavigationTransition: React.FC = ({ children, nodeId }) => {
  return (
    <AnimatePresence mode="wait">
      <motion.div
        key={nodeId}
        initial={{ opacity: 0, scale: 0.98, y: 10 }}
        animate={{ opacity: 1, scale: 1, y: 0 }}
        exit={{ opacity: 0, scale: 1.02, y: -10 }}
        transition={{ duration: 0.4, ease: [0.23, 1, 0.32, 1] }}
      >
        {children}
      </motion.div>
    </AnimatePresence>
  );
};
```

---

## Minimal Actions

Avoid fixed menus. Instead, expose actions that are relevant to the current node or the global intent.

```tsx
const MinimalActions: React.FC = () => {
  const { activeNode, availableActions } = useCognitiveNav();

  return (
    <div className="fixed bottom-8 right-8 flex gap-4">
      {availableActions.map(action => (
        <PrimaryButton key={action.id} onClick={action.handler}>
          {action.label}
        </PrimaryButton>
      ))}
      <ContextualSearch />
    </div>
  );
};
```

---

## Anti-Patterns

1. **URL-based persistence**: Avoid encoding complex state in URLs. Persistence should live in the `UserContext`.
2. **Standard Headers**: Avoid generic "Header" components that take up screen real estate without context. Use `ContextHeader`.
3. **Tabbed Interfaces**: Tabs imply multiple active contexts. We use "One active node" to preserve focus.

---

## Next Steps

1. Create a `UIContract` CSS utility system.
2. Implement the `StoryShell` and `NodeRenderer` in the React frontend.
3. Build the `FocusWrapper` to handle automatic layout shifts based on node metadata.
4. Define standard animations for node-to-node transitions.
