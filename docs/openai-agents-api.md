# OpenAI Agents API: LoopPlane research note

Verified against the official OpenAI documentation on **2026-09-15**.

This document records what the Agents API provides, how it relates to ChatGPT Work, and what LoopPlane should own. It deliberately separates documented facts from product inference because the Agents API is a beta surface and the public contract can change.

## Bottom line

OpenAI describes the Agents API as application access to the OpenAI-managed Codex harness. The service owns the agent loop, durable session state, orchestration, context compaction, recovery, steering, and optional subagents. The application chooses the agent configuration, tools, execution environment, and how required actions are fulfilled.

OpenAI separately documents that ChatGPT Cloud Work runs the Codex harness in an isolated OpenAI-managed environment, and that Local Work uses the same core execution, isolation, and permission mechanisms on the user's device.

That supports this precise conclusion:

> The Agents API and ChatGPT Work expose the same underlying Codex-harness model through different product surfaces.

It does **not** prove that ChatGPT Work is internally implemented as a client of the public Agents API. OpenAI's public documentation does not make that implementation claim. For LoopPlane, “backbone” should mean a shared execution substrate, not a confirmed internal API dependency.

## Where the Agents API sits

OpenAI currently offers three progressively lower-level ways to build agents:

| Runtime | Who owns the loop? | Durable state | Best fit |
| --- | --- | --- | --- |
| Agents API | OpenAI-managed Codex harness | OpenAI-managed sessions, turns, and items | Long-running coding and computer-work tasks with low integration effort |
| Agents SDK | Application, using SDK primitives | Application or SDK session storage | Custom orchestration, handoffs, guardrails, and tracing |
| Responses API | Application | Application-managed or response chaining | Maximum control over every model/tool step |

The Agents API is therefore not a replacement name for the Agents SDK. It is a managed harness product with a resource-oriented API.

## Core resource model

### Agent

An agent is reusable configuration: model, instructions, tools, reasoning settings, and output behavior. A session can reference a saved `agent_id` or receive configuration directly.

Session overrides replace complete fields and arrays rather than merging individual entries. LoopPlane must therefore resolve and store the exact final configuration rather than reconstruct it later from a base agent plus an assumed merge.

### Environment

The environment is where the harness can execute shell commands, modify files, and run code. It is distinct from the harness and from application-owned function execution.

### Session

A session is the durable unit of work. It holds turns, items, environment attachment, current state, and accumulated context. Applications should persist the session ID and use the saved resource for recovery.

### Turn and item

An input creates work in a session. Turns capture agent activity and best-effort usage. Items represent messages, tool calls, command execution, output, and other observable work. A turn can complete even when an individual tool failed, so LoopPlane must evaluate item-level evidence as well as the turn status.

### Event

The API emits a typed event stream for environment connection, turn lifecycle, items, output deltas, required actions, session state, and failures. Streams are live transport, not an event archive: they do not replay missed events. Durable reconciliation comes from retrieving the session and its saved turns/items.

## Session lifecycle

A production integration should treat a run as this state machine:

```text
create session
    -> environment provisioning/connection
    -> submit input
    -> turn created/in progress
    -> zero or more items and required actions
    -> turn completed | failed | cancelled
    -> session idle or failed
    -> reconcile saved turns/items
    -> retrieve artifacts/evidence
    -> delete session and separately release external compute
```

Important rules:

- `agent.session.idle` is not proof of task success. Confirm the root turn completed and inspect failed/cancelled states and tool results.
- Closing the event stream does not cancel the work.
- Live streams do not replay. After reconnecting, retrieve saved state and resume from the durable record.
- A failed, cancelled, or environment-failed event must be surfaced as such.
- Deleting a session does not stop application-managed compute and does not emit a deletion webhook.
- Usage on session/turn resources is best effort, can be `null`, and is not a final invoice.

The API supports streaming and webhooks. Webhooks summarize session events such as created, action required, in progress, idle, and failed. For `action_required`, the application retrieves the session to determine whether it owes a function result or an environment connection.

## Environment modes

| Mode | Files and shell | Lifecycle owner | Main use |
| --- | --- | --- | --- |
| `openai_hosted` | Yes, isolated Linux workspace | OpenAI | Fastest managed execution path |
| `self_hosted` | Yes, in application-selected compute | Application | Custom infrastructure, locality, packages, controls, or persistence |
| `none` | No workspace, shell, `apply_patch`, or executor MCP | No sandbox | Remote MCP and application function workflows that need no filesystem |

### OpenAI-hosted environment

The hosted workspace is Linux at `/workspace` with Python, Node.js, and command-line tooling. Session setup can include files, environment variables, package installation, setup commands, skills, plugins, and capability directories.

Network policy can be enabled, disabled, or restricted to an explicit host allowlist. Restricted entries are exact hosts: protocols, paths, ports, and wildcards are not accepted, and redirected or subdomain hosts must be included separately.

Each session receives its own workspace. Files survive across turns while the sandbox remains available. The sandbox may be reclaimed after one hour without activity or keepalive; the timeout is not configurable. Durable outputs should be placed in `/workspace/outputs`, where the artifacts API captures immutable versions at turn completion.

### Self-hosted environment

The application starts compute and runs `codex exec-server` inside it. The executor connects outbound to the Agents API over WebSocket using the environment ID and a restricted environment key.

Security boundaries matter:

- Keep the application API key outside the sandbox.
- Give the executor the environment key as `CODEX_API_KEY`; it is restricted to connecting that environment.
- The application credential, environment key, and session must belong to the same OpenAI organization/project and user or service account.
- Isolate compute by user or workload. Sessions sharing an environment share its files and available credentials.
- Persist the mapping from Agents API session to provider compute and make startup idempotent.

The API can ask for a connection through `agent.session.action_required` with `required_action.type: "environment_connection"`. Connection wait is limited to five minutes and is not a durable job queue. A late connection does not replay timed-out input. Reusing an environment ID with replacement compute also does not restore files; storage or snapshots remain the application's responsibility.

OpenAI documents provider examples for Modal, Cloudflare, Vercel, Daytona, Blaxel, E2B, Runloop, DigitalOcean, and Oracle Cloud Infrastructure. Those examples are integrations, not evidence that the Agents API owns provider compute lifecycle.

## Tools and capability loading

### Application function tools

The application defines a function name, description, and JSON Schema. When the harness calls it, the session enters a required-action state. The application executes the function and returns an `agent.session.input.tool_result` tied to the original turn and call IDs.

Attaching a sandbox does not cause function tools to execute inside that sandbox. For side-effecting tools, persist outcomes under a session/turn/call idempotency key and check the recorded outcome before retrying.

### MCP tools

The API supports three MCP connection shapes:

| Shape | Connection runs from | Environment required? |
| --- | --- | --- |
| HTTP, `connection_origin: service` | OpenAI service | No |
| HTTP, `connection_origin: environment` | Session environment | Yes |
| stdio | Process inside session environment | Yes |

MCP configuration can restrict `allowed_tools` and can mark the server required. Credentials may be supplied inline per session or, for service-origin HTTP MCP, stored in an OpenAI vault. Inline credentials are encrypted and omitted from returned resources, but code-visible environment variables and stdio credentials can still be read by code in that environment.

### Programmatic Tool Calling

The Agents API enables Programmatic Tool Calling by default. The model can generate JavaScript in an isolated V8 runtime to coordinate eligible tools in parallel, loops, and conditions, reducing intermediate results before returning them to the agent. This runtime is separate from the session's shell environment and has no Node.js, direct network, general filesystem, or persistent JavaScript state.

Programmatic orchestration is suitable for bounded, predictable data flow. Approval-sensitive writes and semantic decisions should remain direct tool calls with explicit authorization boundaries.

### Deferred tool loading

Tools may use `defer_loading`, allowing the harness to discover relevant definitions through tool search rather than loading the entire catalog into every prompt. LoopPlane should record both the configured capability catalog and the exact tools loaded or called during a run.

## Skills, plugins, and credentials

Skills supply instructions and supporting resources. Plugins package skills, MCP configuration, or both.

- In self-hosted environments, plugin directories are mounted/copied and listed as capability directories.
- In hosted environments, plugins can be uploaded as ZIP packages or included through an environment template.
- Plugin or skill changes require a new session; an existing session does not hot-reload them.
- Subagents share the session environment and its installed capabilities.

OpenAI vaults store reusable credentials for service-origin MCP connections without exposing the secret to agent code. They support bearer credentials and existing OAuth grants, including refresh. Deleting a stored credential does not revoke it at the upstream provider and does not stop an already active session.

LoopPlane should never put secrets in versioned agent definitions, plugin packages, prompts, or evidence logs. Its capability profile should refer to credential bindings by opaque identifiers and record the authorization decision, not secret material.

## Multi-agent execution

The managed harness can create subagents with independent context for bounded parallel work. The coordinator and subagents share the same environment and filesystem. They inherit MCP configuration, web search, credentials/allowed tools, files, and shell access.

Function tools are not supported by subagents. This is a significant control-plane constraint: a LoopPlane approval or business function exposed only as an application function cannot be delegated blindly to a subagent.

The default maximum is six concurrent subagents, excluding the coordinator, and is configurable. OpenAI recommends subagents for independent work that benefits from separate context, not short dependent steps. LoopPlane should preserve parent/subagent attribution using turn `subagent_id` and include all subagent usage and actions in evidence.

## Files and artifacts

Behavior depends on the environment:

- `openai_hosted`: input files can be inline or referenced by Files API ID; outputs under `/workspace/outputs` are exposed as immutable artifacts.
- `self_hosted`: input/output retrieval is the application's or sandbox provider's responsibility; files are not published through the OpenAI artifacts API.
- `none`: no agent filesystem exists.

Documented hosted limits are 50 files at session creation, 5 MiB per inline file, 10 MiB total inline, 50 MiB per Files API file, 200 MiB per artifact file, and 500 MiB total artifacts. Artifact versions are distinguished by turn ID and path.

## Observability and cost

The normal project API can list sessions, turns, and items and retrieve best-effort usage. `subagent_id` attributes a turn to a delegated agent; `null` means root-agent work. Detailed turn traces are available in the Platform dashboard, but the dashboard trace endpoints are not a supported customer API.

Agents can make multiple model calls per task, and subagents add calls. Total cost can include model input, cached input, output and reasoning tokens, tools, hosted containers, and third-party services. Prompt caching can help when instructions and tool definitions remain stable, but a session does not guarantee a cache hit.

The pricing model is not a single “Agents API fee”: model calls follow selected-model rates, tools follow their rates, and hosted containers are billed separately. LoopPlane should store usage observations but reconcile billing from authoritative provider records rather than treating the session's best-effort counters as an invoice.

## Data and security constraints

As currently documented:

- Agents API state is retained by OpenAI to provide durable sessions.
- Data residency is available only in the United States.
- Zero Data Retention is not supported for Agents API sessions, including self-hosted environments.
- Self-hosting moves command execution and files to application-managed compute, but it does not turn the managed session/control service into a stateless or ZDR product.
- Network policy, MCP connection origin, sandbox credentials, and function execution location are separate controls and must be resolved explicitly.

These constraints can be disqualifying for some regulated workloads. LoopPlane must expose them as admission-policy inputs, not bury them in adapter documentation.

## Authentication and API maturity

The quickstart requires an API key with:

- `api.agents.read`
- `api.agents.write`
- `api.responses.write`

Raw HTTP requests use the beta header `OpenAI-Beta: agents=v1`; current OpenAI SDKs add it automatically. Session creation uses `POST /v1/agents/sessions`, exposed in the SDK as `client.beta.agents.sessions.create(...)`.

The beta namespace and header are important. LoopPlane should version the adapter, retain raw provider identifiers/events where safe, and expect schema/lifecycle changes.

## Relationship to ChatGPT Work

The documented mapping is:

| ChatGPT Work concept | Closest Agents API concept | Important difference |
| --- | --- | --- |
| Cloud Work isolated runtime | `openai_hosted` environment | Work is a ChatGPT product experience; Agents API is an application integration surface |
| Local Work on user device | `self_hosted` execution in application-selected compute | Local Work is operated by the Codex desktop app and its permission UX |
| Task/conversation | Session with turns/items | Product state and API resources are not documented as interchangeable |
| Connected apps and tools | Function/MCP tools, skills, plugins | Work applies ChatGPT workspace/admin/product controls |
| Continue while away | Durable managed session and hosted environment | This is architectural similarity, not proof of shared public endpoints |
| Isolation and permissions | Environment/network/tool policies | ChatGPT adds user-facing permission and organization policy layers |

So LoopPlane should learn from Work's user experience—durable work, clear environment choice, permissions, intervention, and evidence—while integrating against the documented Agents API contract only.

## What LoopPlane should own

The Agents API already owns the hard inner loop. LoopPlane's defensible layer is the application-specific execution contract around it:

1. **Versioned capability profiles** — exact tools, MCP servers, skills, plugins, packages, and allowed composition.
2. **Resolved environment policy** — hosted/self-hosted/none, provider, image/template, network allowlist, persistence, locality, and cleanup.
3. **Credential bindings** — which workload may use which opaque credential reference through which connection origin.
4. **Admission and approval policy** — who may start a capability, which actions pause, and who may authorize them.
5. **Durable reconciliation** — normalize streams/webhooks, then reconcile against saved provider resources after gaps or restarts.
6. **Evidence and outcome verification** — distinguish model claims, tool completion, artifact production, and verified business effect.
7. **Promotion** — test a candidate agent/capability/environment bundle, approve it, publish a new immutable version, and keep existing sessions pinned.
8. **Provider portability** — normalize lifecycle and evidence without pretending provider-specific capabilities are identical.

LoopPlane should not rebuild context compaction, the Codex execution loop, subagent coordination, or a hosted shell. It should make the use of those primitives governable and reproducible.

## First adapter design

The first implementation should use an Agents API `openai_hosted` session behind a narrow adapter:

```ts
interface AgentRuntimeAdapter {
  createSession(spec: ResolvedRunSpec): Promise<RuntimeSessionRef>;
  submitInput(session: RuntimeSessionRef, input: RuntimeInput): Promise<void>;
  streamEvents(session: RuntimeSessionRef): AsyncIterable<RuntimeEvent>;
  reconcile(session: RuntimeSessionRef): Promise<RuntimeSnapshot>;
  submitRequiredAction(action: RequiredActionResult): Promise<void>;
  listArtifacts(session: RuntimeSessionRef): Promise<RuntimeArtifact[]>;
  deleteSession(session: RuntimeSessionRef): Promise<void>;
}
```

`ResolvedRunSpec` should pin at least:

```ts
type ResolvedRunSpec = {
  schemaVersion: string;
  runtimeAdapter: "openai-agents-v1";
  agentDefinitionVersion: string;
  capabilityProfileVersion: string;
  environmentPolicyVersion: string;
  evidencePolicyVersion: string;
  approvalPolicyVersion: string;
  credentialBindingVersion: string;
  resolvedAt: string;
};
```

The stored run record should additionally include the provider session/environment IDs, actual agent/model/configuration accepted by the provider, event cursor/checkpoint, root turn outcome, required actions, artifact versions, best-effort usage, and cleanup state.

## Questions the prototype must answer

1. Can LoopPlane deterministically resolve a versioned profile into the full session request, including replace-not-merge overrides?
2. Can it recover after intentionally dropping the live stream and reconstruct the result from saved turns/items?
3. Can it prove that `idle` is not treated as success?
4. Can it enforce an application approval function without exposing that function to unsupported subagent execution?
5. Can it show the exact network, tool, plugin, and credential boundary that applied to one action?
6. Can it distinguish a produced artifact from a verified external business effect?
7. Can it clean up both the provider session and any separately owned compute idempotently?
8. Which regulated workloads are excluded by current retention/residency constraints?

## Official source index

- [Agents API overview](https://developers.openai.com/api/docs/guides/agents-api/overview)
- [Agents API quickstart](https://developers.openai.com/api/docs/guides/agents-api/quickstart)
- [Build agents: runtime comparison](https://developers.openai.com/api/docs/guides/agents)
- [Agents API architecture](https://developers.openai.com/api/docs/guides/agents-api/architecture)
- [Configure agents](https://developers.openai.com/api/docs/guides/agents-api/configuration)
- [Sessions](https://developers.openai.com/api/docs/guides/agents-api/sessions)
- [Observe sessions](https://developers.openai.com/api/docs/guides/agents-api/observability)
- [Webhooks](https://developers.openai.com/api/docs/guides/agents-api/sessions/webhooks)
- [OpenAI-hosted environments](https://developers.openai.com/api/docs/guides/agents-api/environments/openai-hosted)
- [Self-hosted environments](https://developers.openai.com/api/docs/guides/agents-api/environments/self-hosted)
- [Sandbox lifecycle](https://developers.openai.com/api/docs/guides/agents-api/environments/lifecycle)
- [Files and artifacts](https://developers.openai.com/api/docs/guides/agents-api/files-artifacts)
- [Function tools](https://developers.openai.com/api/docs/guides/agents-api/tools/functions)
- [MCP tools](https://developers.openai.com/api/docs/guides/agents-api/tools/mcp)
- [Programmatic Tool Calling](https://developers.openai.com/api/docs/guides/tools-programmatic-tool-calling)
- [Plugins](https://developers.openai.com/api/docs/guides/agents-api/plugins)
- [Vaults](https://developers.openai.com/api/docs/guides/agents-api/vaults)
- [Multi-agent execution](https://developers.openai.com/api/docs/guides/agents-api/multi-agent)
- [API pricing](https://developers.openai.com/api/docs/pricing)
- [Your data](https://developers.openai.com/api/docs/guides/your-data)
- [ChatGPT Work overview](https://learn.chatgpt.com/docs/enterprise/chatgpt-work-overview)

