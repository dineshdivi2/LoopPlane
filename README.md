# LoopPlane

An API-first control plane for running versioned, application-specific agent harness configurations. LoopPlane records and enforces the boundaries around a managed or self-hosted runtime—capabilities, environment, credentials, approvals, evidence, and promotion policy.

## Why this exists

Managed harnesses such as OpenAI's Agents API make durable agent execution much easier. Production correctness still needs an application-owned layer that can answer:

> What could this runtime do at the moment it took this action?

LoopPlane treats that answer as a versioned capability profile, recorded with each run.

## Execution model

| | Closed action space | Open action space |
| --- | --- | --- |
| Fixed control flow | 1. Workflow | 3. Generated-computation workflow |
| Model-driven control flow | 2. Bounded agent runtime | 4. Generative harness |

The first milestone targets quadrant 2: the agent chooses actions from an approved capability set. It may generate and run task-local helpers in an isolated environment, but it cannot silently add persistent capabilities or modify the harness.

## Core resources

- **Agent profile** — model, instructions, reasoning policy, adapter compatibility, and supported execution modes.
- **Capability profile** — allowed tools, skills, packages, permissions, and composition rules.
- **Session** — durable work history, checkpoints, outstanding actions, and pinned versions.
- **Execution environment** — compute, filesystem, network boundary, lifecycle, and artifact store.
- **Credential binding** — which workload may access an opaque credential reference and through which connection boundary.
- **Approval policy** — operations that require a recorded human or service decision.
- **Evidence** — requested action, authorization, execution result, artifact, and verified business effect.

See [the architecture note](docs/architecture.md), [the first milestone](docs/first-milestone.md), and the dated [OpenAI Agents API research note](docs/openai-agents-api.md).

## Status

Architecture scaffold only. No OpenAI API key, provider SDK, or deployment configuration is committed to this repository.

## Next step

Implement the session API and an immutable `ResolvedRunSpec`, then connect the OpenAI Agents API hosted environment as the first runtime adapter.
