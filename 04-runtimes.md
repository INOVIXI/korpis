# Runtime Drivers and Isolation Tiers

**Status:** design
**Date:** 2026-08-07
**Depends on:** [`01-model.md`](./01-model.md), [`02-architecture.md`](./02-architecture.md)
**Implements:** Bet 2, Rules K-9, K-10, K-18

---

## 1. The claim being tested

Bet 2 says isolation strength is an operator choice, not a platform property. Every product in this
market froze that decision:

- Pterodactyl and every fork: containers, mandatory. Windows binaries, several anti-cheat systems,
  GPU workloads, and custom kernel modules are permanently unreachable.
- PufferPanel and GameAP: no containers. An exploited game server can reach other services on the
  machine.
- AMP: containers optional, but as a deployment toggle rather than a graded model.

The bet only pays if the interface can express a Windows Job Object and a Firecracker microVM on
the same day it expresses a containerd container — not eventually, but in the first version — since
retrofitting an abstraction means rewriting every consumer of it.

This document is the interface. If it leaks, Bet 2 fails.

---

## 2. Isolation tiers

Four tiers, ordered by the strength of the boundary. The operator chooses one per workload.

| Tier | Boundary | Mechanism | Boot | Overhead | Untrusted code |
|---|---|---|---|---|---|
| `process` | shared kernel | cgroups v2, namespaces, seccomp, Landlock, `unshare` — no image | instant | none | **no** |
| `container` | shared kernel | OCI image, full namespaces, user namespace remapping, seccomp, read-only rootfs | ~100 ms | minimal | with care |
| `microvm` | hypervisor | own kernel, minimal device model | ~150 ms | ~30 MB | **yes** |
| `vm` | hypervisor | own kernel, full device model, own init and lifecycle | seconds | high | **yes** |

**There is no unhardened container tier.** `container` means user namespace remapping, a seccomp
profile, `no-new-privileges`, a read-only root filesystem, and dropped capabilities — always, not as
an option someone can forget to enable. An operator who wants weaker confinement selects `process`
explicitly and knowingly. Making the weak option require a deliberate downgrade rather than an
omission is the difference between a default and a trap (P4).

**The tier is what an operator reasons about.** "This tenant runs code I did not write" →
`microvm`. "This is my own service" → `container`. "This is a trusted helper and I need bare
performance" → `process`. "The customer bought a VPS" → `vm`. That sentence is the entire user-facing
model; drivers are an implementation concern below it.

---

## 3. Drivers

A driver implements one or more tiers on one operating system.

| Driver | Tiers | OS | Notes |
|---|---|---|---|
| `oci` | `container` | Linux | containerd + runc. **The day-one driver** (Rule K-18) |
| `native` | `process` | Linux | Landlock LSM + cgroups v2 + `unshare`, no container runtime dependency — the design Nomad's `exec2` demonstrates |
| `firecracker` | `microvm` | Linux | untrusted code, fast boot, minimal attack surface |
| `qemu` | `vm`, `microvm` | Linux | full virtualization; the `machine` workloads of `01-model.md` |
| `winjob` | `process` | Windows | Job Objects + restricted tokens |
| `winsilo` | `container` | Windows | Server Silos via the Host Compute Service |

Rule K-18 is binding: `oci` ships excellent before anything else ships adequate. VirtFusion beat
entrenched incumbents by supporting one hypervisor extremely well rather than five acceptably. The
interface must accommodate all six from the first version; the implementations arrive in order of
evidence.

### Drivers are separate processes

Each driver runs as its own process, connected to the agent supervisor over a local socket.

This is not modularity for its own sake. It follows directly from §6 of `02-architecture.md`: no
component holds more privilege than its job requires. The `qemu` driver needs device access and KVM;
the `oci` driver needs containerd; the `native` driver needs almost nothing. In one process the
agent would hold the **union** of every driver's privileges, and the union is close to root.

Three further properties come free: a driver crash does not take down the agent or the workloads it
supervises; a third party can ship a driver without being merged into Korpis; and a driver can be
upgraded independently of the agent.

---

## 4. The interface

### 4.1 Capabilities are declared, never inferred

```
Capabilities
  tiers                []Tier
  console              []ConsoleMode      none | logs | interactive | serial
  exec                 bool               run a command inside a running workload
  signals              []SignalName       driver-defined names, not POSIX numbers
  hot_resize           []ResourceKind     which resources change without a restart
  hot_volume_attach    bool
  runtime_snapshot     bool               capture execution state, not just disk
  live_migrate         bool
  gpu                  []GPUMode          none | passthrough | mediated | timeslice
  rootfs_sources       []SourceKind       oci_image | directory | disk_image | iso
  network_modes        []NetworkMode      veth | tap | host | none
  config_schema        JSONSchema         driver-specific configuration
```

The control plane **asks** and never assumes, following Nomad's task driver interface. A version
number is not a capability statement: two builds of the same driver on different kernels have
different capabilities, and inferring from a version is how a scheduler places a workload on a node
that cannot run it.

`config_schema` is the honest part of the abstraction. A VM needs a kernel, a bootloader, and
cloud-init; a container needs none of those. Core does not model that difference — the driver
publishes a schema, the recipe and the operator fill it in, and Korpis validates against it
without understanding it. **The lifecycle is common; the configuration is driver-declared.** An
abstraction that claimed to unify the configuration too would be a lie that leaks on the first
unusual workload.

### 4.2 Lifecycle

```
Prepare(Intent)          → PreparedSpec | Rejection    // pure: validates, resolves, no side effects
Create(PreparedSpec)     → RuntimeID
Start(RuntimeID)
Stop(RuntimeID, signal, grace)
Kill(RuntimeID)
Destroy(RuntimeID)                                     // release all resources

Recover()                → []RuntimeID                 // adoption after agent restart
Inspect(RuntimeID)       → RuntimeStatus
Wait(RuntimeID)          → ExitStatus
Stats(RuntimeID)         → ResourceUsage

Signal(RuntimeID, SignalName)
Exec(RuntimeID, Command) → Stream                      // capability-gated
AttachConsole(RuntimeID) → Stream                      // capability-gated
Resize(RuntimeID, Resources)                           // capability-gated
```

Three of these carry more weight than the rest.

**`Prepare` is pure.** It validates that this driver on this node can satisfy this intent, resolves
references, and returns either a prepared spec or a rejection with a reason — changing nothing. This
is what lets the Planner produce a plan that is known to be satisfiable before anyone approves it,
and it is what makes dry-run genuinely free (P5). A `Prepare` with side effects would make every
dry-run a partial apply.

**`Recover` is not optional.** It is the adoption primitive of §4.4 of `02-architecture.md`. A
driver that cannot enumerate and re-adopt the objects it created is a driver that turns every agent
restart into a customer outage. It belongs in the interface, not in each implementation's good
intentions.

**`Create` and `Start` are separate.** Resources are allocated, volumes mounted, and the network
attached before anything executes. This makes failures happen at a point where nothing is running
yet, and it is a prerequisite for Windows, where creating a job object and starting a process in it
are genuinely distinct operations.

### 4.3 What the interface must never assume

This list exists because these assumptions are invisible until they are load-bearing, and each one
would silently close a tier. Rule K-9 is satisfied by this list, not by intent.

| Never assume | Instead | Closes if violated |
|---|---|---|
| POSIX signal numbers | driver-declared signal **names** | Windows |
| uid/gid identity | an opaque `SecurityContext` the driver interprets | Windows |
| cgroups | resources declared semantically (bytes, millicores, IOPS); the driver maps them | Windows, VMs |
| a PID identifies the workload | `RuntimeID` is an opaque driver-issued string | VMs, Windows |
| one filesystem tree the agent can assemble | mounts declared as specs the driver realizes | VMs, microVMs |
| the workload is a process the agent's kernel can see | liveness comes from `Inspect`, never `kill(pid, 0)` | VMs, microVMs |
| a command line starts the workload | the rootfs source declares its own entry semantics | VMs, ISOs |
| the agent shares a filesystem with the workload | file access goes through the tenant's own worker (§6 of `02-architecture.md`) | microVMs, VMs, Windows |
| stdout/stderr are the output | console mode is declared; serial and VNC are console modes | VMs |

The last two are the ones that quietly fail. An agent that reads a workload's files by opening a
host path works perfectly for containers and is impossible for a microVM — and by the time anyone
notices, that assumption has spread through the file manager, the backup system, and the recipe
installer.

---

## 5. Health across tiers

`kill(pid, 0)` is not a health check, and for half the tiers it is not even possible.

| Tier | What "the process is alive" tells you |
|---|---|
| `process` | the process exists — nothing about whether it works |
| `container` | the container's init exists — nothing about the application |
| `microvm` / `vm` | **the hypervisor is running.** The guest OS may have panicked twenty minutes ago |

Rule K-16 requires application-level health, and the tier table is the reason it cannot be optional.
Probes are declared by the recipe and executed by the agent from outside the workload:

| Probe | Applies to | Mechanism |
|---|---|---|
| `tcp` | any | connect to an endpoint |
| `http` | `service` | request a path, check status |
| `query` | `console` | a game or application query protocol, provided by an extension |
| `exec` | tiers with the `exec` capability | run a command, check exit status |
| `guest_agent` | `microvm`, `vm` | an in-guest agent reports over a virtio channel |
| `exit` | `task`, `scheduled` | exit status |
| `heartbeat` | any | the workload calls Korpis; absence is the signal |

TCAdmin distinguishes "service monitoring" from "game monitoring" and charges for the distinction.
It is right that they are different, and there is no reason it should be a premium feature.

---

## 6. Content and rootfs

The **agent** owns the node-local content store, not the drivers. Recipes, images, and disk
images are fetched once per node, verified against their digest, and shared by every workload that
references them (Rule K-15).

This is TCAdmin's patch system — download from Steam once, distribute across the network —
generalized and made automatic. Pterodactyl downloads the same bytes again for every server, forever.

Drivers **reference** content by digest and never fetch it themselves. Consequences:

- One fetch, one verification, one signature check per digest per node, regardless of driver.
- Layer and block sharing across tiers: a `container` and a `microvm` built from the same base share
  storage.
- A compromised driver cannot pull arbitrary content — it has no registry credentials and no network
  path to a registry (§7 of `02-architecture.md`).
- Prefetch is a scheduling concern: the control plane can hint a digest to a node before placing a
  workload there, so start latency is not a download.

---

## 7. What is deliberately not in the interface

**Live migration is not a driver method.** It is an orchestration built from primitives —
`runtime_snapshot`, volume snapshot, content pre-staging, lease transfer — sequenced by the agent and
control plane. Making it one call would mean each driver reimplements the resumable, verifiable,
reversible state machine Rule K-4 requires, and would then implement it differently and wrongly. The
driver contributes capabilities; `05-scheduling.md` owns the sequence.

**Networking is not a driver method.** Drivers declare which network modes they support and attach
to an interface the agent has already created. A driver that built its own networking would mean six
implementations of endpoint semantics, TLS, egress policy, and traffic accounting. `07-networking.md`
owns that.

**Backup is not a driver method.** It follows from the storage class, not the runtime (Rule K-5).
`06-storage.md` owns it.

The pattern: a driver owns the **execution boundary** and nothing else. Every time something else is
pushed into the driver interface, it becomes something that must be implemented N times and will
behave N different ways.

---

## 8. Conformance

Rule K-7 requires a third party to be able to write a conforming agent. The same must hold for
drivers, or the interface is documentation rather than a contract.

A published **conformance suite** defines what a driver must do:

- every lifecycle method, including failure paths and double-invocation
- `Recover` after a supervisor kill, an OOM kill, and a host reboot — a driver that adopts correctly
  after a clean restart and not after a crash is worse than one that never adopts, because it fails
  only in the situation that matters
- capability honesty: a driver claiming `hot_resize` is tested resizing under load
- resource enforcement: declared limits are verified against actual kernel or hypervisor enforcement
  (Rule K-3 — a limit that is not enforced is a lie in a text field, and this is where that is
  caught)
- isolation: tier-appropriate escape attempts must fail

A driver is conforming if it passes. That is the whole gate. This is how an ecosystem of drivers
exists instead of a series of forks.

---

## 9. Open questions

1. **GPU allocation model.** Passthrough is exclusive, mediated devices (vGPU) partition, and
   timeslicing oversubscribes. These have different scheduling semantics — a GPU is a resource
   dimension in `Quota`, a placement constraint, and a driver capability at once, and it is not
   clear it can be one field. → `05-scheduling.md`
2. **Nested virtualization.** If a node is itself a VM — which is common — `microvm` and `vm` tiers
   require nested virtualization that many providers disable. The scheduler must know this, so it is
   a driver capability determined at runtime rather than a static declaration. → here
3. **Who owns the guest agent for `vm`?** In-guest health, hot resize, and clean shutdown all need
   one. Building it means shipping software into customer VMs and maintaining it across guest
   operating systems. Not building it means VMs are second-class. Neither is comfortable. → here
4. **`process` tier and recipes.** The `native` driver has no image; a recipe for it must
   deliver a directory tree and an entry point. Does that reuse the OCI image format as a delivery
   mechanism (unpacked, not run), or is there a second artifact kind? The first keeps one content
   store and one signing path. → `09-recipes.md`
5. **Windows content store.** Windows container images and Linux OCI images share a format but not
   layers or storage drivers. Whether one content store abstraction covers both, or a Windows node
   runs a parallel one, affects Rule K-15's "fetched once per node". → here, when `winsilo` is
   designed
