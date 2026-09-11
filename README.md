# LoopPlane

An API-first control plane for deploying versioned, custom agent harnesses. LoopPlane records and enforces the application-specific boundaries around a runtime—capabilities, environment, sessions, evidence, and promotion policy.

## Why this exists

Managed agent platforms make durable agent execution much easier. Production correctness still needs an application-owned layer that can answer:

> What could this runtime do at the moment it took this action?

LoopPlane treats that answer as a versioned capability profile, recorded with each run.

## Execution model

| | Closed action space | Open action space |
| --- | --- | --- |
| Fixed control flow | 1. Workflow | 3. Generated-computation workflow |
| Model-driven control flow | 2. Bounded agent runtime | 4. Generative harness |

The first milestone targets quadrant 2: the agent chooses actions from an approved capability set. It may generate and run task-local helpers in an isolated environment, but it cannot silently add persistent capabilities or modify the harness.

## Core resources

- **Harness version** — loop contract, model adapters, context strategy, supported execution modes.
- **Capability profile** — allowed tools, skills, packages, permissions, and composition rules.
- **Session** — durable work history, checkpoints, outstanding actions, and pinned versions.
- **Execution environment** — compute, filesystem, network boundary, lifecycle, and artifact store.
- **Evidence** — requested action, authorization, execution result, and verified business effect.

See [the architecture note](docs/architecture.md) and [the first milestone](docs/first-milestone.md).

## Status

Architecture scaffold only. No OpenAI API key, provider SDK, or deployment configuration is committed to this repository.

## Next step

Implement the session API and an immutable `ResolvedRunSpec`, then connect one hosted agent runtime behind the adapter boundary.
