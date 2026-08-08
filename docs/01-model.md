# Core Domain Model

**Status:** design **Date:** 2026-08-07 **Depends on:** [`00-overview.md`](./00-overview.md)

---

## 1. Why this document comes first

Pterodactyl's central noun is `Server`, and `Server` means *a game server*. Every table, endpoint,
permission, and UI screen is built around that word. The reason no fork has been able to make
Pterodactyl general-purpose is not that nobody tried; it is that the vocabulary does not permit it.

Nineteen documents follow this one. If the nouns here are wrong, all nineteen are wrong.

---

## 2. A correction to the earlier proposal

An earlier draft proposed a single field:

```
shape: console | service | job | cron | stateful | machine
```

Working it through shows this conflates four independent axes:

- `stateful` is not a shape. **Any** workload may have durable volumes, a game server, a database,
  and a nightly job can all be stateful. Statefulness is a property, not a category.
- `machine` is not a shape. A virtual machine is a **runtime driver choice**, the same persistent
  lifecycle, confined by a hypervisor instead of a kernel namespace. Making it a shape would put
  the isolation decision in the wrong field and quietly contradict Bet 2.
- A Minecraft server is simultaneously *console-interactive* and *a network service*. A single enum
  forces a false choice.
- A Discord bot is long-running with no endpoints and no interactive console. The enum has no
  correct value for it.

The correct model separates the axes:

| Axis | Field | Values |
|---|---|---|
| Lifecycle: how the scheduler treats it | `lifecycle` | `persistent` · `task` · `scheduled` |
| Interaction: what the stream surface offers | `console` | `none` · `logs` · `interactive` |
| Exposure: how the network reaches it | `endpoints[]` | declared list |
| State: what survives a restart | `volumes[]` | declared list |
| Liveness: what "healthy" means | `health` | probe definition |
| Confinement: how strongly it is isolated | `runtime` | driver + isolation tier |

"Shape" survives as an informal word for a **named preset over these fields**, published by a
recipe. It is not a field on the workload.

What this buys, concretely:

| Workload | lifecycle | console | endpoints | volumes | health | runtime |
|---|---|---|---|---|---|---|
| Minecraft server | persistent | interactive | game/25565 | data | query: minecraft | oci / container |
| Next.js app | persistent | logs | http/3000 + ingress | none | http: /healthz | oci / container |
| PostgreSQL | persistent | logs | tcp/5432 | data | tcp + query | oci / container |
| Discord bot | persistent | logs | none |, | heartbeat | oci / container |
| Nightly backup | scheduled | logs | none | archive | exit: 0 | oci / container |
| Customer VPS | persistent | interactive (serial) | tcp/22, ip | root disk | agent or tcp | kvm / vm |
| Untrusted plugin build | task | logs | none | workspace | exit: 0 | firecracker / microvm |

One scheduler, one storage layer, one network layer, one identity model, one API. A VM is not a
special case in the code, it is `runtime.driver: kvm`. That is Bet 2 made structural rather than
aspirational.

---

## 3. Object catalogue

Every object below has: a **purpose** (one sentence), **fields**, **invariants** (what must always
be true), and where relevant a **lifecycle**.

Field types are indicative, not a schema. The normative schema lives in `10-api.md`.

---

### 3.1 Tenancy

#### Organization

> The root of a tenancy tree. Owns nothing directly; contains projects and sub-organizations.

```
Organization
  id            OrgID          immutable, never reused
  name          string         mutable, unique within parent
  parent        OrgID?         null for a root organization
  quota         Quota          resources this org may consume, ≤ parent's remaining
  created       Timestamp
```

**Invariants**
- An organization's quota can never exceed its parent's unallocated remainder.
- The tree has no cycles.
- Deleting an organization requires its subtree to be empty.

Recursive organizations are how reselling exists without a reseller feature (Rule K-14, Bet 4). A
hosting provider is a root org. Its customer is a child org with an allocated quota. That
customer's own customers are grandchildren. Nothing in the code distinguishes these levels.

#### Project

> A named grouping of workloads inside an organization, and the usual unit of delegation.

```
Project
  id            ProjectID      immutable
  org           OrgID
  name          string         mutable, unique within org
  quota         Quota?         optional sub-limit within the org's quota
  labels        map[str]str
```

Most grants are scoped to a project. A community's "Minecraft network" and its "website" are two
projects; a moderator can hold authority over one and not the other.

#### Quota

> An enforced upper bound on resource consumption, inherited down the tenancy tree.

```
Quota
  cpu           millicores
  memory        bytes
  disk          bytes
  disk_iops     ops/sec
  egress        bytes/month
  bandwidth     bits/sec
  workloads     count
  volumes       count
  endpoints     count
```

**Invariants**
- Every dimension is enforced by the kernel at the node and metered (Rule K-3, P4). A dimension
  that cannot be enforced is not added to this struct.
- Sum of children's quotas ≤ parent's quota. Allocation is checked at grant time, not at use time.

> Named `Quota`, deliberately not `Plan`, `Plan` is taken by §3.3 and the collision would be
> constant. A commercial "plan" is a billing concept and lives in the billing system (Rule K-12).

---

### 3.2 Identity and authority

#### Subject

> Anything that can act. Not necessarily a person, and not necessarily registered with Korpis.

```
Subject = User | Token | ExternalIdentity | Workload | Extension
```

| Variant | Is | Example |
|---|---|---|
| `User` | a local account | someone who signed up |
| `Token` | a bearer credential | a CI pipeline, a billing integration |
| `ExternalIdentity` | a claim from an identity provider | Discord user `123`, Discord role `@mod` in guild `456`, an OIDC group |
| `Workload` | a running workload acting for itself | a bot that restarts its own sibling |
| `Extension` | a running extension | a backup extension reading volumes |

`ExternalIdentity` being a first-class subject is what makes Bet 1 work. A grant can name *the
Discord role `@moderator` in guild X* directly. Nobody needs a Korpis account for that role to
carry authority, and losing the role removes the authority with no deprovisioning step.

#### Grant

> The only authority primitive. There are no roles.

```
Grant
  id            GrantID
  subject       Subject
  actions       []Action        e.g. workload.start, volume.read, console.write
  scope         Scope           Organization | Project | Workload | Volume | Endpoint | ...
  conditions    Conditions
  parent        GrantID?        the grant this was attenuated from; null only for root
  issued_by     Subject
  issued_at     Timestamp
  revoked_at    Timestamp?

Conditions
  expires_at    Timestamp?
  not_before    Timestamp?
  source_cidr   []CIDR?
  requires_mfa  bool
  requires_approval_by  Scope?   // this grant can propose, not apply, see §3.3
  max_uses      int?
```

**Invariants**
- **A grant can never exceed its parent.** Its actions are a subset, its scope is contained, its
  conditions are at least as restrictive. This is checked at issue time and re-checked at use time,
  because a parent may have been revoked in between.
- Revoking a grant revokes its entire subtree, transitively and immediately.
- Every authority in the system traces to a root grant through an unbroken chain. There is no
  ambient authority anywhere, including for administrators.
- "Administrator" is a grant with a wide scope and no expiry. It is not a flag, a boolean column,
  or a branch in the code (P6).

**Lifecycle**

```
issued ──use──▶ issued
   │
   ├──expires_at reached──▶ expired
   ├──max_uses reached────▶ exhausted
   ├──revoked──────────────▶ revoked  (cascades to subtree)
   └──parent revoked───────▶ revoked
```

The operation RBAC cannot express, which this makes trivial:

> A moderator holds `workload.restart` on project *Survival*, expiring in 90 days. They attenuate
> it into a link: `workload.restart` on **one** workload, expiring in 24 hours, `max_uses: 3`, no
> account required. They send it in a Discord DM. It cannot outlive their own grant, and revoking
> theirs kills it.

#### GrantTemplate

> A named, reusable bundle that expands into grants. A role in the user interface, not in the
> model.

```
GrantTemplate
  id            TemplateID
  org           OrgID
  name          string          "Moderator", "Billing integration", "Read-only auditor"
  actions       []Action
  scope_pattern ScopePattern    e.g. "any project in this org", "the workload it is applied to"
  conditions    Conditions      defaults, overridable at issue time
```

The `Grant`-only model in §3.2 is correct and it is unusable on its own. Nobody wants to assemble
an action list by hand every time someone joins a moderation team, and the first thing every
operator will ask for is "give this person the same access as that person."

A template answers that without reintroducing roles. Applying a template **issues ordinary grants**
and then disappears from the picture:

- The issued grants are normal grants. They attenuate, expire, revoke, and audit like any other.
- They carry no back-reference to the template. Editing a template later does not silently widen
  authority that has already been handed out, a property RBAC does not have, and the source of a
  whole class of production accidents.
- Authorization code never sees a template. It sees grants. There is exactly one authorization path
  (P6).

Templates are convenience at the edge. The model stays pure; the interface stays usable.

#### Identity provider binding

```
IdentityBinding
  provider      discord | oidc | ...
  external_id   string        // guild id, role id, subject claim
  subject       Subject       // the ExternalIdentity this materializes
  verified_at   Timestamp
```

---

### 3.3 The workload and its state

This is the centre of the model.

#### Workload

> The identity of a thing that runs. Deliberately almost empty: it is a stable name to which state
> attaches.

```
Workload
  id            WorkloadID     immutable, never reused
  project       ProjectID
  name          string         mutable, unique within project
  labels        map[str]str
  intent        IntentID       the current declared state
  observation   ObservationID? the most recent observed state, null if never observed
  created       Timestamp
  deleted_at    Timestamp?     soft; the id is retained forever
```

**Invariants**
- `id` is never reused, even after deletion. Logs, backups, grants, and metering records outlive
  the workload and must never be reattributed to a different one.
- `name` is a label for humans and may be changed or reused. Nothing references a workload by name.

#### Intent

> What someone declared should be true. Immutable and versioned.

```
Intent
  id            IntentID       immutable
  workload      WorkloadID
  version       int            monotonic per workload
  parent        IntentID?      the intent this was derived from
  declared_by   Subject
  declared_at   Timestamp

  // --- the declaration itself ---
  recipe        RecipeRef      name + version + digest (see 09-recipes.md)
  lifecycle     persistent | task | scheduled
  schedule      CronExpr?      required iff lifecycle = scheduled
  state         running | stopped   // ignored for task and scheduled
  console       none | logs | interactive
  runtime       RuntimeSpec    driver + isolation tier + driver-specific config
  resources     ResourceSpec   cpu, memory, disk, iops, bandwidth
  volumes       []VolumeSpec
  endpoints     []EndpointSpec
  health        HealthSpec
  config        map[str]Value  validated against the recipe's schema (Rule K-17)
  placement     PlacementSpec  constraints, affinities, preferred region
```

**Invariants**
- An `Intent` is **never mutated**. Changing anything creates a new `Intent` with `version + 1` and
  `parent` set to the previous one.
- `config` is validated against the recipe's declared schema at declaration time. Invalid intents
  are rejected before they exist, not discovered at apply time.
- Every intent records who declared it. There is no anonymous change.

Because intents are immutable and chained, the full history of a workload's declared state is a
linked list, and rollback is "declare intent N again" rather than an inverse operation (P9).

#### Observation

> What the node agent actually sees. Never merged with `Intent`.

```
Observation
  id            ObservationID
  workload      WorkloadID
  node          NodeID
  observed_at   Timestamp
  intent_seen   IntentID       which intent the agent believes it is converging toward

  state         running | stopped | starting | stopping | crashed | unknown
  health        healthy | unhealthy | unknown
  exit_code     int?
  runtime_id    string         container/VM id on the node
  resources     ResourceUsage
  endpoints     []EndpointStatus
  volumes       []VolumeStatus
  since         Timestamp      when the state last changed
```

**Invariants**
- `Observation` is written only by the node agent. `Intent` is written only by the control plane.
  They are separate objects with separate writers and separate trust levels, merging them into one
  spec/status record invites exactly the "the panel says stopped but it is running" class of bug
  (P1).
- When the agent is unreachable, `state` is `unknown`. Korpis never reports a remembered value as a
  current one (P4).
- `intent_seen` makes convergence lag visible: if it trails `Workload.intent`, the workload has not
  caught up yet, and the UI can say so precisely.

#### Plan

> The computed difference between an intent and reality, persisted, inspectable, approvable.

```
Plan
  id            PlanID
  workload      WorkloadID
  from_intent   IntentID?      current
  to_intent     IntentID       proposed
  computed_at   Timestamp
  computed_by   Subject

  steps         []Step
  impact        Impact         disruptive?, downtime estimate, data at risk, cost delta
  status        proposed | approved | rejected | applying | applied | failed | expired | superseded
  expires_at    Timestamp
  approved_by   Subject?
  approved_at   Timestamp?

Step
  action        create_volume | pull_recipe | stop | migrate | reconfigure_network | ...
  target        object reference
  reversible    bool
  detail        map[str]Value
```

**Invariants**
- Every change to a workload's **declared** state passes through a `Plan`. There is no code path
  that produces a new `Intent` without producing one (P5).
- A plan is computed against a specific `from_intent`. If reality moves underneath it, the plan is
  marked `superseded` and must be recomputed, a plan can never be applied against a state it was
  not computed for.
- Plans expire. A stale plan is not an approvable plan.
- A plan containing a step marked `reversible: false`, deleting a volume, releasing an IP, always
  requires explicit approval. A plan whose steps are all non-disruptive is auto-approved by default
  policy.

**Plans govern intent changes, not drift correction**

There is a tension here that has to be resolved explicitly, because getting it wrong breaks either
P5 or the system's viability.

The reconciler runs continuously. A crashed process gets restarted; a container that vanished gets
recreated; a firewall rule that drifted gets rewritten. If each of those produced a `Plan`, a node
running two hundred workloads would generate a constant stream of plan objects that nobody reads,
and the audit trail would be noise. If none of them produced anything, there would be two mutation
paths and P5 would be decorative.

The resolution is that these are different operations:

| | Produces a Plan | Authorized by |
|---|---|---|
| **Intent change**: a subject declares something new | yes, always | the subject's grant, at declaration |
| **Drift correction**: reality diverged from an intent already approved | no | the approval of that intent |

Converging toward an approved intent is not a new decision. It is the execution of a decision
already made and already audited. It still emits `Effect`s (every restart, recreate, and repair is
recorded) but it does not require a new plan, because nobody is choosing anything.

This is also what keeps the interactive path fast. Pressing Start changes `state: stopped →
running`, which is an intent change, so it produces a plan. But that plan's steps are
non-disruptive, so default policy auto-approves and applies it in the same call. The user
experiences one click; the system still has a complete Intent → Plan → Effect record. The plan
object is not a round trip, it is a receipt.

This single object is Bet 3, and it produces as consequences: dry-run (compute the plan, do not
apply it), approval workflow (`requires_approval_by` on a grant), scheduled change (approve now,
apply in a window), rollback (plan toward an older intent), and honest audit (the plan and its
effects are the record).

It is also what makes a chat surface safe to grant authority to. A Discord bot posts the plan with
its impact assessment, someone with the right grant presses Approve, and the audit trail is
complete, without anyone holding standing authority to mutate production from a chat message.

#### Effect

> What actually happened. Append-only.

```
Effect
  id            EffectID
  plan          PlanID?        null for events not originating from a plan
  step          int?
  workload      WorkloadID?
  node          NodeID?
  actor         Subject
  grant         GrantID?       the grant that authorized it
  at            Timestamp
  action        string
  outcome       succeeded | failed | skipped | rolled_back
  error         Error?
  before        map[str]Value?
  after         map[str]Value?
```

**Invariants**
- Append-only. Never updated, never deleted. Retention is by archival, not mutation.
- Every effect names the **grant** that authorized it, not merely the actor. "Who did this" is
  answerable; so is "under what authority, delegated from whom".

Audit is not a separate subsystem writing a parallel log that can drift from reality. The effect
stream *is* the record of what happened.

---

### 3.4 Placement

#### Node

> A machine running the Korpis agent.

```
Node
  id            NodeID         immutable
  name          string
  labels        map[str]str    region, rack, arch, os, gpu, storage-class, ...
  taints        []Taint        workloads must tolerate these to land here
  capacity      Resources      total
  allocatable   Resources      capacity minus system reserve
  allocated     Resources      sum of placed intents
  drivers       []DriverInfo   which runtime drivers this node offers, with capabilities
  status        ready | cordoned | draining | unreachable | unknown
  agent_version string
  last_seen     Timestamp
```

**There is no `Location` object.** Pterodactyl has Nodes inside Locations; Pelican replaced its
nest/location hierarchy with tags and was right to. Region, rack, datacentre, and provider are
**labels**, and placement uses label selectors. A rigid two-level hierarchy cannot express "same
rack", "different power feed", "has NVMe", or "EU only", a label selector expresses all of them.

`drivers` carries **declared capabilities**, borrowed from Nomad's task driver interface: whether a
driver can send signals, exec into a running workload, attach an interactive console, hot-resize,
snapshot, or live-migrate. The scheduler asks; it never assumes (Rule K-9).

**Lifecycle**

```
registering ──▶ ready ⇄ cordoned ──▶ draining ──▶ (empty) ──▶ decommissioned
                  │
                  └──▶ unreachable ──▶ ready  (on reconnect)
```

`cordoned` means no new placements; existing workloads keep running. `draining` means migrate
everything off. Both are real states with real behaviour, not admin conveniences; they are what
makes hardware maintenance possible without downtime.

#### Placement

> The binding of a workload to a node.

```
Placement
  workload      WorkloadID
  node          NodeID
  bound_at      Timestamp
  bound_by      scheduler | operator | migration
  reason        string          human-readable explanation of why this node
```

**Invariants**
- A persistent workload has at most one active placement.
- During migration two placements coexist, exactly one marked authoritative, until cutover (Rule
  K-4).
- Every placement records **why**. A scheduler that cannot explain its choice is a scheduler nobody
  trusts. Detail in `05-scheduling.md`.

---

### 3.5 Definition

#### Recipe

> A signed, versioned, content-addressed package describing how to run a class of workload. The
> replacement for the egg.

```
Recipe
  ref           name + version    e.g. korpis.io/minecraft/paper:1.21.4
  digest        sha256            content address; the true identity
  image         OCIRef            the runtime image
  preset        PresetSpec        default lifecycle, console, endpoints, volumes, health
  config_schema JSONSchema        typed settings with constraints and per-field permissions
  install       []InstallStep     restricted DSL, fetch(url, sha256) / extract / template / chmod
  templates     []TemplateSpec    config file rendering, sandboxed variable set
  probes        HealthSpec        readiness and liveness appropriate to the shape
  signatures    []Signature       cosign
```

**Invariants**
- Identity is the **digest**, not the name. `name:version` is a mutable pointer resolved through a
  lockfile at declaration time; the intent stores the digest. The same intent produces the same
  bytes forever (Rule K-15).
- `install` steps come from a restricted vocabulary. **There is no step that runs arbitrary code.**
  Every fetch declares its expected hash. This is the direct answer to eggs being bash scripts that
  download from the internet at install time, non-reproducible and a supply-chain hole.
- `templates` render in a sandbox with an explicitly allow-listed variable set. There is no ambient
  scope (Rule K-2; this is the exact mechanism behind critical advisory GHSA-pfvc-3p5h-x7h6, where
  egg templating could be made to render the daemon token and registry credentials into a
  tenant-readable file).
- `config_schema` drives generated forms, server-side validation, and per-field permissions. Raw
  environment variables and a raw text editor are not the primary interface (Rule K-17).

Distribution is through OCI registries: content addressing, signing, mirroring, and CDN already
exist there. Korpis does not run a store (§2 of `00-overview.md`).

Detail in `09-recipes.md`.

---

### 3.6 Attached resources

#### Volume

> Durable state with a lifecycle independent of the workload using it.

```
Volume
  id            VolumeID
  project       ProjectID
  name          string
  size          bytes
  class         StorageClass     local-zfs | local-btrfs | local-ext4 | network | ...
  node          NodeID?          null for network-backed classes
  attached_to   WorkloadID?
  mode          rw | ro | shared
  snapshots     []SnapshotRef
```

**Invariants**
- A volume outlives the workload attached to it. Deleting a workload does not delete its volumes;
  that is a separate, explicitly irreversible plan step.
- The storage class determines snapshot capability, and snapshot capability determines backup
  strategy (Rule K-5). Choosing ext4 is choosing project quotas and slow backups; choosing ZFS is
  choosing instant snapshots and cheap incrementals. The model surfaces that choice rather than
  hiding it.

#### Endpoint

> A way the network reaches a workload. The replacement for the allocation.

```
Endpoint
  id            EndpointID
  workload      WorkloadID
  name          string           "game", "rcon", "http", "query"
  protocol      tcp | udp | http | https
  container_port  int
  exposure      node_port | ingress | overlay_only | dedicated_ip
  address       Address?         assigned; shape depends on exposure
  domains       []Domain         for ingress exposure
  tls           TLSSpec?         certificate source and policy
  policy        NetworkPolicy    ingress/egress rules, rate limits
  accounting    bool             meter traffic on this endpoint
```

Pterodactyl's `Allocation` is a bare `(ip, port)` row on a node, which is why it has no ingress, no
TLS, no service discovery, no egress control, and no traffic accounting, and why the allocation
rework is still on Pelican's *planned* list years after the fork. `(ip, port)` is load-bearing for
the entire schema, so it cannot be replaced by anyone who inherited it.

`Endpoint` carries the exposure mechanism, the domain, the TLS policy, the network policy, and the
accounting flag as first-class fields. A game server's UDP port and a web app's HTTPS ingress are
the same object with different values (P7). Detail in `07-networking.md`.

#### Stream

> A durable, addressable channel of output from a workload.

```
Stream
  id            StreamID
  workload      WorkloadID
  kind          console | log | event | metric
  retention     Duration | bytes
  indexed       bool
  segments      []SegmentRef     content-addressed, offloadable to object storage
```

**Invariants**
- Streams are durable and seekable, not ephemeral websocket pipes. Reconnecting resumes at an
  offset; it does not restart from an in-memory ring buffer.
- A stream can be shared by grant: a bounded time range, read-only, to a subject who has no other
  authority. Attaching a crash log to a support ticket is a grant, not a copy-paste.
- Streams outlive the workload. A crashed workload's final output is still there after it is gone.

Detail in `14-streams.md`.

---

### 3.7 Extension

> A separate process that adds capability, holding a scoped grant and speaking a versioned
> contract.

```
Extension
  id            ExtensionID
  name          string
  version       SemVer
  api_version   string           the contract version it speaks
  grant         GrantID          its authority, narrow, revocable, auditable
  subscribes    []EventType
  routes        []RoutePrefix    served under /ext/<name>
  ui_slots      []SlotSpec       declared mount points in the web client
  status        running | stopped | failed | incompatible
```

**Invariants**
- An extension **cannot modify Korpis' own files.** It is a process with a token, not a script with
  filesystem access (Rule K-6, the direct answer to the Blueprint framework's `sed`-patching of
  Pterodactyl core files).
- An extension's effects appear in the effect stream attributed to its grant, exactly like a
  human's.
- Core features use this same mechanism. If the contract is insufficient for something core needs,
  the contract is extended, core is never given a private door (P8).

Detail in `16-extensions.md`.

---

## 4. How it fits together

```
Organization ──┬─▶ Organization        (recursive: this is reselling)
               │
               ├──▶ Quota              (inherited, enforced, metered)
               │
               └──▶ Project ──▶ Workload
                                  │
                                  ├──▶ Intent  ──▶ Intent ──▶ …   (immutable chain)
                                  │       │
                                  │       └──▶ RecipeRef ──▶ Recipe @digest
                                  │
                                  ├──▶ Observation                 (written by the agent)
                                  │
                                  ├──▶ Placement ──▶ Node
                                  │
                                  ├──▶ Volume[]     Endpoint[]     Stream[]
                                  │
                                  └──▶ Plan ──▶ Effect[]           (append-only)

Subject ──▶ Grant ──▶ Grant ──▶ …        (attenuation chain; every Effect names one)
   ▲
   └── User | Token | ExternalIdentity | Workload | Extension
```

Two loops define the system:

**The convergence loop.** `Intent` is declared → the control plane computes a `Plan` → the plan is
approved → the agent applies it and emits `Effect`s → the agent reports `Observation` → the control
plane compares observation against intent → if they differ, a new plan. The loop never stops, and
nothing outside it changes durable state.

**The authority chain.** Every `Effect` names the `Grant` that permitted it. Every `Grant` names
its parent. Follow the chain from any action taken in the system to a root grant, and you have the
complete answer to *who allowed this, and who allowed them*.

---

## 5. Naming and identity rules

1. **IDs are immutable and never reused.** Names are mutable and may be reused. Nothing internal
   references anything by name.
2. **Deletion is soft at the identity layer.** IDs persist forever so that effects, streams,
   backups, and metering records are never reattributed to a different object.
3. **Content-addressed things are identified by digest.** Recipes, stream segments, backup chunks.
   Human-readable names are pointers resolved at declaration time and recorded as digests.
4. **No object carries a `type` discriminator that changes behaviour in core.** Behaviour comes
   from declared capabilities, a driver's `DriverInfo`, a recipe's `preset`, an extension's
   `api_version`. Core asks; it does not switch on kind.

---

## 6. What is deliberately absent

- **No `Server`.** The word does not appear in the model. It is ambiguous between the machine and
  the workload, and it is what trapped Pterodactyl in the game vertical.
- **No `Location` or `Nest`.** Labels and selectors.
- **No `Allocation`.** `Endpoint`.
- **No `Egg`.** `Recipe`, content-addressed and signed.
- **No `Role`.** `Grant`.
- **No `is_admin` flag.** A wide-scoped grant.
- **No `spec`/`status` on one object.** `Intent` and `Observation`, separate writers.
- **No game concept anywhere.** Query protocols, RCON, modpacks, wipes, and map rotation are recipe
  and extension concerns (§2 of `00-overview.md`).

---

## 7. Open questions

Recorded rather than resolved. Each is settled in the document named.

1. **Does `Plan` cover multi-workload changes?** Draining a node touches many workloads. Is that
   one plan with many targets, or many plans under a parent operation? → `05-scheduling.md`
2. **What is the storage engine for `Effect`?** Postgres append-only table, or a separate log with
   Postgres as projection? The answer determines whether cheap time-travel is available. Note that
   an earlier design proposal built the job queue on `LISTEN/NOTIFY` and `SKIP LOCKED` while also
   claiming to keep a SQLite path open, those are contradictory and the contradiction must be
   resolved here, not deferred. → `03-state.md`
3. **Can a `Grant` name a set of workloads by label selector**, or only by explicit ID? Selectors
   are far more useful and considerably harder to reason about when the set changes after issue. →
   `08-identity.md`
4. **Where do `task` and `scheduled` workloads keep their run history?** A `scheduled` workload
   produces many runs. Are runs workloads, or a separate `Run` object? → `05-scheduling.md`
5. **Does `Quota` allow overcommit?** Hosting providers depend on it. Which dimensions may be
   oversubscribed, and what happens under contention? → `05-scheduling.md`
6. **How are `Observation`s retained?** Every observation forever is unaffordable; only the latest
   loses the history that makes incidents explicable. → `15-observability.md`
7. **Can a `Quota` be scoped to a label selector?** As written, `Quota` is a flat resource bound
   per organization. A hosting provider sells "4 GB in the EU", capacity in one region is not
   interchangeable with capacity in another, so a single global bound cannot express what is
   actually being sold. Either `Quota` gains a selector dimension, or an organization holds a set
   of selector-scoped quotas. This is a real gap in the model, not a detail. → `05-scheduling.md`
8. **How does one workload address another?** The model gives a workload `Endpoint`s and an
   `overlay_only` exposure, but no name resolution. A web service needs to reach its database
   without either knowing the other's placement or hard-coding an IP that changes on migration.
   This requires either service discovery as a core concern or an explicit dependency declaration
   on `Intent`. Nothing in the current model provides it. → `07-networking.md`
