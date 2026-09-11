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
  harnessVersion: string;
  capabilityProfileVersion: string;
  environmentPolicyVersion: string;
  evidencePolicyVersion: string;
  runtimeAdapter: string;
};
```

The resolved spec is attached to every action and checkpoint. A later capability change creates a new profile version; existing sessions remain pinned unless an explicit migration is approved.

## Control points

1. Admit an objective only against a compatible harness and capability profile.
2. Persist a session and its resolved run spec.
3. Stream normalized runtime events into the evidence ledger.
4. Pause for approvals before governed operations.
5. Run generated helpers in an isolated, task-local environment.
6. Evaluate a candidate capability change outside production sessions.
7. Promote a tested candidate under policy into a new version.

## Non-goals for the first milestone

- Reimplement a provider's agent loop or hosted sandbox.
- Allow an agent to mutate the running harness.
- Treat tool availability as proof of business effect.
- Migrate active sessions automatically.
