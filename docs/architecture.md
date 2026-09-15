# Architecture

## Boundary

LoopPlane sits between an application and one or more agent runtimes.

```text
Application -> LoopPlane control plane -> runtime adapter -> agent runtime
                    |                    |
                    v                    v
             evidence ledger       execution environment
```

The application owns business rules and integration handlers. The runtime owns its agent loop. LoopPlane owns the resolved execution contract and its audit trail.

## Resolved run spec

Every run resolves these immutable references before execution:

```ts
type ResolvedRunSpec = {
  schemaVersion: string;
  agentDefinitionVersion: string;
  capabilityProfileVersion: string;
  environmentPolicyVersion: string;
  evidencePolicyVersion: string;
  approvalPolicyVersion: string;
  credentialBindingVersion: string;
  runtimeAdapter: "openai-agents-v1";
  resolvedAt: string;
};
```

The resolved spec is attached to every action and checkpoint. A later capability change creates a new profile version; existing sessions remain pinned unless an explicit migration is approved.

## Control points

1. Admit an objective only against a compatible harness and capability profile.
2. Persist a session and its resolved run spec.
3. Stream normalized runtime events into the evidence ledger.
4. Reconcile missed events from the provider's durable session, turn, and item resources.
5. Pause for approvals before governed operations.
6. Keep secrets outside versioned definitions and resolve only opaque credential bindings.
7. Run generated helpers in an isolated, task-local environment.
8. Evaluate a candidate capability change outside production sessions.
9. Promote a tested candidate under policy into a new version.

## First runtime adapter: OpenAI Agents API

The first adapter uses an `openai_hosted` Agents API session. OpenAI owns the Codex loop, context compaction, session orchestration, and hosted sandbox. LoopPlane owns resolution, policy, required-action handling, evidence normalization, outcome verification, and cleanup orchestration.

The adapter must preserve the raw provider session, environment, turn, item, artifact, and subagent identifiers alongside normalized records. Live streams are not replayable, so recovery always reconciles the saved provider resources. An idle session is not a successful run unless the root turn completed and the required evidence policy passes.

See [the Agents API research note](openai-agents-api.md) for the documented contract and product boundaries.

## Non-goals for the first milestone

- Reimplement a provider's agent loop or hosted sandbox.
- Allow an agent to mutate the running harness.
- Treat tool availability as proof of business effect.
- Migrate active sessions automatically.
