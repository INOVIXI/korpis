# Walkthroughs

**Status:** design
**Date:** 2026-08-07
**Depends on:** every preceding document

---

## 0. What this document is

Twenty-three documents, each internally consistent. That is not the same as working together.

Designs do not fail inside a document. They fail **between** documents — where one says "this is
handled in the storage layer" and the storage layer never heard about it. Per-document review cannot
find those, because each document reads correctly on its own.

So: seven concrete scenarios, traced end to end through every layer they touch. The output is not a
tutorial. It is a **defect list**, and it is written to be adversarial — the goal of each trace is to
find the step where nobody is holding the ball.

**Seven traces produced nine findings, and all nine are now closed** and patched into the documents
they belong to — the table in §8 says where each landed. Each cost a paragraph to fix here. Finding
them in month nine would have cost a rewrite.

---

## 1. Creating a workload, and sharing it

**Trace**

| # | Step | Governed by |
|---|---|---|
| 1 | Subject declares a workload; grant checked at declaration | `08-identity.md` §3 |
| 2 | Recipe reference `…:1.20` resolved to a digest, **recorded in the intent** | `09-recipes.md` §8 |
| 3 | Config validated against the recipe's schema, per-field permissions applied | `09-recipes.md` §5 |
| 4 | Quota checked | `05-scheduling.md` §5 |
| 5 | Placement: filter → score → bind, `Explanation` recorded | `05-scheduling.md` §2 |
| 6 | Plan persisted; low-risk, so applied directly and shown after | `11-surface-web.md` §4 |
| 7 | `Intent` v1 written, `Effect` in the same transaction | `03-state.md` §3.1, §4 |
| 8 | Agent receives the assigned intent set | `02-architecture.md` §4.1 |
| 9 | `Prepare` — content fetched by digest, signature verified, **pure** | `04-runtimes.md` §4.2 |
| 10 | Volume created with byte and inode quota in the write path | `06-storage.md` §3 |
| 11 | Endpoint allocated, exposure mode applied | `07-networking.md` §3 |
| 12 | `Create`, then `Start` | `04-runtimes.md` §4.2 |
| 13 | Health probed at the application level | `04-runtimes.md` §5 |
| 14 | `Observation` written by the agent, `intent_seen` advances | `02-architecture.md` §4.3 |
| 15 | Web shows converging, then converged, with the lag | `11-surface-web.md` §3 |
| 16 | Owner issues a share grant; token travels in the fragment | `08-identity.md` §6.1 |

**FINDING 1 — Quota is checked, but not held. `CONFIRMED`**

Step 4 checks quota; step 7 writes the intent. `10-api.md` §6 protects the *workload* with
compare-and-swap on `expected_version`, and that is the wrong object: quota is a property of the
organization, not of the workload being created. Two concurrent creates in the same organization each
check 24 Gi of 32 Gi used, each pass, and both commit — 8 Gi over.

This is not exotic. It is a customer clicking twice, a Discord command racing a web action, or a
pipeline creating a batch.

**Resolution.** Quota consumption is written in the same transaction as the intent, against a row
whose version participates in the CAS — the quota entry, not the workload, is the contended object.
A quota check that does not take a lock on what it is checking is a race, and the honest name for the
current design is "advisory quota", which P4 forbids displaying. → patch `05-scheduling.md` §5.1 and
`03-state.md` §3.2

**FINDING 2 — A Plan's steps have no stated atomicity. `CONFIRMED`**

Steps 10–12 are three durable side effects. If the volume is created and endpoint allocation fails,
what exists? `05-scheduling.md` §3 gives `Operation` ordered steps across *many* workloads, and P9
promises every operation is resumable and reversible — but nothing states the semantics for the steps
*within* a single workload's plan.

**Resolution.** A Plan's steps are ordered, individually idempotent, and resumable from the last
completed step; a failed plan leaves a workload in a state the model can represent — which is what
P9's "no operation may leave a workload in a state the model cannot represent" already demands and
never operationalizes. Partial resources are retained and named in the observation, never orphaned
silently. This is the same treatment `02-architecture.md` §4.4 gives adopted runtime objects. →
patch `05-scheduling.md` §3

*Confirmed correct in this trace:* recipe digest resolved **once, centrally, at step 2** rather than
per node — two agents cannot resolve the same tag differently. This was the seam I expected to find
and `09-recipes.md` §8 already closes it.

---

## 2. Migrating a busy server

**Trace**

| # | Step | Governed by |
|---|---|---|
| 1 | Operator drains a node; `Operation` with `max_unavailable` | `05-scheduling.md` §3 |
| 2 | Per workload: destination chosen, `Explanation` recorded | `05-scheduling.md` §2 |
| 3 | Volume snapshot; incremental transfer while running | `06-storage.md` §8 |
| 4 | Final incremental with the workload paused | `05-scheduling.md` §8 |
| 5 | **Cutover:** source lease epoch fenced, destination started | `02-architecture.md` §4.5 |
| 6 | Edge forwarding target updated in the same step | `07-networking.md` §3.1, §10 |
| 7 | Meter cut at that instant — no reset, no double count | `15-observability.md` §2 |
| 8 | Players reconnect to the same address | — |

**FINDING 3 — The console does not migrate. `CONFIRMED`**

`14-streams.md` §4 puts segment files in the tenant's storage on the node, and the agent owns all
three tiers. Nothing in the migration sequence moves them. So a workload migrates and its scrollback
stays behind on a machine that is about to be decommissioned — and §1 of that same document argues at
length that the console is *evidence*, not a widget.

Worse: offsets are per-stream and monotonic. If the stream restarts on the destination, either
offsets restart (breaking every reader's resume) or the destination must know where the source
stopped.

**Resolution.** The stream is part of the workload's durable state and migrates with the volume, in
the same incremental transfer. The destination continues the offset sequence, which it can only do
if the final offset crosses in the cutover message — so the offset watermark joins the lease epoch as
something the cutover carries. Any range that could not be transferred is a **gap record**, so a
migrated console is honest about what it lost. → patch `14-streams.md` §4 and `05-scheduling.md` §8

**FINDING 4 — A client attached to the console during cutover. `PLAUSIBLE`**

The reader is connected to the source node with a scoped token. At cutover the source is fenced. The
reader sees the connection drop.

This is probably correct behaviour and does not need machinery — the reader reconnects, is directed
to the destination, and resumes at its last offset, which Finding 3's resolution makes possible. It
is recorded because it is only correct *given* Finding 3; without it, the reader resumes at an offset
that means something different. → verify after patching

---

## 3. A node dies

**Trace**

| # | Step | Governed by |
|---|---|---|
| 1 | Agent stops reporting; node marked `unreachable` | `02-architecture.md` §9 |
| 2 | Observations become `unknown` — never a stale value shown as current | `11-surface-web.md` §2 |
| 3 | Lease expires; `on_expiry` decides `keep_running` or `stop` | `02-architecture.md` §4.5 |
| 4 | Metering records a gap with its cause | `15-observability.md` §2 |
| 5 | Workloads on replicated storage are rescheduled | `18-operations.md` §5 |
| 6 | Workloads whose only copy of data was local are **not** — and this is said plainly | `18-operations.md` §5 |
| 7 | On return: adoption by label, epoch checked; if fenced, it cannot resume | `02-architecture.md` §4.4 |

**FINDING 5 — What a `stable` address does while its workload is gone. `CONFIRMED`**

The edge owns the address and forwards to a placement that no longer exists. Nothing says what it
does. The three possibilities are meaningfully different to the person connecting: a black hole (they
wait thirty seconds and blame their own connection), a refusal (fast, honest, generic), or a
protocol-aware status (accurate, only for protocols the router understands).

This is exactly the affordance `22-first-party.md` §6 already built for hibernation — a router that
answers a Minecraft ping with "starting, ~30s". An unreachable workload is the same shape of problem
with a different message.

**Resolution.** Default is immediate refusal, never a black hole — a timeout is P4's "unknown"
rendered as a stall. Where a protocol router is present, it answers with the real state:
`hibernated`, `starting`, `unreachable`, or `data unavailable`. → patch `07-networking.md` §3.1

*Confirmed correct:* step 6 is the honest one, and it is already honest. A design that implied
automatic recovery of a workload whose only data copy died would be the most damaging possible lie,
and `18-operations.md` §5 refuses to tell it.

---

## 4. A reseller's customer's customer

**Trace**

| # | Step | Governed by |
|---|---|---|
| 1 | Operator delegates to a reseller: a grant, scoped to an organization | `08-identity.md` §3 |
| 2 | Reseller delegates to a customer — attenuated, never wider | `08-identity.md` §3.2 |
| 3 | Customer delegates to their own moderator | same mechanism, no new code |
| 4 | Quota inherits down the tree | `05-scheduling.md` §5.1 |
| 5 | Metering rolls up; the reseller sees their sub-tree's usage | `15-observability.md` §2 |
| 6 | `Explanation`s filtered per viewer — the customer learns nothing about the node | `08-identity.md` §8 |
| 7 | No reseller feature was written anywhere | Bet 4 |

**FINDING 6 — Quota inheritance does not say allocation or usage. `CONFIRMED`, and it decides Bet 4**

A reseller holds 32 Gi. They sell 16 Gi to customer A and 16 Gi to customer B, then want to sell
16 Gi to customer C — because in practice, customers use a fraction of what they buy, and
overselling is not an abuse of this market, it *is* this market.

If child quota consumes parent quota **by allocation**, the reseller is blocked at 32 Gi and Korpis
has forbidden the standard commercial practice of every host it is courting. If it consumes **by
usage**, then quota is not a guarantee and P4 is violated the first time three customers get busy
together.

Neither is acceptable, and the design currently states neither.

**Resolution.** Both, explicitly, per the same distinction `05-scheduling.md` §5.2 already draws for
resources: a quota entry has a **reservation** and a **limit**. Reservations sum against the parent
and cannot be oversubscribed; limits may exceed it, by a declared ratio, and the sum of limits is
visible to whoever set it. A reseller sells 16 Gi limits with 4 Gi reservations and knows exactly what
they have promised versus what they have guaranteed — which is what they are actually doing today
with a spreadsheet.

The overcommit ratio is per-dimension and memory defaults to off, exactly as §5.3 already requires.
Bet 4 survives, and it survives because the mechanism already existed one layer down; it had simply
never been applied to quota. → patch `05-scheduling.md` §5.1

**FINDING 7 — Who approves an extension's grants inside a delegated organization. `OPEN`**

`16-extensions.md` §4 says an extension declares what it needs and "the operator" approves. In a
three-level tree there are three parties who could reasonably be called the operator, and an
extension runs as a workload on the *machine owner's* hardware with declared egress.

A reseller's customer installing an extension is asking to run code on someone else's machine. The
grant model already prevents authority escalation — the extension cannot exceed its installer. But
egress, resource consumption, and reputation are the machine owner's exposure, not the installer's.

**Decision needed.** The likely shape is that installation requires the installer's grants *and* an
allow-list controlled by whoever owns the node, defaulting to first-party extensions only. That is a
policy choice with real product consequences, not something to settle inside a walkthrough. →
`16-extensions.md`

---

## 5. A hostile tenant

**Trace** — each attempt, and what stops it.

| Attempt | Stopped by |
|---|---|
| `../../etc/shadow` through the file API | `openat2(RESOLVE_BENEATH)` + Landlock; the string is never a decision | `17-security.md` §4 |
| symlink race in an upload | same — the kernel resolves or it does not | |
| fork bomb | cgroup `pids.max`, no reservation, no way to disable | `05-scheduling.md` §5.2 |
| a million empty files | inode quota | `06-storage.md` §3 |
| 50 MB/s of log output | stream rate limit, drop with gap marker, never blocks the workload, counts against their own quota | `14-streams.md` §4, §5 |
| exhaust the node's conntrack table | per-workload conntrack limit | `07-networking.md` §6 |
| reach `169.254.169.254` | egress default-deny | `07-networking.md` §6 |
| reach a neighbour | network namespace + default-deny | |
| read the daemon's credentials via a template | no ambient scope; a flat pre-resolved variable map | `09-recipes.md` §6 |
| escape the kernel via an LPE | **not stopped** in `process`/`container`; this is why the tier is a choice | `17-security.md` §5 |
| exfiltrate their own operator's secret from a log | redaction at write time — best-effort, labelled as such | `14-streams.md` §7 |

**FINDING 8 — Egress policy is enforced by the party being contained. `OPEN`, already known**

`07-networking.md` §11.6 raises it and hands it to `17-security.md`, which does not answer it. Egress
rules are enforced at the source node by that node's agent. A compromised agent can ignore them, and
a compromised agent is explicitly in the adversary table (`17-security.md` §2).

This trace does not resolve it — it confirms the handoff is dangling, which is the same class of
defect as the edge-availability one already fixed. The options are real and costly: enforce at the
edge (latency, a chokepoint), enforce at the destination (only works inside the fleet), or accept it
and state that a compromised node's egress is uncontained. **Decision needed.** → `17-security.md`

*Otherwise this trace is the strongest in the document.* Every attempt except the kernel LPE meets a
named, specific mechanism, and the LPE is not defended against by anyone and is labelled honestly.

---

## 6. Restoring the control plane from backup

**Trace**

| # | Step | Governed by |
|---|---|---|
| 1 | Postgres, signing keys, and the epoch watermark restored | `18-operations.md` §6 |
| 2 | **Epoch watermark advanced past every epoch that could have been issued** | `03-state.md` §7 |
| 3 | Control plane starts; agents reconnect | `02-architecture.md` §4 |
| 4 | Each presents its intent-set digest; most match; deltas for the rest | `10-api.md` §9 |
| 5 | Workloads never stopped — they were never dependent on the control plane | `02-architecture.md` §4.6 |

**FINDING 9 — Capability tokens outlive the database they were issued from. `CONFIRMED`**

`08-identity.md` §6 makes the grant and the capability token the same object in two representations,
and the agent verifies tokens **offline** against a signing key. That is what makes the system work
during an outage, and it has a consequence nobody wrote down.

Restore to Tuesday's backup. A grant issued Wednesday and revoked Thursday is gone from the database
— but its token is signed by a key that was also restored, so it still verifies. **A revoked grant is
resurrected, silently, and there is no row anywhere saying it exists.** The audit trail is not merely
incomplete; it actively disagrees with reality.

The symmetric case is milder and also wrong: a legitimate grant issued Wednesday keeps working while
the panel shows nothing, so it cannot be revoked through the interface.

**Resolution.** Restore rotates the signing key, exactly as it advances the epoch watermark, and for
the identical reason: a restored control plane must invalidate authority it can no longer account
for. Every token issued after the backup stops verifying, and every session re-authenticates. The
cost is a visible, bounded interruption during a disaster the operator already knows about; the
alternative is silent authority with no record.

Key rotation therefore joins epoch advancement as an **unskippable restore step**, and the token
lifetimes of §6 of `08-identity.md` bound how long the disruption lasts. → patch `03-state.md` §7,
`08-identity.md` §6, `18-operations.md` §6

---

## 7. Upgrading a fleet

**Trace**

| # | Step | Governed by |
|---|---|---|
| 1 | Control plane first; migrations forward-only and online | `18-operations.md` §4 |
| 2 | Agents upgraded one at a time, over days if wanted; N-2 window | `10-api.md` §4 |
| 3 | Each agent restarts and **re-adopts running workloads** — no tenant restart | `02-architecture.md` §4.4 |
| 4 | Extensions declaring an incompatible API version are refused at load, with a message | `16-extensions.md` §6 |
| 5 | Mixed-version fleet is a supported steady state, not a window | `18-operations.md` §4 |

**Nothing new found**, and one behaviour worth naming because it is easy to get wrong: an extension
refused at load in step 4 that happens to be a *provider* — a DNS provider, say — leaves its
dependents in `unsatisfiable` with the provider named, which is the same treatment
`16-extensions.md` §5 gives a provider that is merely down. A refused-at-load provider and a crashed
provider produce the same observable state, and that is correct: from a workload's perspective they
are the same event.

`18-operations.md` §10.5 already proposes `korpis upgrade --plan` to surface step 4 *before* anything
moves. This trace is the argument that it is not optional.

---

## 8. Findings

| # | Finding | Status | Patched into |
|---|---|---|---|
| 1 | Quota is checked without being held — concurrent creates overshoot | **patched** | `05-scheduling.md` §5.1 |
| 2 | A single workload's Plan has no stated step atomicity | **patched** | `05-scheduling.md` §3.1 |
| 3 | Streams do not migrate with their workload; offsets would break | **patched** | `05-scheduling.md` §8, `14-streams.md` §4 |
| 4 | Console reader across cutover | correct, given 3 | — |
| 5 | A `stable` address with no live workload has undefined behaviour | **patched** | `07-networking.md` §3.1 |
| 6 | Quota inheritance says neither allocation nor usage — decides Bet 4 | **patched** | `05-scheduling.md` §5.1 |
| 7 | Who approves an extension inside a delegated organization | **patched** | `16-extensions.md` §4.1 |
| 8 | Egress enforced by the party being contained | **patched** | `17-security.md` §9.1 |
| 9 | Capability tokens outlive a restored database — revoked grants resurrect | **patched** | `03-state.md` §7, `08-identity.md` §6, `18-operations.md` §6 |

Findings 1, 6, and 9 are the ones that would have cost real money to discover late: an overshot
quota is a support ticket, an unstated overcommit model is a reseller telling their customers
something Korpis does not do, and a resurrected grant is a security incident with no audit trail.

All three were found by the same method — following one concrete story across a boundary that two
documents each believed the other was holding.
