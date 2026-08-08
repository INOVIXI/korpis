# Evidence

**Status:** research input for the Korpis design **Date:** 2026-08-07

Korpis departs from how this class of software is normally built, and a design document that says
"we did it differently" without saying *why* is just taste. This is the file that keeps the rest
honest: every rule in `00-overview.md` §6 traces to something specific and citable here.

**On tone, because this matters.** The systems examined below are successful. Pterodactyl in
particular solved a real problem for a large number of people and is why this market exists in a
usable form at all; a great deal of what Korpis does is Pterodactyl's idea. Every advisory listed
here was responsibly disclosed and fixed by maintainers doing unpaid work.

The point of collecting them is not that these projects have bugs, everything has bugs. It is that
**certain failures recur**, in the same shape, across years and across independent codebases, and
recurrence is the signature of an architecture that invites the failure rather than of a developer
who was careless. Those are the ones worth designing around, and the only way to tell them apart
from ordinary bugs is to count.

If you maintain one of these projects and think something here is wrong or unfair, please open an
issue. Being accurate matters more to this document than being right.

Every claim is tagged with a confidence level:

- **[V]** Verified against a primary source (GitHub issue, official docs, security advisory).
- **[R]** Reported by a secondary source that looks credible but was not independently confirmed.
- **[W]** Weak: appeared only in SEO/marketing content. Recorded for completeness, not to be relied
  on.

A note on method: a large share of search results for this topic are hosting-company blog spam.
Several contained outright errors (one claimed Wings is a Node.js process; it is Go). Claims from
those sources were either dropped or demoted to **[W]**.

---

## 1. Pterodactyl

### 1.1 The security record points at one architectural flaw, not ten bugs

Published Wings advisories: **[V]**

| ID | Severity | Title |
|---|---|---|
| GHSA-66p8-j459-rq63 | Critical | Improper link resolution allowing deletion of host files/directories |
| GHSA-494h-9924-xww9 | Critical | Symlink race in server filesystem |
| GHSA-p744-4q6p-hvc2 | Critical | Escape to host from installation container |
| GHSA-pfvc-3p5h-x7h6 | Critical | Wings exposes node configuration secrets through egg configuration-file templating |
| GHSA-gqmf-jqgv-v8fw | High | Arbitrary file write/read |
| GHSA-2497-gp99-2m74 | High | Endless reprocessing of activity log data (SQLite parameter limit) |
| GHSA-ghrq-5wpp-hxx5 | High | Crafted SFTP handshake packet causes denial of service |
| GHSA-rhq6-9rgh-v45c | Moderate | Chmod can change permissions of files outside the server container |
| GHSA-qq22-jj8x-4wwv | Moderate | SSRF during remote file pull |
| GHSA-q6hh-gp44-4hcm | Moderate | Malformed parsed config files cause Wings OOM |

Additional: CVE-2023-32080, install-script-controlled command execution on the host, escaping the
container; fixed in Wings 1.11.6, backported to 1.7.5. CVE-2023-25152, symlink following; fixed in
1.7.4. Pre-1.4.4 Wings had improper container process limits allowing resource exhaustion of the
physical host. **[V]**

**The lesson is not "Wings has bugs."** Six of the twelve issues above are the same class: a
privileged process performs filesystem operations on paths supplied by an untrusted tenant, and
defends itself by *validating path strings*. Path-string validation loses to symlinks, races, and
`..` every time, it is a losing strategy, not a series of oversights.

> **Rule K-1.** The component that touches tenant files must never hold host privilege. Filesystem
> access is confined by the kernel, a per-tenant mount namespace plus `openat2(RESOLVE_BENEATH)` or
> an equivalent, never by string inspection. Path validation is defence in depth, not the defence.

The critical advisory GHSA-pfvc-3p5h-x7h6 deserves separate attention: egg configuration-file
**templating** could be made to render node secrets, the daemon token and Docker registry
credentials, into a file the tenant can read. A secondary source reports this as CVE-2026-52855,
CVSS 9.9, fixed in Wings 1.12.3 **[R]**. The root cause is that the template evaluation context and
the daemon's secret context were the same context.

> **Rule K-2.** Tenant-authored templates evaluate in a sandbox with an explicit, allow-listed
> variable set. There is no ambient scope to leak from.

### 1.2 Resource limits are advertised but not enforced

- **Disk quota is not enforced.** Issue #4554: an 858 GB file existed on a server configured with a
  1 GB disk limit. The issue was **closed as not planned**. **[V]** Related: #2638 (servers are not
  stopped when the disk limit is reached), #5186 (panel misreports disk usage), #1050 (request for
  real disk restrictions).
- **No disk I/O limit.** #2798: backups to spinning disks cause iowait spikes that degrade every
  other server on the node. **[V]**
- **No bandwidth limit and no traffic accounting.** #1871 (limit bandwidth per server), #3352 (mb/s
  cap "to prevent people from killing the network"), #3780 (network usage chart). **[V]**

CPU and memory are handled through cgroups. Disk capacity, disk I/O, and network are not. On a
multi-tenant node that means one tenant can degrade or halt every other tenant, and the operator
has no evidence of who did it.

> **Rule K-3.** Every limit Korpis displays is enforced by the kernel, or Korpis does not display
> it. Capacity, IOPS, bandwidth, and PIDs are first-class limits alongside CPU and memory, and each
> is metered.

### 1.3 Server transfer has been broken for years

Transfer between nodes is the mechanism for hardware maintenance, node decommissioning, and
capacity rebalancing. Its issue history: **[V]**

- #18, #280: original feature requests
- #3015: transfers fail with a "walking error"
- #3332: `processFailedTransfer()` fails on a NULL argument
- #4244, #4614, #4644: assorted transfer failures
- #4505: **a failed transfer deadlocks the server**: unusable from client and admin panel alike,
  state cannot be overridden, every state change returns HTTP 409. Recovery requires truncating the
  `server_transfers` table by hand.
- #5429: symlinks are silently dropped, so the server will not boot at the destination

An independent developer cites a transfer bug that stayed broken for 40 days with a fix available
and no guidance issued to affected installations. **[R]**

The root cause is structural: transfer is written as a single-shot imperative operation. There is
no resumable state machine, no verification step, and no defined rollback, so any interruption
leaves the system in a state the model cannot express.

> **Rule K-4.** Migration is a first-class, resumable, restartable job with explicit phases
> (snapshot, ship, verify, cut over, release), each independently retryable. No operation may leave
> a workload in a state the model cannot represent. There is always a path back.

### 1.4 Backups do not scale

- **Full backups only.** No incremental support; open feature requests for incremental (#5493) and
  for Borg (#4715). A 40 GB server means writing 40 GB every time. **[V]**
- **S3 uploads fail above 5 GB**: multipart upload is not used (#2599). **[V]**
- **Backups stage to local disk first** (#3846), so the node needs free space equal to the backup.
  **[V]**
- **Backup settings cannot vary per location** (#4411). **[V]**
- Combined with 1.2: backup I/O is unthrottled and degrades the whole node (#2798).

> **Rule K-5.** Backup is a consequence of the storage design, not a feature bolted on top. Choose
> a filesystem that snapshots (ZFS/btrfs subvolume per workload), take the snapshot instantly, then
> ship it content-addressed, deduplicated, and encrypted. The workload never stops, the node never
> stages a full copy, and incremental is the default rather than a feature request.

### 1.5 There is no extension API, so the ecosystem patches core files with `sed`

Blueprint is the de facto extension framework, and its own documentation describes the mechanism:
extensions ship shell scripts that run at install/update/remove/export time and modify
Pterodactyl's core files by **targeted string substitution**, replace `foo` with `bar` on install,
replace `bar` with `foo` on removal. Extensions are told not to overwrite files, not to run remote
scripts, not to obfuscate code, and to stay inside `$PTERODACTYL_DIRECTORY`. **[V]**

Every one of those constraints is a social rule, not a technical boundary. The framework cannot
enforce any of them: the scripts run with the installer's privileges.

Observed consequences: extensions break on panel updates, webpack module resolution errors after
upgrades, theme conflicts between extensions, extension settings lost on reinstall unless the
developer persisted them to the database, and installs that wedge in a "you need to finish
installing Blueprint" state. **[V]**

`update.sh` runs from the *pre-update* copy of the extension, not the new one, a detail that
guarantees subtle breakage for anyone who does not read that specific documentation page. **[V]**

An independent developer building a from-scratch alternative observed that virtually every serious
deployment depends on Blueprint, and concluded core extensibility should have been present from day
one. **[R]**

> **Rule K-6.** Extensions are separate processes speaking a versioned contract, holding
> narrowly-scoped tokens, and contributing UI through declared slots. An extension must never be
> able to modify Korpis' own files. Core features use the same extension mechanism third parties
> use, if the mechanism is not good enough for core, it is not good enough to ship.

### 1.6 The panel and the daemon were never decoupled

Issue #141, opened **October 2016**, asked for third-party daemon support. Ten years later there is
still exactly one implementation. **[V]** Panel and Wings are coupled by version rather than by
contract, so they must be upgraded in lockstep and no alternative data plane can exist.

> **Rule K-7.** The control plane and the data plane are separated by a published, versioned,
> independently implementable protocol with explicit compatibility guarantees. A third party must
> be able to write a conforming agent without reading Korpis' source.

### 1.7 Governance was the actual failure

Pelican was forked in mid-2024, primarily because Wings had gone a long stretch without updates and
the fork wanted to ship releases and security fixes faster. **[R]** Pelican relicensed the panel
from MIT to AGPLv3, which pushed commercial hosts relying on proprietary differentiation to look
elsewhere. **[R]** Pterodactyl has continued under MIT, and its leadership has publicly framed the
current phase as cleanup and maintenance (patching holes, untangling dependencies, improving
testing) before new features. **[R]**

A candid outside assessment described Pterodactyl's release process as "almost like it's a hobby
project," with weak testing infrastructure and no coordinated triage or review, notable for
software that became an industry default. **[R]**

> **Rule K-8.** License, versioning policy, compatibility guarantees, and a multi-maintainer
> structure are day-one decisions, written down before the first release. An extension ecosystem
> can only exist on top of a stable, promised contract.

---

## 2. What the forks proved

### Pelican: the ceiling of forking

Shipped, by their own comparison: a first-party plugin system, native webhooks, roles and
permissions for admins, tags replacing the nest/location system, Filament rewrites of both client
and admin UI, Vite instead of Webpack, integrated API documentation with broader coverage, OAuth,
PostgreSQL/SQLite/MariaDB drivers, per-user timezones, IPv6 allocations, and servers without
allocations. **[V]**

Still listed as *planned*, not done: **allocation system rework**. **[V]**

That single line is the most informative sentence in this document. Pelican fixed nearly every
surface-level complaint about Pterodactyl and still has not touched the networking model, because
`(ip, port)` allocations are load-bearing for the entire schema. Some things cannot be fixed by
forking.

### Pyrodactyl: you cannot fix architecture from the frontend

Claims a bundle over 170× smaller than other forks including Pelican, load times ~16× faster,
rendering up to 70% faster, cold builds under 7 seconds. **[R]** Real, measurable, and entirely
frontend: Vite, code splitting, chunking, caching. The Laravel backend and Wings daemon are
unchanged.

Pyro subsequently handed Pyrodactyl to the community and announced a new platform built from the
ground up. **[R]** The most capable performance-focused fork concluded that the remaining problems
were not reachable from a fork.

### TheGamePanel: an independent from-scratch attempt

Cited reasons: poorly written code duplicating framework functionality, many logical
inconsistencies, repeated security vulnerabilities and critical bugs, the 40-day transfer bug,
missing core extensibility, and hobby-grade release process. Approach: build from scratch, minimal
dependencies, true modularity where core features use the same extension system plugins use,
self-contained binary via FrankenPHP, while keeping backward compatibility with existing
Pterodactyl infrastructure. **[R]**

**Three independent teams (Pelican, Pyro, TheGamePanel) reached the same conclusion from three
different directions: the architecture cannot be fixed in place.** That is the strongest available
evidence that Korpis should start from a new model rather than a fork.

---

## 3. Competitors

### AMP (CubeCoders): commercial, and holds the market Pterodactyl cannot enter

Runs on **Windows and Linux**, Docker optional, supports 400+ games, and is widely reported as
easier to set up. Free tier of 2 instances; roughly $10 one-time for 5 instances with no recurring
fee. **[R]**

Windows support is the whole story. Pterodactyl's hard dependency on Docker excludes Windows-only
game binaries, several anti-cheat systems, and workloads needing GPU or custom kernel modules.
**[R]** AMP and TCAdmin hold that segment essentially unopposed by open-source alternatives.

> **Rule K-9.** The runtime is an interface, not a dependency. The interface must be able to
> express a Windows execution model on day one even if no Windows driver ships for a year,
> retrofitting it later means rewriting every consumer.

### PufferPanel: the counter-example

Go, lightweight, and Pterodactyl's own ancestor. Runs game servers as **processes directly on the
host OS** rather than in containers: lower overhead, broader platform compatibility, and
substantially weaker isolation, since an exploited game server can affect other services on the
same machine. **[R]**

Pterodactyl hardcoded containers; PufferPanel hardcoded no containers. Both froze an isolation
decision that belongs to the operator.

> **Rule K-10.** Isolation strength is a per-workload property chosen by the operator, shared
> kernel container, hardened container, microVM, full VM, or bare process, not a property of the
> platform.

### Coolify: the general-purpose comparison

The closest thing to a general-purpose Korpis competitor, and instructive mostly by its limits: no
automatic failover between nodes, nodes must be uniformly AMD64 or uniformly ARM (no mixing), no
seamless migration between self-hosted and managed, a control plane that idles heavy because it
runs its own supporting containers, and no managed databases. **[R]** Eleven critical
vulnerabilities are reported as disclosed in January 2026 including command injection, root key
leakage, and a **default root-level process model**. **[W]**, single-source, treat as directional
only.

Architecturally Coolify assumes one workload shape: an HTTP service behind a proxy. That assumption
is as limiting as Pterodactyl's "long-running process with a console," just in the opposite
direction.

> **Rule K-11.** Workload shape is a field on the workload, not an assumption baked into the
> platform. `console`, `service`, `job`, `cron`, `stateful`, and `machine` are all first-class, and
> they share one scheduler, one storage layer, one network layer, and one identity model.

### Agones: the only mature scheduler model in this space

Google/Ubisoft, announced 2018, a Kubernetes extension for dedicated game servers. Worth stealing
from: **[V]**

- **Fleet**: a set of warm, Ready workloads available for allocation, with a declared replica
  count.
- **Scheduling policy as an explicit choice**: `Packed` bin-packs onto as few nodes as possible
  (right for elastic cloud capacity, enables scale-down); `Distributed` spreads across the cluster
  (right for fixed hardware). Pterodactyl has neither: node selection is manual.
- **Batched allocation**: allocation requests are accumulated for ~500 ms and committed together
  rather than one round trip to the API server per allocation.

Agones is Kubernetes-native, which makes it far too heavy for Korpis' target operator. The *model*
is the part worth taking.

### Nomad: the reference design for a pluggable runtime

Nomad's task driver plugin interface is the closest existing thing to Rule K-9 and should be
studied directly rather than reinvented: **[V]**

- Drivers declare **capabilities** (`SendSignals`, `Exec`, …) so the scheduler knows what a driver
  can do instead of assuming.
- `ConfigSchema` lets a driver publish its own configuration schema; `Init` runs setup after
  `SetConfig`; `ExecTask` runs commands inside the task's execution context.
- Shipped drivers span the full isolation range: `docker` (namespaces + cgroups), `exec` (cgroups +
  chroot), `exec2` (**Landlock LSM + cgroups v2 + `unshare`**, no container runtime at all), and
  `qemu` (full virtualization, resources bounded by the hypervisor rather than the host kernel).

`exec2` is especially relevant: it demonstrates strong isolation on modern kernels without a
container runtime dependency, which is exactly what a `native` driver for Korpis needs.

### Billing: permanently external, permanently shallow

WHMCS, Blesta, Paymenter, WemX, BillingServ, PteroBill. Paymenter has native Pterodactyl
provisioning but limited integration breadth and shallower automation than WHMCS; its cPanel
extension, for comparison, covers the provisioning lifecycle but has no package sync, no SSO, and
no in-panel management. **[R]**

The pattern across the whole category: billing systems can create and delete, and can do almost
nothing else, because the panel exposes provisioning but not identity, metering, or lifecycle.

> **Rule K-12.** Korpis does not implement billing. Korpis implements what billing needs and never
> gets: organizations and teams as first-class objects, plans and quotas as enforced policy, usage
> metering as an event stream (CPU-seconds, GB-hours, egress GB, IOPS), a typed event stream for
> every state change, and single sign-on. A billing integration should be a few hundred lines and
> require no fork.

---

## 4. Consolidated rules

| # | Rule | Comes from |
|---|---|---|
| K-1 | Tenant filesystem access is confined by the kernel, never by path-string validation | 6 of 12 Wings advisories |
| K-2 | Tenant templates evaluate in a sandbox with an allow-listed variable set | GHSA-pfvc-3p5h-x7h6 |
| K-3 | Every displayed limit is kernel-enforced and metered, or it is not displayed | #4554, #2798, #1871, #3352 |
| K-4 | Migration is a resumable, verifiable, reversible job with explicit phases | #4505, #5429, #3332, #3015 |
| K-5 | Backup is a consequence of the storage design | #5493, #2599, #3846, #4411 |
| K-6 | Extensions are out-of-process, versioned, sandboxed; core uses the same mechanism | Blueprint's `sed` model |
| K-7 | Control plane and data plane are separated by a published, versioned protocol | #141, open since 2016 |
| K-8 | License, versioning, and multi-maintainer governance are day-one decisions | Pelican fork, MIT→AGPL split |
| K-9 | The runtime is an interface; it must express Windows on day one | AMP's uncontested market |
| K-10 | Isolation strength is a per-workload operator choice | Pterodactyl vs PufferPanel |
| K-11 | Workload shape is a field, not an assumption | Coolify vs Pterodactyl |
| K-12 | Korpis implements what billing needs, not billing | WHMCS/Paymenter integration depth |

---

## 5. Sources

Primary:
- [Pterodactyl Wings security advisories](https://github.com/pterodactyl/wings/security/advisories)
- [Escape to host from installation container
  (GHSA-p744-4q6p-hvc2)](https://github.com/pterodactyl/wings/security/advisories/GHSA-p744-4q6p-hvc2)
- [CVE-2023-32080: execution with unnecessary
  privileges](https://security.snyk.io/vuln/SNYK-GOLANG-GITHUBCOMPTERODACTYLWINGSSERVER-5529843)
- [CVE-2023-25152: symlink
  following](https://vulert.com/vuln-db/go-github-com-pterodactyl-wings-147358)
- [#4554: disk space attack, 1 TB+ file, closed as not
  planned](https://github.com/pterodactyl/panel/issues/4554)
- [#2638: servers not shut down at disk limit](https://github.com/pterodactyl/panel/issues/2638)
- [#2798: high iowait and memory usage during
  backups](https://github.com/pterodactyl/panel/issues/2798)
- [#1871: limit bandwidth per server](https://github.com/pterodactyl/panel/issues/1871)
- [#3352: network limit per server](https://github.com/pterodactyl/panel/issues/3352)
- [#4505: failed transfer causes server deadlock](https://github.com/pterodactyl/panel/issues/4505)
- [#5429: transfer does not move symlinks](https://github.com/pterodactyl/panel/issues/5429)
- [#3332: processFailedTransfer NULL argument](https://github.com/pterodactyl/panel/issues/3332)
- [#3015: transfers failing, walking error](https://github.com/pterodactyl/panel/issues/3015)
- [#2599: S3 backups cannot exceed 5 GB](https://github.com/pterodactyl/panel/issues/2599)
- [#3846: backups upload to S3 without local
  disk](https://github.com/pterodactyl/panel/issues/3846)
- [#5493: incremental backup support](https://github.com/orgs/pterodactyl/discussions/5493)
- [#4715: Borg backup feature request](https://github.com/orgs/pterodactyl/discussions/4715)
- [#141: third-party daemon support, open since
  2016](https://github.com/pterodactyl/panel/issues/141)
- [Blueprint: extension scripts documentation](https://blueprint.zip/docs/concepts/scripts)
- [Pelican: comparison with Pterodactyl](https://pelican.dev/docs/comparison/)
- [Agones: overview](https://agones.dev/site/docs/overview/), [fleet
  spec](https://agones.dev/site/docs/reference/fleet/), [allocation
  spec](https://agones.dev/site/docs/reference/gameserverallocation/)
- [Nomad: task driver plugins](https://developer.hashicorp.com/nomad/plugins/drivers), [exec2
  driver](https://developer.hashicorp.com/nomad/plugins/drivers/exec2), [qemu
  driver](https://developer.hashicorp.com/nomad/docs/deploy/task-driver/qemu)

Secondary:
- [TheGamePanel: a modern alternative to
  Pterodactyl](https://ollieread.com/articles/thegamepanel-a-modern-alternative-to-pterodactyl)
- [Pyrodactyl](https://github.com/pyrohost/pyrodactyl) · [Our journey and the future of
  Pyrodactyl](https://pyro.engineering/posts/our-journey-and-the-future-of-pyrodactyl/)
- [Pelican FAQ](https://pelican.dev/faq/)
- [Pterodactyl is back and keeps its MIT
  license](https://teramont.net/blog/pterodactyl-is-back-and-keeps-its-mit-open-source-license)
- [AMP vs Pterodactyl (CubeCoders)](https://cubecoders.com/compare/pterodactyl)
- [Wings daemon config disclosure,
  CVE-2026-52855](https://www.thehackerwire.com/wings-daemon-config-disclosure-cve-2026-52855/)
- [Coolify vs
  Dokploy](https://servercompass.app/blog/coolify-vs-dokploy-self-hosted-paas-comparison)
- [Paymenter vs WHMCS vs Blesta](https://builtbyotte.com/blog/paymenter-vs-whmcs-vs-blesta)
