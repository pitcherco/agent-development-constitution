# Reference Architecture

This directory contains reference implementations for the constitutional principles.

## Directory Structure

```
architecture/
├── README.md                    # This file
├── types/                       # Core type definitions
│   ├── agent.ts                # Agent definition schemas
│   ├── tool.ts                 # Tool interface contracts
│   ├── memory.ts               # Memory layer types
│   └── execution.ts            # Execution state types
├── components/                  # Reference implementations
│   ├── role-manager/           # Agent role and permission management
│   ├── planner/                # Task decomposition and DAG building
│   ├── executor/               # Step execution loop
│   ├── tool-gateway/           # Tool mediation layer
│   ├── memory-manager/         # Context storage and retrieval
│   ├── observer/               # Logging, tracing, metrics
│   └── guardrail-engine/       # Policy enforcement
└── examples/                    # Working examples
    ├── simple-agent/           # Single agent, few tools
    ├── multi-agent/            # Coordinated agents
    └── rag-agent/              # Retrieval-augmented agent
```

## Design Philosophy

### 1. Types Over Frameworks

Instead of importing a framework's agent class, define your own types that enforce the constitutional principles:

```typescript
// Your types are your contract
interface Agent {
  id: string;
  purpose: string;
  constraints: Constraint[];
  permissions: Permission[];
  tools: ToolRef[];
}
```

### 2. Composition Over Inheritance

Build agents by composing small, focused components:

```typescript
const myAgent = compose(
  withRoleManager(agentDefinition),
  withPlanner(dagBuilder),
  withExecutor(toolGateway),
  withObserver(logger)
);
```

### 3. Explicit Over Implicit

No magic. Every decision point is visible and logged:

```typescript
// Bad: Framework magic
const result = await agent.run(task);

// Good: Explicit pipeline
const plan = await planner.decompose(task);
const validated = await guardrails.check(plan);
for (const step of validated.steps) {
  const result = await executor.run(step, { budget, context });
  await observer.log(step, result);
}
```

## Getting Started

1. Start with the type definitions in `types/`
2. Implement components based on your needs
3. Reference the examples for integration patterns
4. Always write tests against adversarial scenarios

