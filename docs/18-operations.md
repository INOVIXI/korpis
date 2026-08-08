# Operations

**Status:** design **Date:** 2026-08-07 **Depends on:** [`03-state.md`](./03-state.md),
[`02-architecture.md`](./02-architecture.md), [`05-scheduling.md`](./05-scheduling.md)
**Implements:** Principles P4, P9

---

## 1. The governing constraint

> **A single operator with one machine must be able to install and run Korpis.**

§2 of `00-overview.md` states this as a boundary against Kubernetes' operational cost, and it is
the constraint every decision in this document is checked against. Korpis has a large model; it is
not permitted to have a large *installation*. Every feature here must degrade to one machine, one
operator, and no cluster.

The corollary matters as much: the same binaries, in the same configuration model, scale to a
fleet. There is no "single-node mode" that is a different product with a different upgrade path,
because that path is how a project ends up with a hobbyist edition nobody tests and a production
edition nobody can afford.

---

## 2. Install, honestly

Two binaries and a database:

```
korpis-server    control plane. HTTP/Connect API, reconcilers, scheduler.
korpis-agent     node agent. dials out; no inbound ports.
PostgreSQL       authoritative state (§6 of `03-state.md`. Postgres only, no SQLite, no MySQL).
```

On one machine, all three run on one machine. That is the entire single-node story.

**The database is the one place Korpis cannot claim "just one binary", and it says so rather than
pretending.** §6 of `03-state.md` killed SQLite deliberately: the transactional guarantees that
make `Intent`/`Effect` honest are not optional, and an embedded engine that cannot provide them
would undermine the model in exchange for a nicer README. The installer provisions Postgres through
the system package manager or a container and gets out of the way; Korpis does not vendor, fork, or
supervise a database it did not write.

### The binary is its own installer

> Resolves open question 1 of §10.

```
korpis-server install
```

Download one signed binary, verify it against the published digest, run it. It runs preflight, asks
where PostgreSQL should come from rather than assuming, generates the signing keys, writes the
systemd units, and prints the join command for the first node. Then it is the same binary that runs
the control plane, there is no separate installer artifact to drift out of sync with what it
installs.

**There is no `curl | sh`.** It is what everyone in this market ships and it is inconsistent with
being a project whose recipes reject a `fetch` without a hash (§4 of `09-recipes.md`), whose
releases are reproducible and signed (§11 of `17-security.md`), and which tells operators to verify
artifacts by digest. A one-liner that pipes an unverified script into a shell teaches the opposite
of every other thing Korpis asks of its users. Download, verify, run is three commands instead of
one, and the three are honest.

A container-compose quick-start also exists for evaluation. It **says on startup that it is not a
production configuration** and why, an unsupervised database in a container is precisely the setup
nobody tests and everybody eventually runs. Labelling it is P4 applied to our own packaging.

Distribution packages (`deb`, `rpm`) with a signed repository follow when there is someone to
maintain them. They are a better answer than the installer for operators who already manage
machines with configuration management, and a worse one to start with, because a stale package is a
support burden that outlives the person who built it.

### Preflight

`korpis-agent preflight` reports what a host is missing, kernel version, cgroup v2, user
namespaces, Landlock availability, the filesystem features a storage class needs, and prints
exactly what to run to fix each. **It does not silently modify the host.** An installer that loads
kernel modules and rewrites `sysctl` on someone's machine is a configuration management system, and
Korpis is not one.

### What a node actually requires

Requirements are a function of the isolation tiers and storage classes a node offers, not a single
number, a node that only runs `process`-tier workloads on ext4 needs far less than one offering
microVMs on ZFS. Preflight reports against what the node is *configured to offer*.

| | Required for | Notes |
|---|---|---|
| Linux kernel **6.1+** | everything | 5.13 has Landlock at all; 6.1 LTS is the realistic floor, and later ABI versions confine more |
| cgroup v2, unified hierarchy | every tier | the enforcement path for CPU, memory, PID, and I/O (§10 of `17-security.md`) |
| user namespaces enabled | every tier | root inside a workload must not be root on the host |
| `openat2` (5.6+) | tenant file access | subsumed by the kernel floor |
| Landlock | tenant file access | the filesystem worker's confinement; without it the node offers no tenant file API |
| **KVM + nested virt** | `microvm`, `vm` | absent → the node advertises only `process` and `container` |
| ZFS or btrfs | replicated and snapshot-capable storage classes | ext4/XFS with project quota still gives enforced byte and inode limits |
| IPv6 | the default addressing mode (§4 of `07-networking.md`) | absent → IPv4 only, and the scarcity is the operator's problem |

**A node that lacks something offers less; it does not fail to start, and it never silently offers
what it cannot enforce.** Missing KVM removes two isolation tiers from what the scheduler will
place there (§4.1 of `04-runtimes.md`, capabilities are declared, never inferred). Missing Landlock
removes the tenant file API. This is the same discipline as everything else: the capability is
advertised or it is not, and nothing in between.

Sizing is ordinary and deliberately not prescribed, a node's capacity is whatever `allocatable`
says after reservations, and §5.2 of `05-scheduling.md` already makes that an honest number. The
one non-obvious figure is memory for `microvm` tiers, where per-workload overhead is real and is
the reason hibernation (§6 of `22-first-party.md`) exists.

---

## 3. Enrolling a node

The agent dials out (§4 of `02-architecture.md`), so enrollment opens nothing:

```
korpis node token create --label fra-3 --expires 1h
→ korpis-agent join --server korpis.example --token nj_8f2a…
```

The join token is single-use and short-lived. The agent presents it once, receives a per-node key
pinned to the control plane's certificate, and never uses the join token again. The operator's
firewall requires no inbound rule, NAT is irrelevant, and a node behind a residential connection is
an ordinary node rather than an unsupported configuration.

Removing a node is the reverse and is never abrupt: cordon, drain, decommission (§3 of
`05-scheduling.md`), each producing a Plan that names every workload that will move and where it
will land.

---

## 4. Upgrade

This is where the incumbents break. Pterodactyl's extension ecosystem patches core files with
`sed`, so an upgrade is a coin flip; its transfers deadlock into states requiring manual SQL; and
its agent upgrade restarts tenant workloads.

Four rules, each of which is a property of the architecture rather than a procedure to remember:

**Control plane first, then agents, one at a time.** The N-2 support window of §4 of `10-api.md`
means the control plane always speaks the version every deployed agent speaks.

**A mixed-version fleet is a supported steady state.** Not a window to rush through. An operator
who upgrades the control plane on Monday and gets to the last three nodes in a fortnight is
operating normally, which is what makes cautious upgrading possible for someone who has other work.

**Upgrading an agent does not restart tenant workloads.** This is the single most valuable
operational property in the design and it is why `Recover` is mandatory rather than optional in §4
of `04-runtimes.md`. The agent stops, the new binary starts, re-adopts the running workloads by
their `workload_id` / `intent_version` / `lease_epoch` labels (§4.4 of `02-architecture.md`),
reconciles, and continues. A game server with forty players on it does not notice that its
supervisor was replaced.

**Migrations are forward-only, online, and reversible for one version.** No long-held locks, no
schema change that takes a table offline, and every migration accompanied by the inverse needed to
roll the binary back within the support window. The `Intent` chain is immutable and append-only
(§3.1 of `03-state.md`), so rolling *configuration* back is a different and easier operation: it is
re-declaring an earlier intent version, which produces a Plan and converges. Bet 3's payoff.

---

## 5. High availability, and what it actually covers

**Control plane.** `korpis-server` holds no state that is not in Postgres, so availability is a
matter of running several. There is **no leader**: reconciliation work is claimed from a queue with
`SELECT … FOR UPDATE SKIP LOCKED` (§6 of `03-state.md`), so N replicas share the work, a replica
that dies has its claims reclaimed when they expire, and adding capacity is adding a process. The
few genuinely singleton operations (schema migration, advancing the `max_issued_epoch` watermark)
take a Postgres advisory lock and are brief.

Avoiding leader election here is deliberate. Leader election is where distributed systems acquire
their most interesting failure modes, and this workload does not need it.

**Postgres.** Korpis does not implement database high availability. Patroni, a managed service, or
accepting single-instance risk are all valid; Korpis states the requirement, synchronous durability
for the transaction that writes an `Effect`, and leaves the rest to tools built for it.

**Nodes.** A node is a single point of failure for the workloads on it, and no amount of
control-plane availability changes that. What Korpis provides is the ability to *express* the
answer: replicas, anti-affinity across nodes and failure domains, a replicated storage class, and a
`stable` endpoint whose public address survives the move (§3 of `07-networking.md`). What it does
not provide is automatic recovery of stateful workloads whose only copy of their data was on the
failed node, because that is not recoverable, and implying otherwise would be the kind of claim P4
forbids.

**Edges.** `stable` and `ingress` endpoints (§3 of `07-networking.md`) concentrate many workloads'
reachability into one forwarding point, which reintroduces at the network layer exactly the shared
failure domain the control plane design removed. **This is the cost of the headline feature and it
must be stated rather than discovered:** in Pterodactyl a node's failure takes down that node's
servers and nothing else, because the address lives on the node. With `stable`, an edge's failure
takes down every endpoint behind it, including workloads that are running perfectly on healthy
nodes.

So an edge is never a single machine in any deployment that cares:

| Approach | Failover | Demands |
|---|---|---|
| VRRP / floating address | seconds, within one L2 segment | simplest; single-site only |
| ECMP with equal-cost routes | sub-second, per-flow rehash | a router that speaks it |
| BGP anycast | route withdrawal | an ASN, transit that permits it, real network operations |

Forwarding state is derivable from placements, so a replacement edge reconstructs it from the
control plane rather than needing state replication, an edge holds no truth of its own. On one
machine, the edge is that machine and the question does not arise; the single-machine constraint of
§1 is not violated by an option that only matters at scale.

**The control plane can be entirely absent and workloads keep running.** §4.6 of
`02-architecture.md` guarantees it, and it is the reason disaster recovery here is a recovery
rather than an outage: agents hold their intents and their leases, and a lease's `on_expiry` policy
is `keep_running` by default.

---

## 6. Disaster recovery

Two different things, frequently conflated:

**Tenant data** (volumes, snapshots, backups) is `06-storage.md`, content-addressed, deduplicated,
client-side encrypted, restorable to a new volume.

**Korpis itself** is this section, and it is small:

| Must be backed up | Why |
|---|---|
| the Postgres database | all intents, grants, effects, placements |
| the control plane's signing keys | grants and capability tokens verify against them |
| the `max_issued_epoch` watermark | without it, restore cannot fence safely |

The restore procedure has **two** steps that cannot be skipped, and §7 of `03-state.md` makes both
mandatory rather than documented. They are the same principle applied to the two kinds of authority
the control plane hands out: *a restored control plane must invalidate authority it can no longer
account for.*

> **1. Advance the lease epoch past every epoch it may have issued before the backup was taken.**

A backup is by definition older than the failure, so it contains a stale view of which epochs are
outstanding. A restored control plane that resumes issuing from the backup's watermark will hand
out an epoch a live agent already holds, and two authorities believing they own the same workload
is the split-brain that produces two processes writing one volume. The monotonic watermark exists
for exactly this, and the restore path advances it before serving a single request.

> **2. Rotate the signing key.**

Capability tokens verify **offline** against that key (§6 of `08-identity.md`), which is what lets
console access survive an outage, and it means a token's validity does not depend on the database
that recorded it. Restore an older backup and a grant issued *and revoked* after it is absent from
the database while its token still verifies: authority in circulation, no record of it, and an
audit trail that contradicts reality. Rotation invalidates every token issued after the backup.
Sessions re-authenticate, which is a bounded and visible interruption during a disaster the
operator is already handling, and the short token lifetimes of §6 of `08-identity.md` bound how
long it lasts.

Skipping either step converts a recovery into an incident. Neither is a procedure someone can
forget; both run before the restored control plane serves a request.

Everything else follows from the model: agents reconnect, present the intent-set digest of §9 of
`10-api.md`, and the fleet re-converges from state that was never lost because it was never only in
the control plane.

---

## 7. Certificates and rotation

The classic self-inflicted outage in this class of system is an internal certificate expiring and
every node disconnecting within the same hour, at 03:00, on a holiday.

- Agent keys and control-plane certificates rotate automatically, well before expiry.
- Lifetimes are **staggered at issue**, so a fleet enrolled on one afternoon does not expire on one
  afternoon.
- Approaching expiry is an event with escalating severity (§5 of `15-observability.md`), not a log
  line nobody reads.
- Manual rotation exists and is a documented, rehearsed operation, because it is what a compromise
  requires (§9 of `17-security.md`).

---

## 8. The runbooks that must exist

Naming them is part of the design, because an operational property nobody has written down is one
that will be improvised during an incident:

| Situation | Core of the procedure |
|---|---|
| node unreachable | confirm; decide `keep_running` vs `stop`; drain to peers; do not fence unless evicting |
| node compromised | revoke key, **increment lease epoch**, reschedule from intents, rebuild the host |
| control plane lost | restore Postgres and keys, **advance the epoch watermark and rotate the signing key**, restart, let agents re-converge |
| Postgres lost, no backup | unrecoverable for configuration; workloads keep running; re-declare intents |
| migration stuck | inspect phase; migration is resumable and reversible by construction (K-4, §8 of `05-scheduling.md`) |
| agent version drift | expected and supported; upgrade at leisure within N-2 |
| certificate expiry approaching | rotate; the alarm should have fired days earlier |
| quota exhausted, tenant blocked | quota is enforced in the write path; raise it or free space, never disable enforcement |
| a node's disk filling | inode and byte quota per volume; identify, then drain (§10 of `17-security.md`) |
| an extension misbehaving | it is a workload, throttle, restart, or evict it like one |

Every one of these is a procedure a Pterodactyl operator has performed with manual SQL, and each is
listed here because the design's claim is that none of them require it.

---

## 9. Air-gapped and constrained environments

Recipes and extensions are OCI artifacts in ordinary registries (§2 of `09-recipes.md`), so an
air-gapped install mirrors a registry (a solved problem with existing tools) rather than needing a
bespoke offline bundle format. Digest pinning means a mirrored artifact is verifiably the same
artifact. Nothing in the control plane requires outbound internet access, and telemetry is off by
default (§6 of `15-observability.md`).

---

## 10. Open questions

1. **Bundled Postgres. Resolved in §2**: `korpis-server install` provisions or connects, a compose
   quick-start exists for evaluation and says so at startup, and no database is vendored. What
   remains open is whether distribution packages arrive before or after there is a second
   maintainer to keep them from going stale (§6 of `19-governance.md`). → here
2. **Multi-region control plane.** One Postgres and nodes on three continents means reconciliation
   latency crossing oceans. Regional read replicas plus the `consistency_token` of §6 of
   `10-api.md` cover reads; whether writes need regional partitioning is unresolved and is a large
   change if so. → `03-state.md`
3. **Hardware attestation.** Carried from §13 of `17-security.md`. Measured boot would make
   compromised-node detection real; the key management is disproportionate for a single-operator
   install and would have to be optional. → here
4. **Windows node operations.** Enrollment, preflight, upgrade-without-restart, and drain all
   assume a Linux process model. K-9 requires Windows on day one at the *interface* level; the
   operational surface has not been worked through. → `04-runtimes.md`
5. **Upgrade rehearsal.** A `korpis upgrade --plan` that reports which agents would fall outside
   the support window, which extensions declare incompatible API versions (§6 of
   `16-extensions.md`), and which migrations would run, before anything moves. This is P5 applied
   to the platform itself and is probably not optional. → here
