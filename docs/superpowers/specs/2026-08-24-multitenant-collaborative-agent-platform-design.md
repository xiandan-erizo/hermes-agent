# Multi-Tenant Collaborative Agent Platform Design

**Status:** Proposed

## Summary

Build a product layer that lets organizations co-create and operate Agents
through both a Feishu bot and a web console. A tenant owns one or more
tenant-scoped Agent instances. All authorized members of one tenant Agent
share that Agent's long-term memory; each member's conversation history stays
private unless the tenant deliberately creates a shared conversation.

Hermes remains the execution runtime. A Hermes Profile is an implementation
binding for runtime configuration, credentials, skills, workspace, and
runtime state. It is not a product-facing user, tenant, or permission model.

The central invariant is:

```text
Every read, write, tool execution, memory operation, and session lookup is
authorized and scoped by (tenant_id, tenant_agent_id, principal_id).
```

## Goals

- Let a tenant create or adopt an Agent and invite multiple members.
- Offer one coherent Agent through Feishu and the web console.
- Let all members of the same tenant Agent contribute to and use its shared
  long-term memory.
- Keep member-private conversation history and console access isolated.
- Support a principal who belongs to multiple tenants and multiple Agents.
- Make tenant, Agent, membership, memory provenance, conversation ownership,
  and policy changes auditable.
- Reuse Hermes Gateway, Profiles, session storage, Skills, tools, and memory
  provider integration without adding tenant-specific model tools.

## Non-Goals

- Expose the stock Hermes Dashboard to end users.
- Create one Hermes Profile or one Agent per member.
- Share a tenant's data with another tenant by default.
- Treat an unverified client-supplied tenant, user, profile, or session ID as
  an authorization decision.
- Build a generic multi-runtime broker or marketplace in the first release.

## Product Model

The product presents an Agent Workspace, not a Profile manager.

```text
Tenant Workspace
  +-- Tenant Agent
  |     +-- shared memory and memory governance
  |     +-- Feishu and web chat entry points
  |     +-- private member conversations
  |     +-- members, roles, policies, and audit
  +-- Tenant Agent
        +-- independent shared memory and policies
```

### Product Terms

| Term | Meaning | Stable identifier |
| --- | --- | --- |
| Tenant | An organization and its authorization boundary. | `tenant_id` |
| Agent template | A reusable platform-level blueprint for prompt, Skills, and defaults. | `agent_template_id` |
| Tenant Agent | An Agent instance owned by one tenant. It is the shared-memory and policy boundary. | `tenant_agent_id` |
| Principal | A person or service with a verified identity. | `principal_id` |
| Membership | A principal's role and state in a tenant Agent. | `(tenant_agent_id, principal_id)` |
| Logical thread | A user-visible conversation that may be entered through Feishu or web. | `thread_id` |
| Hermes session | An opaque physical execution session. It may be replaced while its logical thread remains. | `hermes_session_id` |
| Profile binding | The runtime Profile or equivalent isolated runtime selected for a tenant Agent. | `profile_binding_id` |

`agent_id` alone is not a sufficient memory or authorization namespace in a
multi-tenant product. The default shared-memory namespace is
`tenant_agent_id`, which already identifies one `(tenant_id, agent)` pair.

An intentionally platform-global Agent is modeled explicitly as a distinct
global workspace. It must not arise by omitting `tenant_id` from ordinary
tenant data.

## User Experience

### Tenant Administrator

1. Creates a tenant workspace.
2. Selects an Agent template or creates a tenant-specific Agent.
3. Connects the approved Feishu installation and enables the web console.
4. Invites members and assigns `owner`, `editor`, or `member` roles.
5. Configures Agent policy, Skills, tools, and memory governance.

### Member

1. Opens the web console in a selected tenant workspace or sends a Feishu
   message to the tenant Agent.
2. Starts or resumes one of their own logical threads.
3. Uses the same logical thread from web and Feishu when the selected tenant
   Agent is the same.
4. Benefits from the tenant Agent's shared memory and contributes new memory
   candidates through conversation or an explicit memory action.
5. Sees only their own private threads, unless the tenant creates a shared
   thread and grants access to it.

### Multi-Tenant Membership

A person may belong to several tenant workspaces. The web URL and authenticated
server session identify the active tenant Agent. A Feishu group is explicitly
bound to one tenant Agent. A direct-message conversation has no inherent
tenant selection when the same bot serves more than one membership; the
platform stores an active tenant Agent selection and requires an explicit
selection when it is absent or ambiguous.

## Roles and Authorization

| Role | Conversations | Shared memory | Agent configuration | Members and audit |
| --- | --- | --- | --- | --- |
| Member | Own private threads and granted shared threads | Read published memory; create candidates | No | No |
| Editor | Member capabilities | Publish, edit, retract, and restore memory | Policy-limited settings | Read audit as permitted |
| Owner | All tenant-Agent resources | Full governance | Full tenant-Agent configuration | Manage members, integrations, and audit |

Role checks occur server-side for every operation. A browser may request a
resource by ID, but the server derives `tenant_id`, `tenant_agent_id`, and
`principal_id` from authenticated context and verifies membership before it
loads the resource. A missing or mismatched scope returns not found rather
than disclosing that another tenant's resource exists.

## Conversations

### Logical Thread Contract

A logical thread is the product record that connects a member, a tenant Agent,
and its Hermes execution session.

```text
(tenant_agent_id, principal_id, thread_id)
  -> current opaque hermes_session_id
  -> zero or more historical Hermes session segments
```

The logical thread is the stable web/Feishu resume target. Hermes may rotate a
physical session because of reset, compression, migration, recovery, or a
runtime replacement; the product must update the mapping rather than expose a
runtime ID to the browser.

Private threads have an owner principal. Shared threads have an explicit
participant list or tenant-Agent-wide visibility. A group-chat thread is not
implicitly shared merely because it originated in a group; its visibility is a
product policy decision.

### Web and Feishu

The web console is a first-class chat client, not a transcript viewer. Both
entry points resolve the same trusted execution context and operate on the
same logical thread mapping. A web turn must not create an unrelated Dashboard
or API-server session merely because its transport differs from Feishu.

The stock Hermes Dashboard remains an operator console. Its existing session
endpoints list and retrieve the Profile's sessions without an end-user
principal filter, so they are not exposed to product users. The product ships
a separate user console surface with scoped APIs such as:

```text
GET  /v1/me/tenant-agents
GET  /v1/tenant-agents/{tenant_agent_id}/threads
POST /v1/tenant-agents/{tenant_agent_id}/threads
GET  /v1/threads/{thread_id}
POST /v1/threads/{thread_id}/messages
GET  /v1/tenant-agents/{tenant_agent_id}/memory
```

Every endpoint obtains the principal from the authenticated request, not from
a `user_id` parameter.

## Shared Memory

### Scope

Every tenant Agent has one shared memory namespace:

```text
memory_namespace = "tenant-agent:" + tenant_agent_id
```

All active members may benefit from published entries in that namespace. This
is a deliberate collaborative feature: information introduced by one member
may influence the Agent's answers to another member in the same tenant Agent.
Conversation transcripts do not become directly visible to other members just
because an extracted memory entry is shared.

### Governance

Shared memory is not an unversioned prompt appendix. Each entry stores:

```text
memory_id, tenant_agent_id, content, source_thread_id, source_principal_id,
created_at, created_by, status, revision, supersedes_memory_id, retracted_at
```

`status` is one of `candidate`, `published`, `retracted`, or `superseded`.
The first release may auto-publish entries under a tenant policy, but it must
retain provenance and support retraction and restoration. Editors and owners
can moderate the shared memory. Runtime configuration, tool permissions, and
credentials are never writable through ordinary memory extraction.

If a tenant needs personal memory later, it is a separate namespace keyed by
`(tenant_agent_id, principal_id)`. It must never be merged into the shared
tenant-Agent memory implicitly.

## Identity and Tenant Resolution

The system normalizes each inbound identity to a trusted principal binding:

```text
identity_provider + external_subject + tenant_binding -> principal_id
```

For Feishu, the adapter currently supplies a primary `user_id` or `open_id`
and may supply `union_id` as `user_id_alt`. `union_id` is developer-scoped,
not a tenant boundary. Tenant resolution must instead use an authenticated
Feishu installation, a verified workspace binding, or a previously verified
web identity binding. It must not infer a tenant from a display name, group
name, or a browser-supplied request field.

The resolved context contains at least:

```text
tenant_id, tenant_agent_id, principal_id, role, thread_id,
hermes_session_id, policy_revision
```

This context is created on the trusted server path and passed to the Hermes
runtime. It is not supplied or amended by prompts, model output, or browser
parameters.

## Runtime Binding

### Recommended Isolation Level

For a production tenant Agent, bind each `(tenant_id, agent)` to its own
runtime Profile or equivalent isolated execution scope. A Profile owns
configuration, Skills, memory-provider configuration, state database,
workspace, and credentials; sharing it across tenants would make a plugin
responsible for preventing every possible cross-tenant read or write.

A single multiplexing Gateway process may host multiple bindings when the
runtime supports a trusted routing decision before session creation. The
shared Feishu app connects once at the ingress profile. It must not be
configured independently with the same token in every tenant Profile.

Logical isolation in a single Profile is acceptable only for an explicitly
low-risk deployment where the Agent configuration, tools, credentials, and
workspace are identical for all tenants. Even then, the execution context,
session keys, conversation store, and memory provider must include
`tenant_agent_id`. This is not equivalent to strong runtime isolation.

### Current Hermes Constraints

- Profile routing currently selects Profiles from platform, guild, chat, and
  thread discriminators. It does not accept a normalized tenant decision or a
  user-to-tenant-Agent membership lookup.
- `pre_gateway_dispatch` is early enough to resolve trusted ingress context,
  but its current result contract supports `allow`, `rewrite`, and `skip`; it
  does not carry a durable tenant execution context through session creation
  and agent execution.
- `SessionSource` persists platform, user, chat, thread, scope, and Profile
  fields but no tenant identifier.
- Hermes accepts one external memory provider. A tenant-aware provider must
  internally namespace records by `tenant_agent_id`; installing a separate
  memory provider per tenant is not the model.

The product layer therefore needs a concrete bridge contract before it can
route tenant-specific turns safely. The contract must be generic enough to
support this product but must not add a permanent model tool or rebuild the
system prompt mid-conversation.

## Proposed Components

```text
Feishu app / Web console
          |
          v
Tenant Agent Platform
  - identity and tenant resolution
  - membership and policy authorization
  - logical threads and session mapping
  - shared-memory governance
  - audit and user console APIs
          |
          v
Hermes Gateway and runtime
  - Feishu delivery and session execution
  - Profile-scoped Skills, tools, credentials, and workspaces
  - model calls and tool execution
          |
          v
Tenant-aware memory provider
  - namespace = tenant_agent_id
  - provenance, revisions, and retraction
```

The platform may begin as a trusted standalone plugin with a dashboard API and
frontend, provided it owns its durable product records and enforces the same
authorization model. It must not treat the existing Dashboard session token
as an end-user identity mechanism.

## Data Model

The exact database technology is open, but these relationships are required.

```text
tenants(id, name, status, created_at)
principals(id, status, created_at)
principal_identities(id, principal_id, provider, external_subject, tenant_id)
agent_templates(id, revision, configuration_digest, status)
tenant_agents(id, tenant_id, agent_template_id, profile_binding_id, status)
tenant_agent_memberships(tenant_agent_id, principal_id, role, status)
threads(id, tenant_agent_id, owner_principal_id, visibility, status)
thread_participants(thread_id, principal_id, role)
thread_runtime_sessions(thread_id, hermes_session_id, started_at, ended_at)
memory_entries(id, tenant_agent_id, source_thread_id, source_principal_id,
               status, revision, content, created_at)
audit_events(id, tenant_id, tenant_agent_id, principal_id, action, resource,
             payload_digest, created_at)
```

Every non-global foreign key is tenant-consistent. For example, a thread may
reference only a tenant Agent in its own tenant, and a memory entry may cite
only a thread belonging to that tenant Agent.

## Security and Privacy Rules

- Tenant boundary checks happen before thread lookup, memory retrieval, tool
  authorization, and delivery.
- Session and memory identifiers are opaque to browser clients; a valid UUID
  or Hermes session ID never grants access by itself.
- Member-private transcripts stay private. Shared memory retains provenance but
  does not grant transcript access.
- Cross-tenant responses, memory retrievals, search results, and errors fail
  closed and do not reveal resource existence.
- The Agent receives only the tenant Agent's resolved policy and capabilities.
  A shared bot process must not source credentials from a different tenant
  Profile.
- Shared-memory writes are audited and reversible. Tool permissions and secret
  configuration are distinct from memory content.

## Rollout Plan

### Phase 1: Single Tenant Agent Product

- Build the product records for one tenant Agent, member identity binding,
  private threads, and a scoped web console.
- Integrate one Feishu bot and one web chat surface with a logical-thread to
  Hermes-session mapping.
- Implement shared-memory entries with provenance and editorial retraction.
- Keep the standard Hermes Dashboard operator-only.

### Phase 2: Multi-Tenant Control Plane

- Add tenant workspace onboarding, membership roles, and multiple tenant
  Agents.
- Resolve trusted tenant context for Feishu and web before Agent execution.
- Add tenant-Agent-specific runtime bindings and memory namespaces.
- Add audit search and tenant-admin memory governance.

### Phase 3: Strong Runtime Isolation

- Route a shared Feishu ingress to the verified tenant-Agent runtime binding.
- Give tenant Agents isolated Profile/runtime scopes where credentials,
  workspaces, tools, or compliance policies differ.
- Add reconciliation for runtime replacement while preserving logical threads.

## Validation Strategy

- Identity: a Feishu and web login bound to the same principal resolve the
  same tenant Agent and may resume one logical thread.
- Membership: a principal outside a tenant Agent receives no information about
  its threads, memories, or existence.
- Conversation isolation: two members in one tenant Agent cannot read each
  other's private thread through list, detail, search, export, or resume APIs.
- Memory sharing: a published entry from member A is retrievable by the Agent
  for member B in the same tenant Agent, but not in a different tenant Agent.
- Memory governance: retraction prevents future retrieval and leaves an audit
  record; restoring a revision is deterministic.
- Runtime scope: a turn never reads a different tenant's Profile secrets,
  workspace, tools, or memory namespace.
- Recovery: replacing an underlying Hermes session preserves the logical
  thread's continuity without replaying or duplicating product history.

## Decisions Requiring Confirmation

1. The default product promise is that shared memory is shared only within a
   tenant Agent, never across ordinary tenants.
2. Private threads are visible only to their owner by default; group/shared
   thread visibility is an explicit policy.
3. Tenant Agents with different secrets, workspaces, tools, or compliance
   requirements receive isolated runtime Profile bindings.
4. A single Feishu bot may serve several tenant Agents, but direct messages
   require a durable active-tenant selection when one principal belongs to
   multiple tenant Agents.
5. The user-facing console is a new scoped product surface, not a restricted
   view of the stock Hermes Dashboard.
