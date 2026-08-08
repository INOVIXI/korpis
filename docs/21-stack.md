# Implementation Stack

**Status:** design **Date:** 2026-08-07 **Depends on:** [`17-security.md`](./17-security.md),
[`02-architecture.md`](./02-architecture.md), [`10-api.md`](./10-api.md) **Implements:** Principle
P3

---

## 1. Why this document exists, and why it comes last

Language and framework choices were deliberately deferred until the constraints were fixed.
Choosing a stack first is choosing the shape of the system by accident: a project that picks a
language before it knows where its security boundary is will draw the boundary where the language
is comfortable.

Now the boundary is known (§6 of `17-security.md`), so the stack can be derived from it rather than
from taste.

**One decision governs everything below: the components differ in kind, so they do not all get the
same language.** The architecture already separates them into processes speaking defined protocols
(§6 of `02-architecture.md`, §4.3 of `04-runtimes.md`), which means a per-component choice costs
nothing structurally. It would be a strange discipline to insist on uniformity across a boundary
the design exists to keep separate.

---

## 2. Decisions

| Component | Language | Size | Changes |
|---|---|---|---|
| Schema and protocol | **Protocol Buffers**, served over **Connect** | none | rarely, additively |
| Control plane (`korpis-server`) | **Go** | largest | constantly |
| Agent supervisor | **Go** | medium | often |
| **Privileged helper** | **Rust** | tiny | almost never |
| **Per-tenant filesystem workers** | **Rust** | small | almost never |
| Runtime drivers | the runtime ecosystem's language | small each | independently |
| Web client | **TypeScript**, static bundle | medium | constantly |
| CLI | **Go**, one static binary | small | often |
| Database | **PostgreSQL** | none |, |

---

## 3. Go for the control plane, and why memory safety is not the argument

The control plane is an HTTP service in front of PostgreSQL. It parses requests, evaluates grants,
computes plans, runs reconciliation loops, and writes rows. It performs no syscall-level work,
holds no tenant filesystem handle, and is not a confinement boundary.

Go wins here on the things that actually matter for this component: Connect and gRPC are
first-class, the PostgreSQL tooling is mature, a single static binary matches §2 of
`18-operations.md`, compile times keep iteration fast on the codebase that changes most, and the
contributor pool is deep, which matters for a project whose governance model (§8 of
`19-governance.md`) depends on outside contribution scaling past two maintainers.

**Rust's guarantees buy little here, and the security thesis places no weight on them.** §1 of
`17-security.md` says tenant code is hostile and §3 says confinement is the kernel's job. The
control plane never touches tenant-controlled bytes in a privileged context. Choosing Rust for it
would be paying iteration speed for a property this component was not relying on.

---

## 4. Rust where the boundary is, and this one is not a preference

The privileged helper and the per-tenant filesystem workers are the two components that hold root
or sit inside a tenant's namespace. They are also, precisely, the code paths whose equivalents
produced **six of Wings' twelve published advisories**. They get a different language, for two
reasons, and the second is the decisive one.

**Small and security-critical.** A few thousand lines each, rarely changed, doing exactly the work
where a memory-safety bug is a tenant escape. This is the ideal shape for Rust even before the
second argument.

**Go cannot do this part correctly without leaving Go.** This is a concrete technical constraint,
not an aesthetic one:

- Namespace operations (`setns`, `unshare`, `CLONE_NEWUSER`) are **per-thread**. The Go runtime
  creates OS threads on its own schedule and migrates goroutines between them. "Enter this mount
  namespace, then do work" is therefore racy in Go, and any goroutine spawned afterwards may run on
  a thread that never entered the namespace.
- `seccomp` filters and Landlock rulesets are likewise applied per-thread, with the same
  consequence.
- `fork` without immediate `exec` is unsafe in Go, which rules out the classic fork → confine →
  exec pattern that is the natural way to build these processes.

`runtime.LockOSThread()` mitigates some of this and is not sufficient. The proof is in the field:
**runc is written in Go and performs its namespace setup in C**, in a constructor (`nsexec.c`) that
runs before the Go runtime initializes, because there was no correct way to do it from Go.

So the real choice is not Go versus Rust. It is *Go plus a non-Go component at the boundary* versus
*Rust at the boundary*, and once a second language is unavoidable, Rust is strictly the better
second language than C for code whose entire job is to be the thing that does not have
memory-safety bugs.

`openat2(RESOLVE_BENEATH)`, Landlock, and seccomp are all directly expressible in Rust with precise
control over which thread is confined and when, which is the mechanism §4 of `17-security.md` names
as load-bearing.

---

## 5. Drivers follow their ecosystems

Drivers are already separate processes with a versioned interface (§4.3 of `04-runtimes.md`), so
each is written in whatever language its runtime lives in:

| Driver | Language | Why |
|---|---|---|
| `oci` | Go | containerd's API and ecosystem are Go |
| `firecracker` | Rust | Firecracker is Rust; its API is a small HTTP surface either way |
| `qemu` | Go or Rust | it spawns and supervises a process; either is fine |
| `native` | **Rust** | it *is* the confinement path, same argument as §4 |
| `winjob` / `winsilo` | Go or C# | Windows APIs; decided when Windows is built |

This is the payoff for the driver interface being a protocol rather than a plugin ABI. A third
party writing a driver in a language nobody here uses is a supported outcome, not an obstacle
(K-7).

---

## 6. The two-language cost, stated

Two languages in one project is a real cost: two toolchains, two sets of idioms, a smaller pool of
contributors comfortable in both, and a boundary that has to be maintained.

It is accepted because the split is **small and stable**. The Rust surface is a few thousand lines
that change rarely, sits behind a defined interface, and is the part where being right matters more
than being fast to change. The Go surface is everything that changes weekly.

The alternative (all Rust) was considered and rejected: it would slow the largest and
most-contributed-to codebase in exchange for a property that codebase does not depend on. The other
alternative (all Go) is not available, per §4.

Worth noting and not worth weighting: Wings is Go, and Calagopus is Rust. Neither fact influenced
this. The split here is derived from where the confinement boundary sits, and it happens to match
neither.

---

## 7. Protobuf and Connect, restated

Settled in §3 of `10-api.md`. One schema generates the Go server, the TypeScript client, and SDKs
in whatever else is wanted, plus the OpenAPI document. Connect gives Connect, gRPC, and gRPC-Web on
one endpoint, reachable from a browser without a proxy and from `curl` with JSON.

The JSON surface being first-class is the ecosystem decision, not a convenience: an integration
market that requires a protobuf toolchain to send one request does not form.

---

## 8. PostgreSQL only, and the friction it creates

Settled in §6 of `03-state.md` and restated because it is the stack's one genuine tension.

The transactional guarantees that make `Intent` and `Effect` honest are not optional, an `Effect`
written in the same transaction as the state change it describes is the entire basis of the audit
model. SQLite cannot provide what §4 of `03-state.md` requires, and MySQL's `jsonb`-equivalent and
`SKIP LOCKED` story is worse for no gain.

The cost is that §1 of `18-operations.md` promises a single operator with one machine can install
Korpis, and a database is a second thing to install. §2 of `18-operations.md` states this plainly
rather than hiding it, and §10's first open question is about reducing the friction without
shipping an unsupervised production configuration nobody tests.

---

## 9. Web client

TypeScript, compiled to a static bundle with no server runtime (§1 of `11-surface-web.md`). The
framework remains open (§10 of `11-surface-web.md`) and the constraints that will decide it are
already fixed: static output, streams rather than polling, correct rendering for a subject holding
an arbitrary grant set, and small enough that a host serving it from a cheap CDN is ordinary.

---

## 10. Open questions

1. **Agent supervisor language. Resolved: Go.** The tempting argument for Rust was that a
   single-language agent would remove a process boundary's worth of marshalling. That argument
   inverts the design: **the boundary between the supervisor and the privileged components is not a
   cost to be optimized away; it is the point** (§6 of `17-security.md`). Merging them to save an
   IPC hop would merge their privilege domains, which is precisely what the split exists to
   prevent.

With that settled, the choice is made on ordinary grounds and Go wins them: the supervisor is the
part of the agent that changes most (protocol handling, reconciliation, adoption, driver
orchestration) and it holds no root and touches no tenant path.
2. **Rust async runtime.** The Rust components are small and mostly synchronous; pulling in Tokio
   for them may be unnecessary weight. → here
3. **Windows driver language.** C# reaches the Windows APIs most directly and adds a third
   toolchain. Deferred to when Windows is actually built (§13 of `17-security.md` has the harder
   question). → `04-runtimes.md`
4. **Web framework.** Carried from §10 of `11-surface-web.md`. → `11-surface-web.md`
5. **Intent body representation. Resolved in §3.1 of `03-state.md`**: proto3 canonical JSON in the
   `jsonb` column, so the `.proto` remains the only schema and the seam closes.
