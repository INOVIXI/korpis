# Networking: Endpoints, Exposure, Policy, and Discovery

**Status:** design **Date:** 2026-08-07 **Depends on:** [`01-model.md`](./01-model.md),
[`05-scheduling.md`](./05-scheduling.md) **Resolves:** open question 8 of `01-model.md`
**Implements:** Rule K-3 (bandwidth, egress), Rule K-11

---

## 1. The unsolved problem

Pelican fixed nearly every surface complaint about Pterodactyl (plugins, webhooks, admin roles,
OAuth, multi-database, a rewritten UI) and **allocation system rework is still on its *planned*
list.** That single line, from Pelican's own published roadmap, is the most informative fact in
this market.

The reason is structural. Pterodactyl's `Allocation` is a bare `(ip, port)` row belonging to a
node. Everything hangs off it: the server's address, the firewall, what the customer sees, what the
billing system provisions. Replacing it means replacing the schema, so nobody who inherited it can.

What the whole category lacks as a result: ingress, automatic TLS, egress control, traffic
accounting, bandwidth limits (#1871, #3352), usage visibility (#3780), and service discovery.
Coolify and Cloudron have ingress and TLS but only for one workload shape.

And one consequence nobody names:

> **`(ip, port)` on a node ties a customer's address to a machine.** Migrating a server changes its
> address. Every player's saved entry breaks. Every DNS record, firewall rule, and billing record
> that referenced it is wrong. This is why migration is operationally dreaded even where it works,
> not because moving bytes is hard, but because the address moves with them.

Decoupling the address from the placement is the central design goal of this document. Everything
else follows from it.

---

## 2. The endpoint

```
Endpoint
  id, workload, name          "game", "rcon", "query", "http"
  protocol                    tcp | udp | http | https | quic
  container_port              int
  exposure                    Exposure
  address                     Address?        assigned; shape depends on exposure
  domains                     []Domain
  tls                         TLSSpec?
  policy                      NetworkPolicy
  accounting                  bool
  shaping                     ShapingSpec?
```

One object covers a Minecraft UDP port, an RCON TCP port, and an HTTPS ingress with a certificate.
They differ in field values, not in kind (P7).

---

## 3. Exposure modes

This is where the address is either tied to a machine or freed from it.

| Mode | Address | Survives migration | Costs | Right for |
|---|---|---|---|---|
| `node_port` | node's IP + allocated port | **no** | nothing | hobby, single node, dev |
| `stable` | an edge address, forwarded to the current placement | **yes** | an edge tier | game servers in production |
| `ingress` | name-based, shared address | **yes** | a proxy | HTTP/HTTPS, TLS services |
| `dedicated_ip` | an IP belonging to the workload | **yes**, if the IP is routable to any node | an IP, plus BGP or a provider API | games needing a whole IP, IP reputation |
| `overlay_only` | reachable by other workloads only | yes | an overlay | databases, internal services |
| `none` | none | none | nothing | bots, batch jobs |

`node_port` is what Pterodactyl offers, and it is kept because for a single-node install it is
correct and free. It is not the default, and the UI states its consequence, the address changes if
this workload ever moves.

### 3.1 `stable`: the edge

An **edge** owns stable public addresses and forwards to wherever a workload currently is.

```
                      stable address
   player ──────▶ 203.0.113.10:25565
                        │
                        │  kernel-level forwarding, target = current placement
                        ▼
                   node 7 ──migration──▶ node 12
                        │                    │
                        └──── target updated at cutover ────┘

   player's address never changes
```

The forwarding target is updated at migration cutover (§8 of `05-scheduling.md`), in the same step
as the lease epoch change. The client sees a connection reset, reconnects to the same address, and
lands on the new node. **A game server can move between machines without a single player editing
their server list.**

This is the thing that makes migration ordinary rather than an event, and it is why
`05-scheduling.md` could treat draining a node as routine.

**The default data path adds no hop.**

> External review, finding R18. Recorded in §16 of `23-walkthroughs.md`.

This market's users feel five milliseconds, so an exposure mode that puts a box in front of every
UDP packet would be the wrong default no matter what it buys. It does not:

- **An edge is co-resident by default.** For a single-machine install and for any node that is its
  own edge, forwarding is a kernel path on the same machine (nftables DNAT, or XDP/eBPF where the
  volume justifies it), which is a table lookup rather than a hop. §4 already said this; what it
  did not say is that it is the **default** rather than an option.
- **A remote edge is an operator's explicit choice**, taken to get an address that outlives the
  machine, and its cost is one hop, which is stated in the interface at the point the choice is
  made rather than inferred later from latency complaints.
- **Migration does not need a permanent hop.** The window where an address must reach two places is
  the cutover, so a co-resident edge forwards to the destination for the drain interval and stops.
  Paying a hop permanently to make a rare event cheaper is the wrong trade for this workload.

`node_port` remains available for the operator who wants the address on the node and accepts that
migration changes it, which is Pterodactyl's behaviour offered deliberately rather than inherited.

**An edge holds a lease, because everything else that forwards traffic does.**

> Finding 21 of `23-walkthroughs.md`.

Every other authority in this design is fenced. Agents hold leases with epochs, the control plane
fences its own restore, migration cuts over at one instant under a lease. The edge forwards for
hundreds of workloads and, as first designed, held nothing, which produced the one failure the
whole system is built to make impossible: a forwarding plane that has stopped passing traffic on a
host that is up, with an agent that is healthy, every node holding a valid lease and reporting
`running`, and a control plane whose observations are all correct while the service is down.

So:

- **An edge holds a lease with an epoch**, like any other data-plane component. Failover advances
  the epoch, which is what lets a replacement know the previous edge is no longer authoritative for
  an address, rather than inferring it.
- **Liveness is measured through the forwarding path, not from the process that owns it.** A
  synthetic connection through the same path a player takes is the only measurement that
  distinguishes "the edge process is running" from "the edge is forwarding". This is K-3 applied to
  a component that had escaped it: what is displayed is what was observed, and what was being
  observed was the wrong thing.
- **The detection interval is declared.** An operator choosing `stable`, and with it the
  concentrated failure mode §11.2 admits to, is entitled to a number for how long the concentration
  lasts before it is noticed. §5 of `18-operations.md` carries the failover mechanisms; this is the
  detection that has to precede any of them.

Dependents of a failed edge become `unsatisfiable` with the edge named, the same treatment §7 of
`05-scheduling.md` gives every other unsatisfiable condition, which is also what makes a hibernated
workload behind a dead edge visible: its wake trigger lives in the edge, so an edge that fails
silently would leave it in a state that is accurate and unreachable at once.

**When there is nothing to forward to.**

> Finding 5 of `23-walkthroughs.md`.

The address outlives the placement, which is the entire point, so the edge routinely holds an
address whose workload is stopped, hibernated, unschedulable, or on a node that has died. What it
does then is visible to every person connecting, and the three options are not equivalent:

| | What the person experiences |
|---|---|
| black hole | thirty seconds of nothing, then a timeout they blame on their own connection |
| refusal | immediate, honest, generic |
| protocol-aware status | immediate and *accurate*, only where a router understands the protocol |

**The default is immediate refusal. Never a black hole.** A timeout is P4's "unknown" rendered as a
stall, and it is the one failure mode that reliably produces a support ticket blaming the wrong
thing.

Where a protocol router is registered (§3.3), it answers with the real state (`hibernated`,
`starting`, `unreachable`, `data unavailable`) which is the same affordance §6 of
`22-first-party.md` uses to hold a player's client while a hibernated server boots. An unreachable
workload is that problem with a different message, not a different mechanism.

An edge is a role a node holds, not separate hardware, a small deployment can run the edge on the
same machine as its workloads. Forwarding is kernel-level (nftables DNAT, or XDP/eBPF where the
throughput justifies it) so it costs a NAT table entry rather than a userspace proxy hop, which
matters for latency-sensitive game traffic.

### 3.2 `ingress`: name-based routing

For HTTP, HTTPS, and TLS-with-SNI, routing is by name and one address serves everything:

- HTTP `Host` header, HTTPS/TLS SNI
- Automatic certificates via ACME (HTTP-01, TLS-ALPN-01, or DNS-01 through a DNS provider
  interface)
- Certificate lifecycle (issue, renew, revoke) owned by the control plane, distributed to edges
- HTTP/2, HTTP/3, WebSocket, gRPC pass through

This is the part Coolify and Cloudron already do well and Pterodactyl has never had. It is
table-stakes for the `service` shape and is why Pterodactyl cannot host a web app credibly.

**UDP cannot use SNI.** There is no name in a UDP packet, so most game traffic cannot be
name-routed by anything generic. That is why `stable` exists as a distinct mode: the address is the
identity, and the edge forwards it.

### 3.3 Protocol-aware routers are extensions

Some protocols carry a hostname in their handshake. Minecraft Java sends the server address it was
asked for, and Minecraft also supports DNS `SRV`, so `play.example.com` can point at any host and
port. A router that reads that handshake could name-route many Minecraft servers behind one address
and one port.

That router is an **extension**, not core. `00-overview.md` §2 says no game appears in the core,
and a Minecraft handshake parser in the ingress path would be exactly that. Core provides the
forwarding plane and a hook; a protocol-aware router registers as a handler for a port. The
capability exists; the game does not enter the core.

---

## 4. Addressing

**IPv6 is the default. IPv4 is the scarce resource.**

Every workload gets a routable IPv6 address where the node has IPv6. IPv4 is allocated
deliberately: shared by port under `node_port` or `stable`, shared by name under `ingress`, or
dedicated when someone pays for it.

This inverts what every panel in this market does, and it is worth doing because IPv4 is the single
largest per-workload cost a hosting operator carries. An operator who can serve IPv6-capable
clients natively and reserve IPv4 for those who need it has a materially cheaper cost structure.
Pelican added IPv6 allocations; treating IPv6 as the default rather than an option is the next
step.

```
AddressPool
  id, kind          ipv4 | ipv6
  cidr, scope       node | edge | cluster
  routing           static | bgp | provider_api
  reserved          []Address
```

`routing` determines whether a `dedicated_ip` can move between nodes. Static routing means the IP
belongs to one node and a workload holding it cannot migrate. BGP or a provider API means the IP
can follow the workload, which is what makes `dedicated_ip` compatible with migration rather than
an anchor.

---

## 5. Service discovery

> Resolves open question 8 of `01-model.md`.

A web service must reach its database without knowing where it is placed or hard-coding an IP that
changes on migration.

The usual answer is cluster DNS. Korpis' primary mechanism is **explicit dependency declaration**,
with DNS as an optional convenience:

```
Intent.dependencies[]
  name       "db"
  target     WorkloadRef | EndpointRef      by ID, or by selector within the project
  inject     env | file | both
  required   bool                            block start until healthy
```

Korpis resolves each dependency at start and injects the current address:

```
KORPIS_DEP_DB_HOST, KORPIS_DEP_DB_PORT       (env)
/korpis/deps/db.json                          (file, for workloads that reload)
```

Explicit declaration is better than implicit DNS for four reasons, and they are the reasons this
model was chosen over the familiar one:

1. **The dependency graph is visible**, so the scheduler can co-locate a service with its database
   (`05-scheduling.md` affinity) instead of discovering the coupling from latency complaints.
2. **Start order is derivable.** `required: true` means the dependency must be healthy first, no
   crash-loop-until-the-database-appears.
3. **A `Plan` can show the blast radius.** "Migrating this database will reconfigure three
   dependent workloads, two of which restart" appears in the plan *before* approval (P5). Implicit
   DNS makes that impossible to compute, because nothing records who depends on whom.
4. **It works across every tier.** Environment injection reaches a container, a microVM, and a VM
   through cloud-init identically. Cluster DNS assumes a resolver the workload cooperates with,
   which a VM guest may not.

The cost is that an address change requires a restart or a reload. Mitigated by the file form for
workloads that watch for changes, and by DNS where the class supports it.

---

## 6. Policy

No panel in this market has egress control, and it is the gap that hurts operators most, because a
compromised game server is a launchpad.

```
NetworkPolicy
  ingress   []Rule       who may connect in
  egress    []Rule       where it may connect out
  conntrack ConntrackLimits
```

**Defaults, applied to every workload unless deliberately changed:**

| Destination | Default | Why |
|---|---|---|
| Other tenants' workloads | **deny** | lateral movement between customers |
| Node management interfaces, SSH, the agent socket | **deny** | privilege escalation path |
| The Korpis control plane | **deny** | a workload has no business reaching it |
| Link-local metadata (`169.254.169.254`, `fd00:ec2::254`) | **deny** | classic cloud-credential theft |
| RFC1918 / ULA generally | **deny** | the operator's own infrastructure |
| Public internet | **allow**, rate-limited | mod downloads, auth servers, updates |
| SMTP (25, 465, 587) | **deny** by default | spam is the fastest way to lose an IP range's reputation |

The metadata endpoint deny is small and specific and closes a real, repeatedly-exploited hole: a
workload that can reach `169.254.169.254` on a cloud node can often read the instance's IAM
credentials and take over the operator's entire cloud account.

**Connection tracking limits** cap concurrent connections, new connections per second, and
half-open connections per workload. Without them one workload can exhaust the node's conntrack
table and take every other workload offline, a resource exhaustion path with no visible cause, and
one that appears in practice long before anyone thinks to look for it.

---

## 7. Accounting and shaping

Closes #3780 (no usage visibility), #1871 and #3352 (no bandwidth limit).

**Accounting** is per-endpoint byte and packet counters maintained in the kernel (nftables counters
or eBPF maps), sampled into the metering stream (§5 of `03-state.md`). It feeds the `egress`
dimension of `Quota` and is never reconstructed from averages, because it has financial
consequences (Rule K-12).

Counting in the kernel rather than in a proxy matters: it captures traffic the application never
sees or deliberately hides, and it cannot be evaded by a compromised workload.

**Shaping** uses `ResourceSpec.bandwidth` with the same reservation/limit split as everything else
(§5.2 of `05-scheduling.md`):

- reservation → guaranteed bandwidth, subtracted from the node's link capacity, never
  oversubscribed
- limit → ceiling
- the gap → burst, contended fairly under load

Implemented with HTB or an eBPF-based shaper, per direction. A workload sold "100 Mbit guaranteed,
1 Gbit burst" gets exactly that, and Rule K-3 means Korpis does not display the number unless the
kernel enforces it.

---

## 8. Abuse and attack

`00-overview.md` §2 says Korpis is not a DDoS mitigation provider. It is, however, the component
that knows which workload is being attacked and which one is attacking, so its scope is:

| Korpis does | Korpis does not |
|---|---|
| per-source connection and packet rate limits at the edge | absorb volumetric attacks |
| SYN cookies, conntrack limits, per-endpoint thresholds | scrub traffic |
| surface attack signals as metrics and alerts | operate a global anycast network |
| null-route an endpoint on operator command or policy | decide unilaterally to drop a customer's traffic |
| provide an interface for an upstream scrubbing provider | become one |
| detect and rate-limit *outbound* abuse from a workload | none |

Outbound detection is the half operators forget. A compromised game server used as a DDoS reflector
or a spam relay costs the operator their IP reputation, and the operator usually finds out from an
abuse complaint. Egress rate limits plus outbound accounting make it visible within minutes.

---

## 9. The overlay

`overlay_only` and cross-node dependencies need node-to-node connectivity.

**WireGuard mesh**: kernel-level, encrypted, low overhead, and it tolerates nodes on ordinary
internet connections, which matters because §3 of `02-architecture.md` deliberately allows nodes
behind NAT.

**And that is exactly the limitation, stated plainly:** the agent dialling out means a node needs
no inbound port *from the control plane*. An overlay needs nodes to reach *each other*. A node
behind symmetric NAT can reach the control plane and cannot be reached by its peers.

Three positions, and Korpis takes the third:

1. NAT traversal: STUN/hole punching. Works often, fails unpredictably, and unpredictable
   networking is worse than absent networking.
2. Relay everything through the control plane: always works, makes the control plane a data-path
   bottleneck, and contradicts §4.6 of `02-architecture.md`, where the data plane survives control
   plane loss.
3. **The overlay is optional and its requirement is explicit.** Nodes that can reach each other
   join the mesh. Nodes that cannot are marked `overlay: unavailable`, and the scheduler will not
   place a workload with cross-node dependencies or `overlay_only` endpoints on them (§2.1 of
   `05-scheduling.md`).

A hobbyist's NAT-bound node runs game servers perfectly and cannot host half of a two-tier
application. That is a true statement about their network, and Korpis reports it as a constraint at
scheduling time rather than as a mysterious connection failure at runtime.

---

## 10. What changes at migration cutover

Collected here because it is the sequence that decides whether §1's central goal actually holds:

| Exposure | At cutover |
|---|---|
| `node_port` | **the address changes.** Clients must be told. This is why it is not the default |
| `stable` | edge forwarding target is updated; the address does not change |
| `ingress` | proxy upstream is updated; the name and certificate do not change |
| `dedicated_ip` | with BGP or a provider API the IP is readvertised from the new node; with static routing the workload cannot migrate at all |
| `overlay_only` | overlay routes update; dependents are re-injected |

Every one of these is a step inside the `cutover` phase, ordered after `verify` and atomic with the
lease epoch change. There is no window in which traffic is forwarded to a node that no longer holds
the lease.

---

## 11. Open questions

1. **Edge forwarding implementation.** nftables DNAT is simple, well-understood, and adds a
   conntrack entry per flow. XDP/eBPF is faster and bypasses conntrack but is harder to write and
   debug, and its behaviour varies across kernels and NIC drivers. The crossover point is a
   measurement not yet taken. → here
2. **Edge high availability.** A single edge is a single point of failure for every `stable` and
   `ingress` endpoint behind it, reintroducing, at the network layer, exactly the fragility §4.6 of
   `02-architecture.md` removed at the control layer. VRRP, ECMP, and anycast all work and all have
   different operational demands. **Resolved in §5 of `18-operations.md`**, never a single machine;
   VRRP, ECMP, or anycast by deployment size, with state derivable from placements.
3. **IPv4 exhaustion strategies.** CGNAT, port-range sharing, and per-name sharing all extend a
   scarce IPv4 pool and all degrade some workloads. Which are offered, and how the degradation is
   surfaced to whoever is buying, is unresolved. → here
4. **Do dedicated IPs participate in quota?** An IP is a countable exclusive resource like a device
   (§6 of `05-scheduling.md`) and probably belongs in `QuotaSet`, but IPv4 and IPv6 are so
   different in scarcity that one dimension may be misleading. → `05-scheduling.md`
5. **Certificate scale.** ACME rate limits are per registered domain, and a provider issuing
   certificates for thousands of customer domains will hit them. Whether Korpis needs an internal
   CA for internal names, ACME only for customer-facing ones, or an ACME account pool is
   unresolved. → here
6. **East-west policy enforcement point. Resolved in §9.1 of `17-security.md`**: east-west is
   enforced at both ends, so the destination never trusts the source; north-south is uncontained on
   a compromised node and says so; and each node's expected egress profile is published so an
   operator can enforce at a layer the node does not control.
