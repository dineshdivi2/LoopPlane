# First milestone: bounded agent service

## Outcome

An application can submit an objective using a named agent profile and capability profile, observe progress, intervene at policy gates, and retrieve evidence-backed results.

## Vertical slice

1. `POST /sessions` resolves and stores a `ResolvedRunSpec`.
2. The `openai-agents-v1` adapter starts one `openai_hosted` Agents API session and stores its provider identifiers.
3. The adapter streams normalized events and reconciles durable turns/items after a disconnect or restart.
4. `GET /sessions/:id` returns state, checkpoints, pending approvals, cleanup state, and result status.
5. `GET /sessions/:id/evidence` returns append-only evidence records with provider attribution.
6. A generated helper runs only in the session workspace and becomes an immutable artifact, not a persistent tool.

## Acceptance criteria

- A completed run can be replayed as an ordered evidence timeline.
- Every action has a resolved capability profile version.
- An approval gate prevents execution until an authorized decision is recorded.
- A failed or cancelled provider turn is visible as failed or cancelled; it is never reported as success from an idle session alone.
- Dropping and reconnecting the live stream does not lose the final outcome because the adapter reconciles saved turns and items.
- The stored run identifies the exact agent definition, capability, environment, approval, evidence, and credential-binding versions used.
- Generated helper output is attached to the session without altering its pinned agent or capability profile.

## Deferred

- Cross-provider adapters.
- Persistent capability registration.
- Harness self-modification.
- Multi-tenant authorization and billing.
- A self-hosted environment adapter.
