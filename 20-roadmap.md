# Build Order

**Status:** design
**Date:** 2026-08-07
**Depends on:** every preceding document

---

## 1. This is order, not scope

Everything described in the twenty preceding documents ships. Nothing here removes anything; this
document decides only what is built before what, and why that particular sequence rather than a more
comfortable one.

The comfortable sequence is to build the visible parts first — a panel, a server list, a console —
and add the model underneath later. That sequence produced Pterodactyl's abandoned v2, and it is
worth being precise about why: **some decisions cannot be retrofitted, and building the visible parts
first is a commitment to all of them by default.**

---

## 2. Four ordering rules

**A. Anything that shapes stored history goes first.** `Intent`, `Plan`, and `Effect` determine what
the system can ever say about its own past. A system that mutated state directly for six months
cannot later produce an audit trail for those six months, and reconciliation cannot be added to an
imperative core without rewriting it — which is the abandoned-rewrite failure, exactly.

**B. Anything a third party depends on across an upgrade boundary goes early.** Once agents exist on
machines Korpis does not control, the protocol is frozen (§2 of `10-api.md`). Getting it wrong before
anyone has deployed it costs a week. Getting it wrong afterwards costs a major version and everyone's
trust.

**C. A security boundary is built before the thing it protects, never after.** Privilege separation in
the agent, kernel confinement, and grant evaluation are not hardening passes. Six of twelve Wings
advisories are one class that exists because the boundary was drawn after the feature worked.

**D. The second implementation of an interface is the test of the interface, and it must be the most
*different* one available.** Not the easiest second one. After the `oci` driver, the driver interface
is proven by `firecracker` — a separate kernel, no shared filesystem, no shared PID namespace — not
by `native`, which is similar enough to prove nothing. The same rule applies to the second storage
class, the second exposure mode, the second surface, and the second identity source.

Rule D sits alongside K-18 rather than against it: ship **one excellent** driver, and let the second
one's purpose be interface validation rather than coverage. Five adequate drivers is what K-18
forbids; two excellent and maximally different ones is how the interface earns the right to be called
stable.

---

## 3. Dependency graph

```
  10-api ──────────────────────────────────────────────┐
  (schema, protocol, versioning)                       │
     │                                                 │
     ├─→ 01-model ─→ 03-state ─→ 02-architecture ──┐    │
     │   (nouns)     (Postgres,   (control/data,   │    │
     │               I/P/E)        leases)         │    │
     │                                             │    │
     └─→ 08-identity ──────────────────────────────┤    │
         (grants — evaluated by everything)        │    │
                                                   ↓    ↓
                              04-runtimes ─→ 17-security
                              (drivers)      (boundaries)
                                   │
        ┌──────────────────────────┼──────────────────────────┐
        ↓                          ↓                          ↓
  06-storage                 05-scheduling               07-networking
  (volumes, quota,           (placement, drain,          (endpoints,
   backup)                    migration, Operation)       exposure, egress)
        │                          │                          │
        └──────────────┬───────────┴──────────────────────────┘
                       ↓
                 14-streams ─→ 15-observability
                 (console)     (metering, audit)
                       │
        ┌──────────────┼──────────────┬───────────────┐
        ↓              ↓              ↓               ↓
  13-surface-cli  11-surface-web  12-surface-discord  16-extensions
  (reference)                                          │
                                                       ↓
                                                  09-recipes
                                                  (step providers,
                                                   config schema)

  18-operations and 19-governance apply throughout, not after.
```

`10-api` is at the root because the schema is the only artifact every other component shares.
`08-identity` reaches everything because authorization is evaluated on every call in every document
above. `09-recipes` sits late not because it is unimportant — it is how anything actually gets
installed — but because its escape hatch is extension-provided step providers, and building it before
the extension contract exists would bake in a worse escape hatch.

---

## 4. Phases

### Phase 0 — The parts that cannot be retrofitted

Schema and protocol. PostgreSQL with `Intent`/`Plan`/`Effect`, immutability enforced by the database.
Grants, attenuation, and capability tokens. The agent, dialling out, level-triggered, with full
privilege separation and lease fencing from the first commit. One runtime driver (`oci`), excellent.
The CLI.

**No web client in Phase 0.** This will look wrong and it is the most important ordering decision in
the document. §1 of `13-surface-cli.md` makes the CLI the reference client precisely so that P2 is
mechanically enforced; a web UI built while the API is still moving is how a project acquires a
privileged surface and, eventually, two disagreeing APIs. The CLI is also an order of magnitude
cheaper to build, so the model gets exercised sooner.

**Done when:** `korpis apply -f` places a workload on a node, converges it, survives an agent restart
without restarting the workload, and every action in that sequence — including the denied ones — is a
row in the effect log naming its grant.

That is the smallest artifact that proves the entire thesis.

### Phase 1 — Usable by one operator on one machine

Web client. Durable streams and console. Volumes with kernel-enforced byte and inode quota,
snapshots, and content-defined-chunking backup. Scheduling: filter, score, bind, with `Explanation`;
cordon and drain. Networking: `node_port` and `stable` exposure, default-deny egress.

**Done when:** the single-machine constraint of §1 of `18-operations.md` is real — install, run
workloads, back them up, restore them, and read an honest console, with every displayed limit
kernel-enforced (K-3).

### Phase 2 — Multi-tenant, and the two bets that live there

Organizations and recursive delegation. `GrantTemplate`. `IdentityBinding` and role-to-template
mapping. Quota sets. Metering, with the exactness guarantees of §2 of `15-observability.md`.
Tenant-visible audit. External identity through OIDC.

**The chat client is not here.** It is a first-party extension and ships in Phase 4 (§10 of
`12-surface-discord.md`). What Phase 2 owes it is the machinery — external identity mapped to
subjects, roles mapped to grant templates — and that machinery is owed to OIDC, SAML, resellers, and
CI regardless, so no part of it is chat-specific work.

**Done when:** a reseller delegates to a sub-tenant who delegates to a customer, quota inherits
correctly, and an external role grants a person authority over one project without an account being
created anywhere.

### Phase 3 — Isolation as a choice, and workloads that move

The `firecracker` driver — Rule D's validation of the driver interface. Migration, all seven phases,
resumable and reversible (K-4). Replicated storage classes. Edge forwarding, so a `stable` endpoint
survives the move. Devices, including GPU.

**Done when:** a running workload with a 40 GB volume moves between nodes, keeps its public address,
and its meter neither resets nor double-counts across the cutover.

### Phase 4 — The ecosystem

The extension contract, with the core extensions of §2 of `16-extensions.md` built through it and not
beside it. Recipes as OCI artifacts, signing, the install DSL, the config schema, egg import with its
three grades.

The chat client lands here, as an extension, alongside the chat-adapter provider interface.

**Done when:** game query, RCON, a DNS provider, a Steam install step, and a complete chat client are
all extensions, and removing any of them removes only itself. The chat client is the sharpest of
these: an extension that is a *full* client, not a peripheral, is the strongest available proof that
the contract is sufficient — which is exactly why it is worth building through the contract rather
than in-core, where it would be easier and would prove nothing.

This is also the phase in which Korpis becomes a credible game panel, and that is deliberate
sequencing rather than an afterthought. `22-first-party.md` explains the position: general-purpose
underneath, unmistakably excellent at game hosting on day one, with every game-facing capability
delivered through the extension contract this phase exists to prove. The generality is structural and
costs nothing; being nobody's first choice would cost everything, and a first-party set is the
cheapest possible answer to it.

Two items in that set depend on Phase 3 rather than this one — the Minecraft handshake router and
hibernation both need the edge — which is why they arrive here and not earlier.

### Phase 5 — The segments nobody else can reach

Windows drivers. Confidential-computing tiers if §13 of `17-security.md` resolves in their favour.
Protocol-aware routing. Everything the earlier phases' interfaces were shaped to permit.

---

## 5. Where each bet dies

The four bets in §4 of `00-overview.md` are falsifiable claims, so each has a phase at which it is
tested and a specific observation that would kill it.

| Bet | Falsified if | Tested in |
|---|---|---|
| **1 — authority without accounts** | someone cannot be given real, bounded authority without a row in a users table — falsified early by the share link, later and harder by the chat extension | Phase 1 (share link), confirmed Phase 4 |
| **2 — isolation is an operator choice** | the second, maximally different driver requires changing the driver interface | Phase 3 (partially observable in Phase 0) |
| **3 — the diff is a first-class object** | Plan computation is too slow for interactive use, or common operations have to bypass it | Phase 1 |
| **4 — correct tenancy gives reselling free** | any feature has to be written whose purpose is reselling | Phase 2 |

Bet 3 is testable earliest and is the one whose failure would be most expensive, since every other
document assumes Plans are cheap enough to compute on every change. If Phase 1 shows they are not,
that is the moment to find out — not Phase 4.

---

## 6. Risks, ordered

**1. Capability-based authorization has no precedent in this market.** §1 of `08-identity.md` states
this up front. Every competitor uses roles, every user expects roles, and the mitigation is a
presentation layer — `GrantTemplate` rendered as something role-shaped — rather than a schema change.
If that presentation does not land, the model is right and unusable, which is the same as wrong.
*Resolves: Phase 2.*

**2. Funding.** §9 of `19-governance.md` is honest that P10 removes every revenue mechanism the
incumbents use and that projects have died in this position. Structural mitigation: Korpis operates
no infrastructure, so its costs are near zero. *Never fully resolves.*

**3. Scope against a small team.** Twenty-one documents is a large model. The mitigation is Rules
A–D: the expensive-to-retrofit parts are small and early, and the large parts — drivers, storage
classes, exposure modes, surfaces, extensions — are additive behind interfaces that Phase 0 and
Phase 3 prove. *Continuously managed.*

**4. Plan latency.** Bet 3's failure mode. *Resolves: Phase 1.*

**5. The protocol freezing wrong.** Rule B puts the protocol first, which means it is designed with
the least implementation experience. The conformance suite and the N-2 window bound the damage;
nothing eliminates it. *Resolves: Phase 3, when the second driver and migration stress it.*

---

## 7. What building it requires

Not a large infrastructure — but not one machine the whole way either, and the point at which a
second and third are needed is specific.

**Phases 0–1: one machine.** Everything runs as VMs on it: PostgreSQL, the control plane, three or
four agents, a Firecracker or two.

| | Minimum | Comfortable |
|---|---|---|
| CPU | 8 physical cores | 12–16 |
| RAM | 32 GB | **64 GB** |
| Disk | 1 TB NVMe | 1 TB NVMe **plus a separate second disk** |
| Virtualization | VT-x/AMD-V with **nested virt** | — |
| Kernel | 6.1 LTS or newer | 6.6+ |

Three of those are load-bearing rather than preference:

- **Nested virtualization is not negotiable.** Bet 2 — isolation strength as an operator choice — is
  untestable without actually running Firecracker and QEMU, inside VMs. Most cheap VPS products do
  not offer it, which is why the primary machine is owned hardware rather than rented.
- **RAM is the binding constraint**, because microVM density and the hibernation claim of §6 of
  `22-first-party.md` are both memory questions.
- **A separate disk is genuinely needed.** §3 of `06-storage.md` is largely about `EDQUOT` arriving
  in the write path and about inode exhaustion, and verifying either means filling a filesystem on
  purpose.

The same machine is the CI runner. The conformance suites want VMs across a kernel matrix, which is
the second reason for nested virt.

**Phase 2 onward: two more machines, in two locations.** One box stops being enough at a specific
point, for specific reasons — migration across a real network rather than a virtual bridge, lease
fencing under a real partition, an edge with a real public address, and the nftables-versus-XDP
crossover of §11.1 of `07-networking.md`, which is a measurement that means nothing taken on a
virtual bridge. Modest machines suffice; no nested virt is needed, since tier work stays on the
primary. Plus one public IPv4, an IPv6 block, a domain, and any cheap S3-compatible bucket for
backup-target behaviour.

**Later and narrow:** one GPU, rented by the hour, for the device model of §6 of `05-scheduling.md`
in Phase 3. One Windows Server in Phase 5, where §13.5 of `17-security.md` observes that the
isolation table has no honest Windows column yet — that column gets written on that machine.

**Not needed, and worth saying because the reflex is to acquire them:** a Kubernetes cluster, a
managed database, separate CI infrastructure, or a load-generation fleet. The scale assumptions in §8
of `03-state.md` are simulable on one box, and §1 of `18-operations.md` guarantees the single-machine
path stays real precisely so this stays true.

---

## 8. Not on the critical path

These can be built at any time by anyone and block nothing: additional recipes, themes, translations,
additional exposure modes, additional storage classes, additional identity providers, SDKs in further
languages, dashboards, and every extension in §2 of `16-extensions.md` beyond the four that validate
the contract.

That list is deliberately long. It is what Rule D buys: once an interface has been proven by a second,
maximally different implementation, the third through twentieth are contributions rather than
architecture.

---

## 9. Status

| Document | State |
|---|---|
| `research/evidence.md` | complete — evidence, confidence-tagged, rules K-1…K-12 |
| `00-overview.md` … `20-roadmap.md` | complete — the core twenty-one |
| `21-stack.md` | complete — languages, derived from where the boundary sits |
| `22-first-party.md` | complete — the recipe and extension set, and what it proves |
| `23-walkthroughs.md` | complete — seven traces, nine findings, all patched |
| `README.md` | complete — entry point |

Roughly forty numbered open questions remain, distributed across the documents, each assigned to the
one that will settle it. They are the design's honest edges, not omissions: an open question with an
owner is a decision deferred deliberately, and every one of them was recorded where the person who
hits it will find it.

**Everything that blocks Phase 0 is closed.** The intent body's representation (§3.1 of
`03-state.md`), whether the agent caches grants (§10 of `02-architecture.md` — it does not; it
verifies tokens), break-glass access during an outage (a local socket, not a listener), and every
component's language (`21-stack.md`) are settled. What is left open belongs to phases that have not
started.

The model is drawn. What remains is to build it.
