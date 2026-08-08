# Threat Model and Security Architecture

**Status:** design **Date:** 2026-08-07 **Depends on:** [`04-runtimes.md`](./04-runtimes.md),
[`08-identity.md`](./08-identity.md), [`02-architecture.md`](./02-architecture.md) **Resolves:**
open question 3 of `14-streams.md`; open question 5 of `15-observability.md` **Implements:**
Principles P3, P4; Rules K-1, K-2, K-3

---

## 1. The founding assumption

**Tenant code is hostile.** Not "may be untrusted", hostile, deliberately, by default, in every
document above this one.

This is not pessimism about customers. It is the accurate description of a platform whose entire
purpose is running software the operator did not write, chosen by someone the operator has never
met, frequently a modded game server assembled from plugins downloaded from a forum. The realistic
question is never whether tenant code will attempt to escape; it is whether it succeeds, and what
it reaches when it does.

Every panel in this market that has been broken was broken by getting this backwards, by treating
tenant input as data that needs checking rather than as an adversary that needs confining.

---

## 2. Adversaries, named

Threat models fail by being written about "attackers" in the abstract. These are the specific
parties whose interests diverge from the operator's, each of which forces a different mechanism:

| Adversary | Wants | Primary defence |
|---|---|---|
| **A tenant's workload** | escape isolation, read the host, reach other tenants | kernel confinement (§4), isolation tier (§5) |
| **A tenant's user** | reach objects outside their scope | grants, evaluated server-side on every call (§6 of `08-identity.md`) |
| **A sub-tenant** | escalate to their reseller's authority | attenuation, a grant cannot produce a stronger child (P6) |
| **A recipe author** | run code on nodes via a popular recipe | no arbitrary code in the install DSL (§4 of `09-recipes.md`), signing, digest pinning |
| **An extension author** | exceed granted authority | extensions are workloads with declared grants (§4 of `16-extensions.md`) |
| **An unauthenticated outsider** | reach the API, the agent, or a workload | no inbound agent ports (§4 of `02-architecture.md`), default-deny egress and ingress |
| **A compromised node** | forge observations, reach other nodes, keep running after eviction | single-writer ownership, lease epochs, no node-held long-lived credentials (§7) |
| **A compromised control plane** | everything | acknowledged as total; blast radius and recovery in §9 |
| **Discord, or an external IdP** | none | treated as a third party, not as an authority (§3 of `12-surface-discord.md`) |
| **The operator** | none | **tenants are not protected from the operator.** Stated in §8, honestly. |

---

## 3. What the evidence says to build

Six of the twelve published Wings advisories are one class: **a privileged process defending
tenant-controlled paths by inspecting strings.** Symlinks, hardlinks, TOCTOU races, `..` traversal,
and mount tricks each beat string validation, and each did.

One more is worth naming because it is a different lesson. GHSA-pfvc-3p5h-x7h6 (Critical): tenant-
controlled egg template variables were rendered in a context where they could reach the daemon
token and registry credentials. The failure was not the templating engine. It was that the
credentials were reachable from the render context at all.

Two rules follow, and they are P3 and K-2:

> **Confine with the kernel, not with strings.** Validation is defence in depth; it is never the
> defence.
>
> **A secret that is not present cannot be leaked.** Reduce what is reachable before hardening the
> path to it.

---

## 4. Kernel confinement, mechanism by mechanism

Each of these stops a specific thing. Listing them without saying what they stop is how security
sections become decoration.

| Mechanism | Stops |
|---|---|
| user namespaces | root inside the workload being root on the host |
| mount namespaces | seeing, or reaching, any path outside the workload's tree |
| **Landlock** | the filesystem worker touching anything outside the tenant subtree, *including via symlink* |
| `openat2(RESOLVE_BENEATH)` | traversal and symlink escape at the syscall boundary, without a race window |
| seccomp | reaching syscalls the workload has no business calling, shrinking kernel attack surface |
| capability drop | every privileged operation not explicitly required |
| cgroups v2 | CPU, memory, PID, and I/O exhaustion (§10) |
| network namespaces + nftables | reaching other tenants, the host, or the metadata address (§6 of `07-networking.md`) |
| no-new-privs | setuid binaries inside the workload regaining privilege |
| read-only and `noexec` mounts | writing then executing in places that should hold only data |

**Path handling is the load-bearing case.** Korpis does not check a path and then open it, that is
the TOCTOU window that produced half the advisories. It opens with `RESOLVE_BENEATH` inside a mount
namespace, under a Landlock ruleset, from a process that has no access to anything else. If the
kernel resolves the path, it was legal; if it does not, no check was needed. The string is never
consulted as a security decision.

---

## 5. Isolation tiers, described honestly

K-10 makes isolation strength a per-workload operator choice. That choice is only meaningful if the
tiers are described truthfully, including what each one does **not** stop. P4 applies to security
claims more than anywhere else.

| Tier | Boundary | Stops | Does **not** stop |
|---|---|---|---|
| `process` | namespaces, cgroups, Landlock, seccomp, no container image | filesystem escape, resource exhaustion, network reach | kernel LPE, CPU side channels |
| `container` | the above, plus image isolation and a reduced syscall surface | the same, plus most host-filesystem exposure | **kernel LPE**, CPU side channels |
| `microvm` | a separate kernel (Firecracker) | kernel LPE, most side channels, almost all escape classes | hypervisor vulnerabilities, hardware-level channels |
| `vm` | a separate kernel with full device emulation (QEMU/KVM) | as microvm | as microvm, with a larger device attack surface |

**A shared-kernel tier does not survive a kernel privilege-escalation vulnerability.** That is true
of every container platform, it is true here, and it is why the tier is an operator choice rather
than a platform property. An operator running their own services picks `process` and gets density;
an operator running arbitrary customer code picks `microvm` and pays for it in memory and boot
time. There is no configuration that gives both, and Korpis does not imply one exists.

§2 of `04-runtimes.md` explicitly refuses to ship an unhardened container tier, because a tier that
looks like isolation and is not is worse than a tier honestly labelled `process`.

---

## 6. Privilege separation inside the agent

The agent is where tenant data meets root, so it is decomposed until no single process holds both
(§6 of `02-architecture.md`):

```
supervisor              unprivileged. talks to the control plane, holds no root, touches no tenant path.
privileged helper       root, tiny, no tenant input. performs an enumerated set of operations.
filesystem workers      one per tenant, inside that tenant's mount namespace, Landlocked to its subtree.
runtime drivers         separate processes, each holding only its own driver's privileges.
```

The argument is a privilege union. A single process performing all four roles holds the union of
their privileges at all times, so any bug anywhere in it is a bug in a root process with access to
every tenant's data. Split, a bug in the filesystem worker is a bug confined to one tenant's
subtree by the kernel, and a bug in the supervisor is a bug in an unprivileged process.

Drivers are separate for the same reason (§4.3 of `04-runtimes.md`): the QEMU driver needs
privileges the OCI driver does not, and a merged runtime would grant both to both.

---

## 7. Secrets

The principle from §3: reduce reachability first.

- **Nodes hold no long-lived registry credentials.** §7 of `02-architecture.md` issues short-lived,
  digest-scoped pull tokens instead. A compromised node yields a token that expires and that can
  pull one thing.
- **Nodes hold no object-storage credentials** by default, for the same reason (§9 of
  `14-streams.md` carries the open form of this for stream offload).
- **Tenant secrets never enter a template context.** §6 of `09-recipes.md` renders from a flat,
  pre-resolved, enumerated variable map in a logic-less language inside the tenant's namespace.
  There is no ambient scope to walk, which is what defeats the GHSA-pfvc-3p5h-x7h6 class rather
  than escaping harder.
- **Secrets are redacted before persistence**, on the node, in the write path (§7 of
  `14-streams.md`), and that redaction is labelled best-effort in the interface, because pattern
  matching cannot catch a workload that encodes its own secret first.
- **Capability tokens are short-lived**, travel in the URL fragment when shared (§6.1 of
  `08-identity.md`), and never reach an extension's frame (§5 of `16-extensions.md`).
- **The agent's own key** is per-node, rotatable, and its compromise is bounded by §9.

---

## 8. What the operator can see, stated plainly

Tenants are not protected from the operator. The operator controls the hardware, the kernel, the
storage, and the network; any claim otherwise would be false, and P4 forbids it.

What Korpis provides instead is **visibility**, which is achievable and is what actually changes
behaviour:

- Operator access to a tenant's console, files, or backups produces an `Effect` naming the
  operator, the action, and the object (§4 of `15-observability.md`).
- **That effect is visible to the tenant** (§7 of `08-identity.md`), in their own audit view.
- Break-glass access is the same mechanism, not a bypass, a grant, issued, time-bounded, logged (§6
  of `08-identity.md`).

> **Resolving open question 5 of `15-observability.md`**, which accesses are exempt from tenant
> visibility: **none are.** An operator debugging a node-wide failure touches many tenants and
> produces many effects; that is correct and is not a reason for an exemption. Exemptions are how
> audit logs become decorative, and an operator who genuinely needs unobserved access has root on
> the machine and does not need Korpis to provide a supported path to it. Volume is a UI problem,
> solved by grouping effects under the operation that caused them, not a reason to hide them.

> **Resolving open question 3 of `14-streams.md`**, tenant-chosen short retention versus operator
> need for evidence: the operator may set a **retention floor**, a minimum below which a tenant
> cannot go, and that floor is **disclosed to the tenant** in the interface alongside the retention
> setting. An operator cannot silently retain more than they declared. This is the same shape as
> `min_retention` on volumes (§5.3 of `06-storage.md`), and the disclosure is the part that makes
> it legitimate rather than surveillance.

---

## 9. Compromise, and what it costs

**A compromised node.** The blast radius is that node's tenants, plus whatever its credentials
reached, which §7 has deliberately made small. Recovery: revoke the node key, and **increment the
lease epoch**. §4.5 of `02-architecture.md` and §8 of `03-state.md` make epochs monotonic and
fencing mandatory, so a compromised agent that keeps running cannot reacquire ownership of
anything, and cannot write an observation the control plane will accept. Workloads are rescheduled
elsewhere from their intents, which are unaffected. The node is not trusted to clean itself up; it
is fenced and rebuilt.

### 9.1 Egress is enforced by the party being contained

> Resolves open question 6 of `07-networking.md`; Finding 8 of `23-walkthroughs.md`.

Egress rules are applied at the source node by that node's agent, and a compromised agent can
ignore them, which puts a control in the hands of the adversary it is meant to constrain.

The honest framing first: **a compromised agent means root on that machine.** At that point the
attacker can already rewrite nftables, read every tenant's data on the node, and forge
observations. Egress is the least of what has been lost, and containing the outbound traffic of a
machine whose root you no longer hold is defending the wrong thing. The answer to node compromise
is the paragraph above (revoke, fence, rebuild) not a cleverer firewall.

That said, one real improvement was being left on the table:

**East-west egress is enforced at both ends.** A workload on a compromised node reaching a workload
on a healthy node is stopped by the *destination's* agent, which already maintains a network
namespace and nftables rules for its own workloads. The source enforces for efficiency (drop early)
and the destination enforces because it does not trust the source. Genuine defence in depth at
essentially no cost, and it means a compromised node cannot use the fleet as its lateral path.

**North-south egress is uncontained on a compromised node, and this is stated rather than
implied.** Only the source, or a chokepoint the operator owns, can enforce it, and nothing at this
layer does better.

**So Korpis publishes each node's expected egress profile** in a machine-readable form. An operator
who needs enforcement at a layer the node does not control (a VLAN, a security group, an upstream
ACL) configures it from that profile instead of guessing. Korpis does not operate that layer and
does not pretend to; it supplies the only thing it credibly can, which is an accurate statement of
what legitimate traffic looks like.

**A compromised control plane.** Total. There is no partial story here and inventing one would be
dishonest: the control plane holds the authority to issue grants. What can be bounded is
reconstruction, §8 of `03-state.md` requires restore to fence with a monotonic `max_issued_epoch`
watermark and to rotate the signing key, so a restored control plane can neither issue an epoch a
live agent would accept as current nor honour a token it has no record of, which is what prevents a
compromise or a restore from turning into two authorities.

**A compromised tenant account.** Bounded by that subject's grants, all of which are enumerable,
all of which are revocable individually, and every use of which is in the effect log. There is no
admin flag to inherit.

---

## 10. Resource exhaustion, indexed

Every resource a tenant can exhaust, and where it is bounded. This list exists to be checked; an
unbounded resource is a denial-of-service vulnerability with a friendlier name.

| Resource | Bound | Where |
|---|---|---|
| CPU | cgroup v2 weight and quota | §5.2 of `05-scheduling.md` |
| memory | cgroup v2 `memory.max`, no overcommit on reservations | §5.2 of `05-scheduling.md` |
| processes and threads | cgroup `pids.max`; this is the fork bomb | §5.2 of `05-scheduling.md` |
| disk bytes | filesystem quota in the write path, returning `EDQUOT` | §3 of `06-storage.md` |
| **inodes** | inode quota, a million empty files fills a filesystem with bytes to spare | §3 of `06-storage.md` |
| disk I/O | cgroup v2 `io` weight and limits | §3 of `06-storage.md` |
| backup I/O | the backup process runs in its own throttled cgroup | §3 of `06-storage.md` |
| network bandwidth | shaping per endpoint | §7 of `07-networking.md` |
| **conntrack entries** | per-workload limit, a shared table is a shared failure | §6 of `07-networking.md` |
| ports and addresses | allocation is explicit | §3 of `07-networking.md` |
| log volume | stream rate limit, drop with gap marker, never block | §5 of `14-streams.md` |
| log storage | counts against the tenant's storage quota | §4 of `14-streams.md` |
| metric cardinality | tenant-controlled values are never labels | §3 of `15-observability.md` |
| API calls | rate limits per grant, not per IP | §10 of `10-api.md` |
| control-plane work | expensive operations carry separate budgets | §10 of `10-api.md` |
| **outbound abuse** | egress default-deny, rate limiting, detection | §6 and §8 of `07-networking.md` |

---

## 11. Supply chain

Four things arrive from outside and each is pinned by digest and verified before use:

| | Verification |
|---|---|
| recipes | cosign signature, digest-pinned, `fetch` without a hash is a **parse error** (§4 of `09-recipes.md`) |
| extensions | cosign signature, digest-pinned, declared grants approved at install |
| container images | digest-pinned; tags are resolved once and recorded, never re-resolved silently |
| the Korpis binaries themselves | reproducible builds, signed releases, published SBOM |

Imported Pterodactyl eggs are the deliberately dangerous case, and §9 of `09-recipes.md` handles it
by grading rather than refusing: `clean`, `hashable`, and `unverified`, where `unverified` runs in
a microVM with no network beyond confirmed hosts. An import path that pretends imported artifacts
are trustworthy would undo this entire document.

---

## 12. Disclosure

A published security policy, a contact, and a stated response window exist before the first
release, not because it is best practice, but because K-8 makes governance a day-one decision and a
project without a disclosure path receives its vulnerabilities as public tweets.

Advisories are published with the same detail Korpis' own research used to derive these rules
(`research/evidence.md`): what was reachable, what it affected, which versions, and what the fix
changed. A project that learned this much from reading other people's advisories does not get to
publish vague ones.

---

## 13. Open questions

1. **CPU side channels between tenants in shared tiers.** Not defended against; §5 says so plainly.
   Whether Korpis offers core pinning and sibling-thread exclusion as a declared workload property
   (with the density cost stated) is unresolved, and the honest default is that customers needing
   this pick `microvm`. → here
2. **Hardware attestation.** Verifying that a node runs the kernel and agent it claims would raise
   compromised-node detection substantially and drags in TPM, measured boot, and a key-management
   story disproportionate to a single-operator install. → `18-operations.md`
3. **Confidential computing tiers.** AMD SEV-SNP and Intel TDX would let a tenant be protected
   *from the operator*, which §8 currently states is impossible. That is a real fifth tier and a
   large commitment. → `04-runtimes.md`
4. **Recipe author reputation.** Signing proves who published a recipe, not whether they should be
   trusted. Anything resembling curation collides with "Korpis is not a marketplace" (§2 of
   `00-overview.md`), so the likely answer is operator-managed allow-lists and nothing central. →
   `09-recipes.md`
5. **Windows isolation parity.** K-9 requires the runtime interface to express Windows on day one;
   job objects and server silos are genuinely weaker than Linux namespaces, and §5's tier table has
   no honest Windows column yet. It needs one before Windows ships, not after. → `04-runtimes.md`
