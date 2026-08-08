# State: Storage, the Event Log, and Consistency

**Status:** design **Date:** 2026-08-07 **Depends on:** [`01-model.md`](./01-model.md),
[`02-architecture.md`](./02-architecture.md) **Resolves:** open question 2 of `01-model.md`

---

## 1. Event sourcing is not needed here, and that is the important finding

The obvious design for a system built on Intent → Plan → Effect is full event sourcing: the event
log is the source of truth, and all queryable state is a projection rebuilt from it. It buys time
travel, perfect audit, and replay.

It was the wrong answer, and working out why changed this document.

Ask what history is actually needed for:

| Need | Provided by | Requires event sourcing? |
|---|---|---|
| Who did what, under what authority | the `Effect` log | no, an append-only table is enough |
| Roll back to an earlier configuration | the `Intent` chain | **no: intents are already immutable and versioned** |
| Explain an incident after the fact | `Observation` history | no: that is a time series, not an event log |
| Bill for usage | metering records | no: also a time series |

The second row is the finding. **`Intent` is already an immutable, versioned, parent-linked
chain.** The thing anyone would want to time-travel (the declared configuration of a workload) is
*already* stored as its complete history, by construction. Rollback is "point at intent version N",
not a replay of anything.

So event sourcing would pay full price, projections that drift, rebuilds measured in hours, event
schema migration, and debugging through an indirection, to buy something the model already has.

**Decision: PostgreSQL is authoritative. Intents are immutable rows, append-only by construction.
Effects are an append-only table written in the same transaction as the state change they describe.
No projections, no rebuild, no replay.**

The transactional coupling is the part that matters: a state change and its effect record commit
together or neither commits. Audit cannot drift from reality, because there is no window in which
one exists without the other. This is what a separate audit-logging subsystem (the usual design)
cannot guarantee.

---

## 2. Three data classes, three storage strategies

The mistake to avoid is treating all state as one kind of thing. Korpis has three, with different
volumes, lifetimes, and consistency requirements.

| Class | Objects | Volume | Consistency | Where |
|---|---|---|---|---|
| **Configuration** | Organization, Project, Quota, Grant, Workload, Intent, Plan, Recipe, Volume, Endpoint, Placement, Lease, Extension | small, highly relational | strong, transactional | PostgreSQL |
| **Audit** | Effect | high, append-only, never updated | strong at write, eventually archived | PostgreSQL, time-partitioned → object storage |
| **Telemetry** | Observation history, metrics, metering samples | very high, low value per row | best-effort | **not the main database**, see §5 |

The third row is the one that kills systems that get it wrong. A thousand workloads reporting every
few seconds is millions of rows a day of data whose individual value approaches zero. Writing that
into the same database that holds grants and intents means the transactional store is dominated by
traffic that never needed a transaction.

Pterodactyl avoids this by not storing observations at all. CPU and memory graphs are polled live
over a websocket and discarded. That is why no Pterodactyl installation can answer "what was this
server doing at 3am on Tuesday". Korpis stores it, but not there.

---

## 3. Configuration state

Standard relational design in PostgreSQL. Three properties are non-obvious and load-bearing.

### 3.1 Intents are append-only by constraint, not by convention

```sql
CREATE TABLE intent (id             uuid PRIMARY KEY,
  workload_id    uuid NOT NULL REFERENCES workload(id),
  version        bigint NOT NULL,
  parent_id      uuid REFERENCES intent(id),
  declared_by    text NOT NULL,
  declared_at    timestamptz NOT NULL DEFAULT now(),
  body           jsonb NOT NULL,
  UNIQUE (workload_id, version));

REVOKE UPDATE, DELETE ON intent FROM korpis_app;
```

The privilege revocation is deliberate. Immutability enforced by the database cannot be
accidentally violated by application code, and application code is where every such invariant
eventually gets violated.

`body` is `jsonb` rather than fifty columns because the intent body evolves with every new runtime
driver, endpoint type, and health probe kind, and because it is validated against a schema before
insertion (Rule K-17) rather than by column types. Indexed expressions cover the fields that are
actually queried, placement constraints, recipe digest, lifecycle.

#### The body is protobuf, stored in its canonical JSON encoding

> Resolves open question 5 of §12 of `10-api.md` and 5 of §10 of `21-stack.md`.

The API carries the intent body as a typed protobuf message (§3 of `10-api.md`); the store holds
`jsonb`. Two representations of the same thing is how they drift, and an intent written two years
ago must still be readable, because rollback is re-declaring an earlier version (§4 of
`18-operations.md`).

There is one representation, not two: **`body` holds the protobuf message in proto3 canonical
JSON.** The schema is still the `.proto`; the column is that message in its defined textual
mapping. So it is queryable and indexable in PostgreSQL, it round-trips exactly, and there is no
second schema to keep in step.

Three details make it hold across versions:

- **Field names, not numbers.** Numbers would make the column unreadable and defeat the point.
  Names are just as stable, because §4 of `10-api.md` already forbids renaming a field in the wire
  schema, the rule that protects field numbers protects JSON keys for free.
- **Rows are never rewritten.** Intents are immutable, so a row is exactly the bytes that were
  validated at write. A field that a later version removed is still present in old rows and still
  reserved forever, so nothing reinterprets it.
- **`schema_version` is recorded on the intent.** Reading an old intent means parsing it with the
  schema that was in force when it was written, rather than hoping the current one still fits. This
  is what makes rollback to a two-year-old intent a defined operation instead of an optimistic one.

The compatibility rules of §4 of `10-api.md` therefore cover the database as well as the wire,
which is the property being bought: one schema, one set of rules, one place a mistake can be made.


**Defaults are written, not omitted.**

> External review, finding R10. Recorded in §16 of `23-walkthroughs.md`.

Proto3 canonical JSON omits fields that hold their default value. Combined with "rows are never
rewritten, so nothing reinterprets them", that produces exactly the outcome the rule was written to
prevent: a stored body that does not contain a field is not recording *the default at the time it
was written*, it is recording *whatever the default is when someone reads it*. Change a default in
2028 and every row from 2026 silently changes meaning while remaining byte-identical.

So intent bodies are serialized **with defaults emitted**. Rows are larger and their meaning is
frozen, which is the trade this whole section exists to make.

The rule that would otherwise be needed instead, "a field's default may never change", is added to
§4 of `10-api.md` as well, because it costs nothing and defends the same property from the other
side.

**Integers are strings, and the index has to know that.** Canonical JSON encodes `int64` and
`uint64` as strings. Expression indexes over those fields, and any range query on them, need an
explicit cast, and a cast that is not in the index definition is a sequential scan. The fields this
actually affects are few and known (byte counts, epochs, versions), so they are indexed with the
cast written into the index rather than discovered from a slow query log.
### 3.2 Conflicting declarations are detected, not merged

Two clients declaring intents for one workload at the same time is normal: a web user and a Discord
command, or an operator and an autoscaler. Last-write-wins would silently discard one of them.

Every intent creation is a compare-and-swap:

```sql
UPDATE workload
   SET intent_id = $new, intent_version = $expected + 1
 WHERE id = $workload AND intent_version = $expected;
```

Zero rows affected means someone else moved first. The loser receives a conflict carrying the
intent that won, and must recompute its plan against the new base. This is the same rule as §4.1 of
`02-architecture.md`: a plan can never be applied against a state it was not computed for.

### 3.3 Authorization is not in the database

PostgreSQL row-level security is deliberately **not** used, despite fitting the tenancy model well.

Grants are evaluated in the Gateway and nowhere else (P6, §5 of `02-architecture.md`). Putting a
second copy of the authorization logic in database policies would create two places where authority
is decided, which eventually disagree, and the disagreement is a security bug that is invisible
until it is exploited.

One authorization path. The database enforces referential integrity and immutability; it does not
enforce policy.

---

## 4. The effect log

```sql
CREATE TABLE effect (id           uuid NOT NULL,
  at           timestamptz NOT NULL,
  plan_id      uuid,
  step         int,
  workload_id  uuid,
  node_id      uuid,
  actor        text NOT NULL,
  grant_id     uuid,
  action       text NOT NULL,
  outcome      text NOT NULL,
  error        jsonb,
  before       jsonb,
  after        jsonb,
  PRIMARY KEY (at, id)) PARTITION BY RANGE (at);

REVOKE UPDATE, DELETE ON effect FROM korpis_app;
```

- **Partitioned by time, on the ingest clock.** Monthly partitions. Detaching an old partition is
  instant; deleting millions of rows is not.

> External review, finding R8. Recorded in §16 of `23-walkthroughs.md`.

`at` is the time the control plane **ingested** the effect, from one clock, and it is what the
primary key and the partition boundary use. An agent's own reading of when the thing happened is a
separate column, `occurred_at`, and it is never a key of anything.

The first version of this schema did not draw that line, and a partition key fed by node clocks has
two failure modes that are not theoretical. A node whose clock is minutes fast writes effects into
a partition that does not exist yet or into the wrong month. And causal order inverts in the log
that exists to preserve it: `stop decided 12:00:05` above `container stopped 11:59:58`, in a table
whose entire purpose is being the record nobody can argue with.

Both readings are kept, because both are true and they answer different questions. `at` orders the
log and is monotonic within the control plane. `occurred_at` is what the node believed, it is what
a latency investigation needs, and a large gap between the two is itself a finding worth an alert.

```sql
  at            timestamptz NOT NULL,   -- ingest clock, PK and partition key
  occurred_at   timestamptz,            -- the agent's reading, never a key
```
- **Archived, never deleted.** Detached partitions are exported to object storage in a documented
  open format and remain queryable through an explicit archive path. Retention policy governs what
  is *online*, not what exists.
- **Every effect names the grant.** Not just the actor, the authority. `01-model.md` §3.3 explains
  why: "who did this" and "under what authority, delegated from whom" are different questions, and
  only the second one is useful during an incident.
- **Effects are written by drift correction too.** Restarts, recreations, and repairs are recorded
  even though they produce no `Plan` (§4.7 of `02-architecture.md`). The plan log answers "what did
  someone decide"; the effect log answers "what actually happened". They are not the same question
  and the system must be able to answer both.

---

## 5. Telemetry

Observations arrive constantly and are 99% uninteresting. They are split by how they are used.

**Latest observation: PostgreSQL, one row per workload, updated in place.** This is the
current-state cache the UI reads. Overwriting is correct: it is not history, it is "now". History
lives in the next row of this table.

**Observation history: a separate time-series path.**

```
raw          full fidelity          hours
1-minute     downsampled            weeks
5-minute     downsampled            months
1-hour       downsampled            years
```

State transitions (`running → crashed`, `healthy → unhealthy`) are **not** downsampled. They are
promoted into the effect log, because a crash is an event and belongs in the record that survives.
Downsampling loses the CPU curve, not the fact that something died.

**Metering: a separate append-only series**, because it is the one telemetry stream with financial
consequences and must not be downsampled, dropped under pressure, or reconstructed from averages
(Rule K-12). Detail in `15-observability.md`.

The concrete choice of time-series engine is deferred to `15-observability.md`. What is decided
here is that it is **not the transactional store**, and that observations never enter a transaction
with configuration state.

---

## 6. The job queue, and killing the SQLite assumption

An earlier design proposal built the job queue on PostgreSQL's `LISTEN/NOTIFY` and `SELECT … FOR
UPDATE SKIP LOCKED`, and in the same breath said a SQLite path would stay open for small
installations. Those two statements contradict each other: both mechanisms are PostgreSQL-specific.
`01-model.md` recorded this as an open question. It is resolved here.

**Decision: PostgreSQL is the only supported store. SQLite is not supported. MySQL and MariaDB are
not supported.**

The alternative (abstracting the queue so several engines work) was rejected for reasons that are
worth stating, because "support more databases" always sounds generous:

- `SKIP LOCKED` and `LISTEN/NOTIFY` together give a correct, fast, dependency-free job queue. An
  abstraction that also runs on SQLite must degrade to polling, and the degraded path becomes the
  one that is under-tested and where the subtle bugs live.
- Multiple engines multiply the test matrix forever, and the multiplication compounds with every
  feature. Pelican added SQLite, MariaDB, and PostgreSQL drivers, which is three code paths for
  every query, permanently.
- `jsonb`, partial and expression indexes, partitioning, `generated always as identity`, and
  transactional DDL are all used above. A portable subset would mean giving up most of them.

The cost is real and is accepted: an operator must run PostgreSQL. `00-overview.md` requires that a
single operator with one machine can install and run Korpis, and that requirement is met by
shipping PostgreSQL as part of the installation rather than by weakening the store. One database
that always behaves the same way is worth more than three that mostly do.

**The queue itself:**

```sql
CREATE TABLE job (id            uuid PRIMARY KEY,
  key           text UNIQUE NOT NULL,   -- idempotency key
  kind          text NOT NULL,
  payload       jsonb NOT NULL,
  run_after     timestamptz NOT NULL DEFAULT now(),
  attempts      int NOT NULL DEFAULT 0,
  max_attempts  int NOT NULL DEFAULT 10,
  locked_by     text,
  locked_until  timestamptz,
  status        text NOT NULL);
```

Workers claim with `FOR UPDATE SKIP LOCKED`, wake on `NOTIFY`, and fall back to polling on a slow
timer so a missed notification delays a job rather than stranding it, level-triggered, for the same
reason as §4.1 of `02-architecture.md`.

`key` is `(workload_id, intent_version, step_index)` for plan steps, which makes enqueueing the
same work twice a no-op and satisfies the idempotency requirement of §4.7 of `02-architecture.md`
directly rather than by discipline.

No Redis. No NATS. No broker. They are added if and when a measurement demands it, not because the
architecture diagram looks more serious with them.

---

## 7. One plan at a time, and how the other one is stopped

> External review, finding R6. Recorded in §16 of `23-walkthroughs.md`.

Making the diff a first-class object (Bet 3) buys dry-run, approval, and truthful audit. It also
buys an obligation that a level-triggered reconciler never has, and this design had not paid it.

Kubernetes has no Plan, so a controller simply looks at the newest desired state on every pass and
the question does not arise. Korpis persists the plan, which means a plan can be **half applied**
when a newer intent arrives. `Plan.status` covers `superseded`, but supersession as written is
about the moment of computation: a plan is invalid if reality moved under it. It says nothing about
a plan that is `applying`, whose agent is on step two of five, when someone clicks stop.

The compare-and-swap that guards the mutation is on `intent_version`. It is not on "no plan is
currently applying", so the new intent commits, a second plan is computed, and it is computed
against a reality that is *half of the first plan*, which is a state the model has no way to name.

Two things close it, and both are needed:

**Plans are serialized per workload.** A workload has at most one plan in `applying` at a time,
enforced by the store rather than by convention. A plan proposed while another is applying is
computed and persisted as normal, so it is inspectable, and it enters `queued`.

**A queued plan preempts rather than waits, when it can.** Waiting for a five-step migration to
finish before honouring a stop button is not acceptable, so `plan.cancel` is part of the agent
protocol:

```
control plane ──plan.cancel(plan_id, at_step_boundary)──▶ agent
agent finishes the step in flight, does not begin the next
        ──effect: plan cancelled at step N──▶ control plane
plan.status = cancelled, workload state = partial, resources named
```

Cancellation lands **on a step boundary**, never inside one, which is what makes it cheap: §3.1 of
`05-scheduling.md` already requires every step to be ordered, idempotent, and resumable from the
last completed one, and a cancelled plan is a partial plan the model can already express. The next
plan is computed against that partial state exactly as it would be against a failed one.

`plan.cancel` is a request, not a command. A step in flight completes, because interrupting a
half-written volume is how a system acquires states nobody declared. The cost is a bounded wait
equal to one step, which is stated, rather than an unbounded wait equal to one plan, which was the
alternative nobody had chosen on purpose.

---

## 8. Backup and restore of the control plane

The store is the only stateful component in the control plane, so backing up Korpis is backing up
one PostgreSQL database. Continuous WAL archiving with point-in-time recovery.

**Restoring a store while agents are running is dangerous, and the danger is leases.**

A restored store contains lease epochs from the moment of the backup. Agents in the field hold
*newer* epochs. Under the fencing rule of §4.5 of `02-architecture.md`, an agent presenting an
older epoch is fenced, the restored control plane would consider every running agent stale, while
those agents consider themselves authoritative. That is precisely the split brain leases exist to
prevent, reintroduced by the recovery procedure.

**Restore therefore includes a mandatory epoch fence.** Getting the arithmetic of that fence right
took a correction, and the wrong version is worth leaving visible because it is an easy mistake to
repeat.

> External review, finding R11. Recorded in §16 of `23-walkthroughs.md`.

The first version of this section said the store records a `max_issued_epoch` watermark "so the
restore has a concrete number to advance beyond". **That watermark is in the database being
restored**, so it holds the value as of the backup, and every epoch issued between the backup and
the failure is by definition larger than it. Advancing past a stale watermark does not clear the
epochs that actually exist in the field. The sentence described an intention, not a mechanism.

Two mechanisms do work, and the restore uses both:

1. **A bounded issue rate makes the watermark usable again.** Epochs are issued at a rate the
   control plane caps and records, so `W_backup + rate × max_backup_age` is a real upper bound on
   what could have been issued, and the restore advances past that. The cap is the reason the bound
   exists; without it the arithmetic has no ceiling.
2. **Reconnecting agents raise the floor.** The restored control plane refuses to issue for a
   configured settling window, during which every agent that reconnects presents the epoch it
   holds, and the control plane jumps past the maximum it saw. This closes the case where 1's bound
   was computed against a wrong backup age.

The adversarial corner of 2 is that an agent reports the epoch, so a compromised or broken agent
can report an enormous one and exhaust the epoch space. **Epochs are signed by the control plane
that issued them**, and an agent presenting an epoch it cannot prove was issued is fenced rather
than believed. An epoch is authority, and this design does not take an unverified claim of
authority from anywhere else either.

Mechanism 1 alone is safe but conservative. Mechanism 2 alone is exact but depends on agents
appearing. Together the fence is correct when every agent reconnects and still correct when none
does.

Agents reconnect, discover their epochs are stale, and **freeze mutations while leaving processes
running**, then request reissue according to their `on_expiry` policy.

> External review, finding R12.

Fail-static rather than fail-stop, and the default matters because the alternative is a recovery
procedure that stops tenants' workloads. Fencing exists to prevent two authorities mutating one
workload. A fenced agent already refuses mutations, so killing the processes it is supervising
prevents no inconsistency that refusing mutations has not already prevented. It only converts a
control-plane incident into a tenant-visible outage, which is the opposite of what the restore is
for. `on_expiry: keep_running` is the default for exactly this reason (§4.5 of
`02-architecture.md`), and `stop` remains available for the workloads whose correctness genuinely
depends on single ownership of a shared resource.

The system converges from a known state instead of a contested one. Slower than pretending nothing
happened; correct.

**And restore rotates the signing key, for the same reason.**

> Finding 9 of `23-walkthroughs.md`.

Leases are not the only authority that outlives the database. §6 of `08-identity.md` makes a grant
and its capability token the same object in two representations, and the agent verifies tokens
**offline** against a signing key, which is what lets console access survive a control-plane
outage, and which has a consequence nobody wrote down until a walkthrough went looking for it.

Restore to Tuesday's backup. A grant issued Wednesday and revoked Thursday is absent from the
restored database, but its token is signed by a key that was restored along with it, so it still
verifies. **A revoked grant is silently resurrected, and no row anywhere records that it exists.**
The audit trail is not merely incomplete; it contradicts reality. The symmetric case is milder and
also wrong: a legitimate grant issued Wednesday keeps working while the interface shows nothing, so
nobody can revoke it.

So restore rotates the signing key. Every token issued after the backup stops verifying, and every
session re-authenticates. **A restored control plane must invalidate authority it can no longer
account for**, the same principle as the epoch fence, applied to the other kind of authority the
system hands out.

The cost is a bounded, visible interruption during a disaster the operator is already handling. The
alternative is authority in circulation with no record of it, which is the worse outcome by a wide
margin, and the token lifetimes of §6 of `08-identity.md` bound how long the disruption lasts.

**Key rotation and epoch advancement are both unskippable restore steps.** Neither is a documented
procedure someone can forget; both are performed before the restored control plane serves a
request.

---

## 9. Scale expectations

Stated so they can be measured against, and so the point at which this design must be revisited is
known in advance rather than discovered in production.

| Dimension | Comfortable | Revisit at |
|---|---|---|
| Nodes | 1 – 500 | 1,000 |
| Workloads | 1 – 50,000 | 100,000 |
| Intents/second | < 10 | 100 |
| Effects/day | millions | tens of millions |
| Observations/second | thousands | handled outside the transactional store by design |

The load that scales fastest is telemetry, and §5 keeps it out of PostgreSQL entirely. The
transactional store scales with *decisions*, not with *workloads*, and decisions are made by humans
and schedulers at human and scheduler rates. A node running two hundred workloads that nobody
touches generates approximately zero transactional load.

---

## 10. Open questions

1. **Which time-series engine?** Embedded (a local columnar store, keeping the single-binary
   promise) versus external (Prometheus/VictoriaMetrics/TimescaleDB, better tooling, another
   dependency). This interacts with §5 and with the metering guarantees, since metering must not be
   lossy. → `15-observability.md`
2. **Retention defaults for intents.** Intents are small and are the rollback history, so keeping
   them forever is affordable, until a workload edited by an autoscaler accumulates millions.
   Pruning by age loses old rollback targets; pruning by count loses the oldest. Neither is
   obviously right. → here, once autoscaling behaviour is designed
3. **Is `Plan` worth persisting after it is applied?** Applied plans are referenced by effects and
   are the record of what was decided, but they are large and most are auto-approved non-events.
   Possibly: persist plans that required approval or contained irreversible steps, and summarize
   the rest into their effects. → here
4. **Archive query path.** Detached effect partitions in object storage must stay queryable for
   incident review and compliance. Is that a Korpis feature, or an exported open format and
   somebody else's query engine? The second is cheaper and matches "Korpis is not a monitoring
   stack" (§2 of `00-overview.md`). → `15-observability.md`
5. **Control plane read scaling.** Read replicas serve the dashboards well, but replication lag
   makes a client see its own write disappear, the worst possible UX for an operator who just
   pressed a button. Read-your-writes routing is the standard fix and it needs designing rather
   than assuming. **Resolved in §6 of `10-api.md`**, a `consistency_token` returned by every write
   and replayed on reads.
