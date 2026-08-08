# Architecture and the Reconciliation Protocol

**Status:** design **Date:** 2026-08-07 **Depends on:** [`00-overview.md`](./00-overview.md),
[`01-model.md`](./01-model.md)

---

## 1. Why this document matters more than the others

Almost everything in Korpis can be changed later. Storage engines can be swapped, UIs rewritten,
schedulers replaced, drivers added. One thing cannot: **the protocol between the control plane and
the data plane.** Once agents are deployed on machines Korpis does not control, that contract is
frozen for as long as those machines exist.

Pterodactyl's Panel and Wings are coupled by version rather than by contract. Issue #141. "support
for third-party daemons", has been open since October 2016, and ten years later there is still
exactly one implementation. That is not neglect; it is what happens when a protocol is never
designed as a protocol.

Rule K-7 requires that a third party be able to write a conforming agent without reading Korpis'
source. This document is what makes that possible.

---

## 2. The split

```
┌─────────────────────────────────────────────────────────────────┐
│  CLIENTS                                                        │
│  web · Discord · CLI · API consumers · extensions               │
│  Peers. None privileged. All speak the same API. (P2)           │
└───────────────────────────┬─────────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────────┐
│  CONTROL PLANE                                                  │
│                                                                 │
│  Owns: Organization, Project, Quota, Grant, Workload,           │
│        Intent, Plan, Recipe refs, Placement, Lease              │
│  Reads: Observation, Effect                                     │
│                                                                 │
│  Decides. Never commands.                                       │
└───────────────────────────┬─────────────────────────────────────┘
                            │  one long-lived stream per node,
                            │  dialled OUT by the agent
┌───────────────────────────▼─────────────────────────────────────┐
│  DATA PLANE, one agent per node                                │
│                                                                 │
│  Owns: Observation, runtime objects, local content store,       │
│        volumes, network namespaces, stream segments             │
│  Reads: Intent, Lease                                           │
│                                                                 │
│  Converges. Never waits to be told.                             │
└─────────────────────────────────────────────────────────────────┘
```

**The ownership rule is absolute.** Every object in `01-model.md` has exactly one writer. The
control plane never writes an `Observation`; the agent never writes an `Intent`. There is no object
both may modify. This is what makes "the panel says stopped but the server is running" structurally
impossible rather than merely unlikely (P1).

---

## 3. Connection direction: the agent dials out

**The agent opens a single long-lived connection to the control plane. The control plane never
connects to a node.**

Pterodactyl works the other way: the Panel makes HTTP calls to Wings, so every node needs a
reachable address, an open inbound port, and a certificate. That requirement is invisible in a
datacentre and crippling everywhere else.

Dialling out gives, all from one decision:

- **Nodes need no inbound ports.** No public IP, no port forwarding, no inbound firewall rule.
- **NAT and residential connections work.** A home machine, a machine behind CGNAT, a machine in
  someone else's cluster, all are ordinary nodes.
- **The firewall story is one sentence:** the node must be able to reach the control plane on 443.
- **Authentication is one-directional.** The agent proves who it is on connect. The control plane
  never needs credentials for the node.
- **Console, logs, intents, observations, and effects all multiplex over one connection.** There is
  no second channel to secure, monitor, or debug.

The cost is that the control plane holds many long-lived connections. That is a well-understood
engineering problem with well-understood solutions, and it is a far better problem than requiring
every operator to expose every node to the internet.

---

## 4. The protocol

### 4.1 Level-triggered, never edge-triggered

This is the single most important decision in the document.

The control plane does **not** send "start workload X". It sends **the complete set of intents
assigned to this node**. The agent's job is to make reality match that set, creating what is
missing, removing what should not be there, and correcting what has drifted.

Edge-triggered systems ("start X", "stop Y", "delete Z") are correct only if no message is ever
lost, reordered, or duplicated. Every real network violates all three. When an edge-triggered
message is dropped, the state divergence is permanent: nothing will ever resend "start X", because
from the sender's perspective it already happened.

Level-triggered systems self-heal. A dropped message is repaired by the next sync, because every
sync carries the whole truth rather than a delta from an assumed prior state.

The concrete consequence for a workload: **there is no `restart` command in the protocol.** Restart
is `Intent.state: running` plus a change to a `restart_epoch` field. The agent sees an intent it
has not converged to and converges. If the message is lost, the next sync carries the same intent
and the same convergence happens. Nothing is missed and nothing is done twice.

Deltas are permitted as a bandwidth optimization, but only as a compression of a level-triggered
stream: every delta names the version it applies to, and a mismatch triggers a full resync. The
agent is never required to have received any particular earlier message.

### 4.2 The stream

One bidirectional stream, multiplexed.

**Control plane → agent**

| Message | Purpose |
|---|---|
| `AssignedIntents` | the complete set of intents placed on this node, with versions |
| `IntentDelta` | optimization only; carries the base version it applies to |
| `LeaseGrant` | authority to run a specific workload, with an epoch and an expiry |
| `LeaseRevoke` | immediate withdrawal, used at migration cutover |
| `StreamAttach` | a client wants console/log output; open this stream |
| `StreamInput` | console input from a client, carrying the authorizing grant |
| `ContentHint` | recipe digests likely to be needed soon; prefetch |
| `Probe` | liveness check with a nonce |

**Agent → control plane**

| Message | Purpose |
|---|---|
| `Hello` | agent version, protocol version, driver capabilities, node capacity |
| `Observation` | current observed state of one workload |
| `NodeStatus` | capacity, allocation, pressure signals, driver health |
| `Effect` | something happened, with outcome and error |
| `StreamData` | console/log output segments |
| `LeaseRenew` | keep-alive for held leases |
| `ContentStatus` | which recipe digests are present locally |
| `Ack` | version acknowledged and converged to |

### 4.3 Convergence is measurable

Every `Intent` carries a monotonic `version`. Every `Observation` carries `intent_seen`, the
version the agent believes it is converging toward.

That single field makes the whole system honest:

- `intent_seen == Workload.intent.version` and `Observation.state == Intent.state` → converged
- `intent_seen < Workload.intent.version` → the agent has not received or not yet applied the
  latest declaration. The UI says "applying", with the exact lag.
- `intent_seen == version` but states differ → the agent has the intent and cannot satisfy it. This
  is a real failure and is surfaced as one, not as a spinner.
- No observation within the staleness window → `unknown`. Never a remembered value presented as
  current (P4).

### 4.4 Adoption: a restarting agent must not kill anything

When the agent starts (after a crash, an upgrade, or a reboot), running workloads must survive.

Every runtime object the agent creates is labelled at creation with `workload_id`,
`intent_version`, and `lease_epoch`. On startup the agent:

1. Enumerates all runtime objects across all drivers.
2. Matches them to assigned intents by `workload_id`.
3. **Adopts** matches: takes over supervision without touching the process.
4. Reconciles differences: if the adopted object was created from an older intent version, apply
   the difference; if it can only be corrected by recreation, that is a step with a `disruptive`
   flag.
5. Reports objects with no matching intent as orphans. It does **not** delete them automatically,
   orphan removal is a plan step requiring approval, because "the control plane forgot about it"
   and "this should be deleted" are indistinguishable from the node's point of view, and one of
   them destroys customer data.

An agent upgrade is therefore a process restart, not a workload restart.

### 4.5 Leases and fencing

**Every running workload requires a valid lease. No lease, no execution.**

```
Lease
  workload      WorkloadID
  node          NodeID
  epoch         int          monotonic, per workload, never reused
  expires_at    Timestamp
  on_expiry     keep_running | stop
```

The control plane issues exactly one valid lease per workload at any time. The epoch increments on
every issue. An agent presenting an older epoch is fenced: its writes are rejected and it is told
to stop.

This is what Pterodactyl lacks, and it is why its transfer failures deadlock. When a transfer is
interrupted, both nodes may believe they own the server, the panel cannot express that state, and
the recovery procedure is truncating a table by hand (issue #4505). With leases, the answer is
mechanical: whoever holds the current epoch owns it; everyone else stops. Migration cutover is
`LeaseRevoke(old_epoch)` then `LeaseGrant(new_epoch)`, and there is no window in which two writers
are both authorized.

**`on_expiry` is a per-workload policy, and it must be, because the correct answer differs:**

- A game server with local storage: `keep_running`. If the control plane is unreachable, players
  are still connected and stopping them serves nobody. A split brain is impossible anyway, nothing
  else can use that local disk.
- A stateful workload on shared or network storage: `stop`. Two writers is data corruption. Losing
  availability is recoverable; losing the database is not.

Defaulting this globally would be wrong in one direction or the other for half of all workloads.

### 4.6 What happens when the control plane is down

This is the payoff of P1, and it should be stated as a guarantee rather than a hope.

| Capability | Control plane down |
|---|---|
| Running workloads | **keep running** |
| Crash restarts, health checks, drift correction | **keep working**, the agent has the intent and does not need permission to execute a decision already made |
| Leases with `on_expiry: keep_running` | honoured until the control plane returns |
| Leases with `on_expiry: stop` | workload stops when the lease expires, by design |
| New intents, plans, placements | unavailable |
| Console and log streaming | unavailable, routed through the control plane |
| Web, CLI, chat, API | unavailable |

The data plane does not depend on the control plane to keep doing its job. That is the difference
between a panel and a platform.

Console unavailability during a control plane outage is a real gap and is treated as one, see §10.

### 4.7 Idempotency

Every step is keyed by `(workload_id, intent_version, step_index)`. Applying a step that has
already been applied is a no-op that reports success. Retry is therefore always safe, which is what
allows every operation to be retried rather than requiring a distinction between "failed" and
"possibly failed" (P9).

---

## 5. Inside the control plane

Components, each with one responsibility and a defined interface. Nothing here reaches around
another component.

| Component | Responsibility |
|---|---|
| **Gateway** | Terminates all client connections. Authenticates the subject. Resolves and evaluates grants. Rejects anything unauthorized before it reaches the model. The single authorization path (P6). |
| **Planner** | Given a proposed `Intent`, computes a `Plan`: the steps, their reversibility, and the impact assessment. Pure, it reads state and produces a plan; it changes nothing. |
| **Scheduler** | Chooses a node for a workload, honouring constraints, quotas, driver capabilities, and placement policy. Records *why*. Detail in `05-scheduling.md`. |
| **Dispatcher** | Owns the agent connections. Publishes assigned intents and leases, ingests observations and effects. Stateless, any replica can serve any agent. |
| **Store** | The authoritative state. Owns intents, plans, grants, and the effect log. Detail in `03-state.md`. |
| **Streams** | Routes console and log data between agents and clients, and persists segments. Detail in `14-streams.md`. |
| **Content** | Resolves recipe references to digests, verifies signatures, tells nodes what to prefetch. Detail in `09-recipes.md`. |

The Planner being pure is deliberate: it makes dry-run free, makes plans testable without a running
cluster, and prevents the class of bug where computing a plan has side effects.

---

## 6. Inside the agent

The agent is where Rule K-1 and P3 are either honoured or lost. Six of twelve published Wings
advisories are one class: a privileged process performing filesystem operations on tenant-supplied
paths, defending itself by inspecting path strings. The structure below is designed so that class
of bug has nowhere to live.

```
┌──────────────────────────────────────────────────────────────┐
│  Supervisor            unprivileged user, no CAP_*           │
│  Holds the stream. Runs the reconciliation loop.             │
│  Computes steps. Never touches tenant files.                 │
└───────┬───────────────────────┬──────────────────────────────┘
        │                       │
        │ unix socket           │ unix socket
        ▼                       ▼
┌──────────────────┐   ┌────────────────────────────────────────┐
│  Privileged      │   │  Runtime drivers                       │
│  helper          │   │  oci · native · kvm · microvm · windows│
│  root, tiny,     │   │  Each declares its capabilities.       │
│  socket-         │   │  Detail in 04-runtimes.md.             │
│  activated       │   └────────────────────────────────────────┘
│                  │
│  Verbs only:     │   ┌────────────────────────────────────────┐
│  mount, unmount, │   │  Filesystem workers. ONE PER TENANT   │
│  cgroup write,   │   │  Runs INSIDE that tenant's mount ns.   │
│  netns create,   │   │  Unprivileged. Landlock-confined.      │
│  volume create,  │   │  openat2(RESOLVE_BENEATH) for every    │
│  snapshot        │   │  path operation.                       │
│                  │   │  Serves the file manager and SFTP.     │
│  No paths from   │   │  Cannot name a path outside the tenant │
│  tenants. No     │   │  root, not by policy, by kernel.      │
│  arbitrary exec. │   └────────────────────────────────────────┘
└──────────────────┘
```

Three properties follow from this structure:

**No component both holds privilege and accepts tenant input.** The supervisor accepts
control-plane input and holds no privilege. The privileged helper holds privilege and accepts only
a fixed verb set with structured arguments, never a tenant-supplied path. The filesystem workers
accept tenant input and hold no privilege, and are confined to a subtree by the kernel, so a
traversal bug produces `EXDEV`, not a host file.

**File operations happen inside the tenant's namespace, not the agent's.** Pterodactyl's Wings
performs file operations for every tenant from one privileged process, which is why a symlink race
in one server's directory can reach the host. In Korpis, the worker serving a tenant is *already
inside* that tenant's mount namespace. There is no host path for it to reach, because the host
filesystem is not in its namespace.

**SFTP and the file manager are the same worker.** They are the two highest-risk surfaces in any
panel, arbitrary paths, arbitrary names, arbitrary sizes, from untrusted users. Giving them one
confined implementation instead of two privileged ones removes the most productive bug source in
this product category.

---

## 7. Transport, identity, and versioning

### Transport

A single bidirectional streaming connection over TLS 1.3, dialled out by the agent. Message framing
is a versioned binary schema, normative definition in `10-api.md`.

### Node identity

An agent authenticates with a per-node key established at enrolment. Enrolment uses a single-use,
short-lived token that is itself an attenuated grant (§3.2 of `01-model.md`), issuing a node token
is not a special code path, it is `node.enroll` scoped to one organization with `max_uses: 1`.

**No node ever receives a credential it does not need.** The critical advisory GHSA-pfvc-3p5h-x7h6
exists because Wings held registry credentials and a daemon token in a scope that tenant-authored
templates could reach. In Korpis, registry credentials are held by the Content component and
exchanged for **short-lived, digest-scoped pull tokens** issued per fetch. A compromised node
yields a token that can pull one already-known artifact for a few minutes.

### Protocol versioning (Rule K-7)

- The protocol is versioned independently of both the control plane and the agent.
- The agent declares its protocol version and driver capabilities in `Hello`. The control plane
  never infers capability from a version number, it asks, following Nomad's task driver interface.
- The control plane supports the current protocol version and the two preceding ones. Support
  windows are published, not discovered.
- Upgrade order is always: control plane first, then agents, one at a time. A mixed fleet is a
  supported state, not an emergency.
- The protocol definition is Apache-2.0 and published separately from the implementation, so that a
  third-party agent is a normal thing to build rather than a reverse-engineering exercise.

---

## 8. High availability

**The control plane is stateless.** Every replica can serve any client and any agent. Agents
connect through a load balancer and reconnect freely; there is no affinity between an agent and a
replica.

**State lives in the store**, which is the only stateful component and the only thing requiring a
replication strategy. Detail in `03-state.md`.

**Leases are held in the store with fencing epochs**, so replica failover cannot produce two
authorized writers for one workload, the epoch check is in the data path, not in a coordination
layer that could be bypassed.

The honest framing: control plane HA matters **less** here than in comparable systems, because §4.6
guarantees the data plane survives a total control plane outage. Losing the control plane costs
management, not availability. That is a deliberate consequence of P1, and it means HA can be added
when it is needed rather than being a precondition for the first release.

---

## 9. Failure modes

Each row is a state the system must be able to represent and recover from without manual database
surgery, the standard Pterodactyl fails (issues #4505, #3332).

| Failure | Behaviour |
|---|---|
| Agent process dies | Workloads keep running. On restart the agent adopts them (§4.4). |
| Agent host reboots | Workloads restart per their intent. Adoption finds what systemd already restarted. |
| Network partition, node side | Agent keeps converging to its last known intents. Leases with `keep_running` persist; `stop` leases expire and the workloads stop. |
| Network partition, control plane side | Node marked `unreachable`. Observations go `unknown`, never a stale value shown as current. No new placements on that node. |
| Node returns after being declared dead | Its leases have expired and been reissued elsewhere. It presents old epochs, is fenced, and stops those workloads. No split brain. |
| Control plane replica dies | Agents reconnect to another replica. No state is lost; the store is authoritative. |
| Store unavailable | Control plane serves reads from cache where safe and rejects writes. Agents keep converging. |
| A plan fails halfway | Applied steps are recorded as effects. The plan is `failed`. The next reconciliation computes a fresh plan from actual current state; there is no partially-applied limbo, because the next plan is computed from observation, not from an assumption about where the last one stopped. |
| Migration interrupted | Source retains the lease until cutover completes and is verified. Interruption leaves the source authoritative and the destination's partial copy is garbage-collected. The workload never stops being owned by exactly one node (Rule K-4). |
| Two agents claim one workload | Impossible. One valid lease epoch exists at a time. |
| Agent is compromised | It holds a node key scoped to its own node, short-lived digest-scoped pull tokens, and no tenant credentials. It cannot read another node's intents, mint grants, or write effects attributed to another actor. |

---

## 10. Open questions

1. **Emergency console during a control plane outage. Resolved: a local socket, not a listener.**
   The framing assumed the operator reaches the agent over the network, which collides with §3, the
   agent has no inbound port, and opening one for an emergency would add permanent attack surface
   for an occasional need.

But if the control plane is down, the operator already has to reach the machine some other way, and
that way is SSH or a physical console. So break-glass is a **unix socket on the node**:
`korpis-agent console <workload>`, authorized by a break-glass token (§6 of `08-identity.md`) or by
being root on the box, with every use recorded locally and replayed to the effect log on reconnect.

No network surface is added, nothing is cached that was not already deliberate, and the honest
limit is stated rather than hidden: **this does not work from a phone.** A mechanism that pretended
otherwise would be a listening socket on every node in the fleet, permanently, for an event that
happens twice a year.

2. **Does the agent cache grants at all? Resolved: no, it verifies tokens, and the distinction is
   the whole answer.** Settled in §6 of `08-identity.md`.

A grant is a policy that must be evaluated against a chain of parents, conditions, and scopes;
caching that on a node would mean putting the authorization engine and its inputs on the data
plane, where a compromised node could reason its way to a conclusion nobody authorized. A
capability token is the *result* of that evaluation (already performed, online, by the control
plane) signed, narrowly scoped, and short-lived.

So the agent holds no grants and evaluates nothing. It checks a signature and an expiry. Revocation
latency equals the token lifetime, which is the honest trade and the reason lifetimes are short.
3. **Full-resync cost at scale.** A node running several hundred workloads receiving the complete
   intent set on every reconnect is expensive. Content-addressed intent sets with digest comparison
   would let the agent say "I already have this exact set", but that adds a second identity concept
   to intents. **Resolved in §9 of `10-api.md`**, a set-level digest, not a per-intent identity.
4. **Multi-workload plans across nodes.** Draining a node produces migrations for many workloads.
   Is that one plan with many steps, many plans under a parent operation, or a distinct `Operation`
   object? This is question 1 of `01-model.md` and it needs settling before the scheduler is
   designed. → `05-scheduling.md`
5. **Should the agent be able to reject an intent?** A node may be unable to satisfy an intent it
   has been assigned, a driver is missing, hardware is absent, a volume will not mount. Currently
   this surfaces as persistent non-convergence. An explicit `Reject` with a reason would let the
   scheduler reschedule immediately instead of waiting for a timeout, but it lets the data plane
   make a decision, which sits uneasily with §2. → `05-scheduling.md`
6. **Observation frequency and cost.** Continuous streaming gives an accurate `since` timestamp and
   fast failure detection at the cost of constant traffic; on-change reporting with periodic
   heartbeats is far cheaper but blurs exactly when something changed. → `15-observability.md`
