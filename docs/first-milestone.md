# First milestone: bounded agent service

## Outcome

An application can submit an objective using a named harness version and capability profile, observe progress, intervene at policy gates, and retrieve evidence-backed results.

## Vertical slice

1. `POST /sessions` resolves and stores a `ResolvedRunSpec`.
2. A runtime adapter starts one agent session and streams normalized events.
3. `GET /sessions/:id` returns state, checkpoints, pending approvals, and result status.
4. `GET /sessions/:id/evidence` returns append-only evidence records.
5. A generated helper runs only in an isolated task directory and becomes an artifact, not a persistent tool.

## Acceptance criteria

- A completed run can be replayed as an ordered evidence timeline.
- Every action has a resolved capability profile version.
- An approval gate prevents execution until an authorized decision is recorded.
- A failed or cancelled provider turn is visible as failed or cancelled; it is never reported as success from an idle session alone.
- Generated helper output is attached to the session without altering its pinned harness or capability profile.

## Deferred

- Cross-provider adapters.
- Persistent capability registration.
- Harness self-modification.
- Multi-tenant authorization and billing.
