# Metrics, Metering, Audit, and Alerting

**Status:** design
**Date:** 2026-08-07
**Depends on:** [`03-state.md`](./03-state.md), [`14-streams.md`](./14-streams.md), [`06-storage.md`](./06-storage.md)
**Implements:** Principles P4, P5; Rules K-3, K-12, K-16

---

## 1. Four things, not one

Panels in this market build one subsystem called "stats" and use it for everything. That is why their
billing numbers are wrong: a series designed to draw a graph is sampled, lossy, and interpolated,
which is correct for a graph and unacceptable for an invoice.

| | Consumer | Guarantee | Retention | Store |
|---|---|---|---|---|
| **Metrics** | operators, dashboards | best-effort, lossy, gap-tolerant | short at full resolution | time-series, outside Postgres |
| **Metering** | billing systems | **exact, gap-marked, idempotent** | years | Postgres, partitioned |
| **Audit** | operators, tenants, compliance | append-only, attributable, tamper-evident | years | Postgres, `Effect` |
| **Alerting** | humans | delivered or visibly not | none — it is a routing function | — |

They have different guarantees, so they are different systems. Collapsing them is a decision that
cannot be undone later, because by then invoices depend on it.

---

## 2. Metering, because money depends on it

### Measured, never assumed

Every metered quantity comes from a kernel counter — cgroup v2 accounting, `statfs`, nftables
counters, driver-reported device usage. **No metered quantity is ever derived from a configured
limit.** Pterodactyl reports the limit you set as though it were consumption because it cannot
measure the difference; that is the failure P4 and K-3 exist to prevent, and it is worst here,
because the number becomes an invoice.

### What is metered

```
cpu_seconds            cumulative, per workload
memory_byte_seconds    the integral, not the peak — peak is reported separately
disk_bytes_stored      sampled at a declared interval
disk_io_bytes          read and write, separately
network_bytes          ingress and egress, separately, on-net and off-net separately
device_seconds         per device class (GPU and others), from the driver
address_hours          dedicated IPv4 is scarce and is a real cost (§4 of `07-networking.md`)
backup_bytes_stored    post-deduplication, with the dedup scope named (§5.3 of `06-storage.md`)
```

Memory is metered as byte-seconds because that is what capacity actually costs, and peak is reported
alongside it because that is what people argue about. Both are published; which one a billing system
charges on is not Korpis' decision (K-12).

### Gaps are recorded, never interpolated

A missing interval is written as a gap with its cause — node unreachable, agent restarting, counter
reset — and the billing system sees the hole. The alternative, interpolation, means a host silently
bills a customer for a period nobody measured, which is indefensible the first time it is challenged
and is standard practice everywhere else.

Same principle as `14-streams.md` §3, applied where it is most expensive to violate.

### Idempotent by construction

A sample's identity is `(workload, resource, interval_start)`. Re-delivery after a network failure
overwrites rather than adds. An agent that reconnects and re-sends an hour of buffered samples cannot
double-bill anyone, which matters because that reconnect is a normal event, not an exceptional one.

### Continuity across restart, reboot, and migration

Counters are cumulative **per workload**, not per process, per container, or per node. A workload that
restarts does not reset its meter; a node that reboots does not lose the interval; a migration does
not double-count and does not drop.

Migration is the hard one, and it is already solved: the seven-phase migration of §8 of
`05-scheduling.md` has exactly one instant at which ownership transfers, fenced by a lease epoch. The
source node's final sample and the destination's first are cut at that instant. Without a fenced
cutover this is unsolvable — which is a good illustration of why the fencing was designed first.

---

## 3. Metrics

Prometheus exposition and OTLP. No custom protocol, no proprietary agent, no bundled dashboard
product — §2 of `00-overview.md` is explicit that Korpis is not a monitoring stack.

Three scrape surfaces:

- the control plane's own metrics — reconciliation lag, plan latency, queue depth, API rates
- each agent's metrics — node capacity, driver health, convergence lag per workload
- **a per-tenant endpoint, scoped by grant** — a tenant scrapes their own workloads into their own
  Prometheus, seeing exactly what their grants permit and nothing about the node or its neighbours

That third one does not exist in any competitor and costs nothing here, because the authorization
model already answers "what may this subject see" for every other read.

**Cardinality is a hard constraint, not a guideline.** Workload identifiers are bounded and are
labels. Player names, file paths, remote addresses, and anything else a tenant controls are not, and
never become labels — a tenant able to inject unbounded label values can take down the operator's
monitoring, which makes it an availability vulnerability rather than a tidiness concern.

Storage: recent data at full resolution near where it is produced, rolled up on the control plane,
and remote-written to whatever the operator already runs for anything longer. §3 of `03-state.md`
keeps telemetry out of the transactional store; metering is the deliberate exception, because its
volume is low and its value is high.

---

## 4. Audit is not a separate system

The `Effect` log **is** the audit log. There is no second table, no separate writer, and therefore no
possibility of the two disagreeing.

This falls out of §4 of `03-state.md`: an `Effect` is append-only, written in the same transaction as
the state change it describes, and names the grant that authorized it. Nothing can change state
without producing one, because producing one is how state changes. Bolt-on audit logs are incomplete
in exactly the cases that matter, since the paths that skip them are the unusual ones.

Three properties worth naming explicitly:

**Denials are recorded.** Most systems log what happened; the interesting security signal is what was
attempted and refused. A denied action produces an `Effect` with its outcome, its subject, and the
action that was missing.

**Sensitive reads are recorded.** Reading a secret config field, downloading a backup, attaching to
another tenant's console under operator authority (§7 of `08-identity.md`) — these are events. An
audit log that only contains writes cannot answer "did anyone look at this".

**Tamper-evidence, not just append-only.** `REVOKE UPDATE, DELETE` stops the application; it does not
stop someone with database access. Each day's effects are hash-chained and the day's root is signed
and published. An operator who edits history can still do so, but no longer silently — which is the
achievable guarantee, and claiming more would violate P4.

Export is to open formats for whatever SIEM the operator runs. Tenants can read their own audit trail
scoped by grant, which is unusual in this market and is simply what happens when the audit log is the
same object the authorization model already filters.

---

## 5. Alerting stops at the boundary

Korpis emits **events** with severity, subject, and structured detail. It does not build alert
routing, deduplication, escalation policies, on-call schedules, or silences — Alertmanager,
PagerDuty, and their peers exist and are better at it.

The minimum that ships, because it is Bet 1 rather than a monitoring feature:

```
selector: project "community-servers", severity >= warning
deliver:  discord channel #ops-alerts
```

A selector, a severity, and a destination. The destination holds a grant and receives only what that
grant permits (§7 of `12-surface-discord.md`). Anything richer — time windows, dependency
suppression, escalation — is an extension or an external system, and the event stream is published so
that either can consume it.

**Health is application-level** (K-16). A process that is running is not a healthy Minecraft server;
a healthy Minecraft server answers a query on its port. Health checks are declared per workload and
defined per tier in §5 of `04-runtimes.md`, and it is health — not process liveness — that generates
events.

---

## 6. Korpis phones home only if asked

An open-source project collecting usage telemetry is a trust decision, and the honest form of it is:
**off by default, opt-in, and the exact payload is printable before you enable it.**

```
korpis telemetry show     prints precisely what would be sent
korpis telemetry enable
```

No instance identifier is transmitted unless the operator enables it, nothing about tenants,
workloads, or names is ever included, and the aggregate is published so the project's own data is
visible to the people who supplied it. P10 forbids charging for growth; quietly measuring it would be
a different way of taxing trust.

---

## 7. Open questions

1. **Metering resolution.** Per-minute intervals are ~525,000 rows per workload per year; per-hour is
   8,760 and loses burst detail that "dedicated vCPU" claims depend on (§5.2 of `05-scheduling.md`).
   High-resolution recent with automatic rollup is likely, and the rollup must remain gap-honest. → here
2. **Restatement.** A metering bug discovered after invoices are issued needs a correction path that
   is auditable rather than an `UPDATE`. Append-only correction records with an explicit supersedes
   link are the shape; the API for it is unspecified. → here
3. **Node-local buffering horizon.** How long an agent buffers metering while the control plane is
   unreachable determines how long an outage can last before revenue data is lost. §4.6 of
   `02-architecture.md` guarantees workloads keep running; it does not yet guarantee they keep being
   measured. → `02-architecture.md`
4. **Trace propagation.** Following a request across declared dependencies (§5 of `07-networking.md`)
   requires injecting context into tenant traffic, which is intrusive and possibly not Korpis' place.
   Carried from §9 of `14-streams.md`. → here
5. **Tenant-visible operator access.** §7 of `08-identity.md` promises tenants can see when an
   operator accessed their workload. Which accesses are exempt — an operator debugging a node-wide
   failure touching every tenant — is a policy question with real privacy weight.
   **Resolved in §8 of `17-security.md`** — none are exempt.
