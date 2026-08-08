# Identity, Tenancy, and Authority

**Status:** design **Date:** 2026-08-07 **Depends on:** [`01-model.md`](./01-model.md),
[`02-architecture.md`](./02-architecture.md) **Resolves:** open question 3 of `01-model.md`; 2 of
`02-architecture.md`; 5 of `05-scheduling.md` **Implements:** Bet 1, Bet 4, Principle P6, Rule K-14

---

## 1. The risk in this document

Every other decision in Korpis has a precedent somewhere. This one does not: no product in this
market has shipped capability-based authorization in a hosting panel. TCAdmin, Multicraft, Pelican,
Calagopus, AMP, Cloudron, all use roles, because roles are what everyone knows.

That is the risk and it is accepted, because roles cannot express the operation a chat-native
platform performs constantly: *let this specific person restart this one server for the next 24
hours, without creating an account.* Bet 1 requires it. Bet 4 requires the same machinery pointed
at tenancy.

The mitigation is not confidence. It is that §4 adds a familiar layer on top, templates that behave
like roles in the interface, so if the pure model proves unusable to real operators, the recovery
is a UI change and not a schema change.

---

## 2. Tenancy

```
Organization ──▶ Organization ──▶ Organization ──▶ …
      │                │
      └──▶ Project     └──▶ Project ──▶ Workload
```

Organizations nest without limit. Each holds a `QuotaSet` (§5.1 of `05-scheduling.md`) that can
never exceed its parent's unallocated remainder. Projects group workloads and are the usual unit of
delegation.

**There is no `Reseller` object, and there never will be.**

| Real role | What it is in Korpis |
|---|---|
| Hosting provider | a root organization |
| Their customer | a child organization with an allocated `QuotaSet` |
| That customer's customers | grandchildren |
| A community's admin team | a project-scoped grant |
| A single game server owner | a workload-scoped grant |

Nothing in the code distinguishes these levels. TCAdmin sells reseller support as a premium feature
because its tenancy was not built for it; here it is what a correct tenancy model produces by
default (Bet 4, Rule K-14).

---

## 3. Grants

### 3.1 Grants are purely additive. There are no deny rules.

A request is `(subject, action, resource)`. It is permitted if **any** valid grant chain permits
it.

No denies, no precedence, no ordering, no priority numbers.

This is a deliberate and slightly uncomfortable stance, because "allow everything except delete"
becomes "do not grant delete" rather than "grant everything, deny delete". The reason is that deny
rules are where authorization systems become unpredictable: once denies exist, the answer to "can I
do this" depends on evaluation order, specificity rules, and inheritance precedence, and the honest
answer to "why was I denied" becomes a debugging session. Every large IAM system has this problem
and none has solved it.

Purely additive means the evaluation is a search for one permitting chain, and the explanation for
a denial is always the same sentence: *no grant you hold permits this.*

Narrowing an existing broad grant means revoking it and issuing narrower ones. That is more work,
and it is visible, auditable work rather than an invisible rule interaction.

### 3.2 Attenuation, precisely

A child grant is valid only if **all** hold against its parent:

| Dimension | Rule |
|---|---|
| actions | `child.actions ⊆ parent.actions` |
| scope | `child.scope ⊑ parent.scope`, contained, per §3.3 |
| expiry | `child.expires_at ≤ parent.expires_at` |
| network | `child.source_cidr ⊆ parent.source_cidr` |
| MFA | `child.requires_mfa ≥ parent.requires_mfa` |
| uses | `child.max_uses ≤ parent.remaining_uses` |
| approval | `child.requires_approval_by` at least as strict |

Checked at issue **and re-checked at every use**, because a parent may have been revoked in
between. A grant is not a certificate that keeps working once printed; it is a claim that is
re-derived every time it is presented.

### 3.3 Scope containment

```
Volume | Endpoint | Stream  ⊑  their Workload
Workload                    ⊑  its Project
Project                     ⊑  its Organization
Organization                ⊑  any ancestor Organization
Selector(P, expr)           ⊑  P            for any expression
```

The last line resolves open question 3 of `01-model.md`.

### 3.4 Selector scopes, and the trap in them

> Resolves open question 3 of `01-model.md`.

Granting over a label selector ("restart anything tagged `tier=staging` in this project") is far
more useful than enumerating IDs. It is also dangerous in a way that is easy to miss: **the set
changes after the grant is issued.** Label a production workload `tier=staging` by accident and the
grant now covers it.

Two rules make selectors safe, and both are necessary:

**Rule 1: a selector is always bounded by an explicit container.** `Selector(project P, expr)`,
never a bare expression. A selector-scoped grant cannot escape the tenancy subtree it was issued
within, whatever anyone labels anything.

**Rule 2: a selector-scoped grant can never confer `label.write` on what it selects.**

Rule 2 is the important one. Changing labels is what widens a selector grant's reach, so if a
selector grant could confer label-writing, it could be used to widen itself, an authorization
system that can bootstrap its own privileges. Under Rule 2, `label.write` is only ever conferred by
a grant scoped explicitly, by ID or by container. Whoever can move a workload into a selector's
reach is necessarily someone who already had authority over that workload directly.

A selector grant can therefore only ever be widened by someone who did not need it widened.

### 3.5 Lifecycle and revocation

```
issued ──use──▶ issued
   ├── expires_at reached ──▶ expired
   ├── max_uses reached ────▶ exhausted
   ├── revoked ─────────────▶ revoked  ⟶ cascades to entire subtree
   ├── parent revoked ──────▶ revoked
   └── root binding unverified ▶ suspended ⟶ cascades, reversible (§5.1)
```

Revocation cascades transitively and takes effect immediately in the control plane. The one place
it is not instantaneous is a cached capability token at the edge, bounded and quantified in §6.

**Revocation is derived, not written to the subtree.**

> External review, finding R4.

"Cascades transitively and immediately" describes two different implementations and the document
implied both. Writing `revoked_at` onto every descendant makes a deep subtree either one enormous
transaction or an asynchronous job, and if it is asynchronous then "immediately" is false.

So the cascade is **derived**: `revoked_at` is written on the grant that was revoked and nowhere
else, and validity is computed by walking the chain, which authorization already does at use time
("re-checked at use time, because a parent may have been revoked in between", §3 of `01-model.md`).
Revocation is therefore genuinely instantaneous and costs one row.

The audit consequence has to be stated, because it is the price. "When did this grant stop being
valid" is a computed answer rather than a stored one, so the `Effect` log records the **revocation
that caused it**, and a query about a descendant resolves through the chain to that event. An
`Effect` still names the grant it acted under (§3.3 of `01-model.md`); what it does not do is claim
that grant was independently revoked when its parent was.

`suspended` exists because one root of a grant tree is not a row in this database. A grant rooted
in an `ExternalIdentity` depends on an assertion Korpis re-checks rather than owns, and §5.1 gives
that re-check an interval and a cascade. Suspension is reversible and revocation is not, which is
the right asymmetry: an identity provider having a bad afternoon should not destroy delegations
that a human will have to rebuild by hand.

---

## 4. Templates: roles in the interface, not in the model

```
GrantTemplate
  name           "Moderator" · "Billing integration" · "Read-only auditor" · "Support, 4 hours"
  actions        []Action
  scope_pattern  ScopePattern
  conditions     Conditions      defaults, overridable at issue
```

Applying a template **issues ordinary grants** and then plays no further part. Three properties
follow, and the second is the one RBAC cannot offer:

1. Issued grants are ordinary grants: they attenuate, expire, revoke, and audit identically.
2. **They carry no back-reference to the template.** Editing a template later does not silently
   widen authority already handed out. In every role-based system, editing a role changes what
   everyone holding it can do, retroactively and invisibly, a reliable source of production
   accidents.
3. Authorization code never sees a template. There is exactly one evaluation path (P6).

---

## 5. External identity

```
Subject = User | Token | ExternalIdentity | Workload | Extension
```

```
IdentityBinding
  provider      discord | oidc | saml
  selector      guild+role · guild+user · oidc issuer+claim
  subject       ExternalIdentity
  verified_at   Timestamp
  constraints   BindingConstraints
```

An `ExternalIdentity` is a first-class subject, so a grant can name *the `@moderator` role in
Discord guild 456* directly. Nobody needs a Korpis account. Losing the role removes the authority
with no deprovisioning step, because there was no provisioning step.

### 5.1 Losing a role removes authority only if something notices

> Finding 19 of `23-walkthroughs.md`.

"No deprovisioning step" is a feature and it is also the reason there is no event. Discord does not
tell Korpis that a role membership ended; the role simply stops appearing in the next signed
interaction, and if the person never interacts again, Korpis observes nothing at all. §3.5 makes
revocation cascade to an entire subtree, and that machinery runs on a revocation. Nothing was
revoked here.

So the promise holds for exactly one of the three things a role holder can leave behind:

| What they left | What removal of the role does today |
|---|---|
| a console session | dies within one token lifetime, 120s. **Correct** |
| a child grant they issued to a helper, 7 days | keeps working. **Wrong** |
| a share link, 24 hours | keeps working. **Wrong** |

An external binding is therefore **re-verified on a declared interval**, not only when its holder
happens to interact:

```
IdentityBinding
  …
  reverify_interval   Duration
  state               verified | unverified
  last_verified_at    Timestamp
```

A binding that fails re-verification enters `unverified`, which **suspends every grant rooted in it
and cascades exactly as revocation does**. Suspension rather than revocation, because an identity
provider being unreachable must not silently destroy a delegation tree: `unverified` is visible,
dated, and reversible, and it fails closed for authority while failing loudly for the operator.

**A structural ceiling, so that re-verification is not the only defence.**

> External review, finding R1, which reached the same defect from the algebra rather than from a
> trace and proposed one mechanism this section did not have.

Re-verification bounds how long a stale membership can authorize, and it depends on an interval
someone configures. A second bound that no configuration can widen is cheap here: **a grant whose
chain contains a set-subject anywhere carries a hard expiry ceiling**, independent of what its
issuer asked for. A moderator with a ninety-day role grant can mint a child that outlives their
membership by at most the ceiling, not by ninety days.

The two mechanisms fail differently, which is why both are worth having. Re-verification catches a
membership that ended while the delegation tree stays intact. The ceiling catches the case where
re-verification itself is misconfigured, unreachable, or set too long, and it does so without
asking anyone to have got a number right.

`reverify_interval` is the honest analogue of token lifetime. It is the bound on how long authority
outlives its source, it is stated rather than convenient, and it applies to OIDC and SAML equally,
because a group claim in a token that was issued an hour ago is exactly the same problem.

### 5.2 The Discord trust boundary, stated plainly

When a Discord interaction arrives, it is signed by Discord (Ed25519) and its payload includes the
member's roles in that guild. Korpis verifies the signature and takes the role list as fact.

**That means Korpis trusts Discord's assertion of who holds which role.** Anyone who obtains
*Manage Roles* in that guild can grant themselves whatever Korpis granted that role. This is not a
flaw to be engineered away; it is the necessary consequence of accepting an external identity
provider, and it is true of OIDC and SAML equally. What matters is that it is stated rather than
discovered.

Two consequences are built in:

**Default: grants to Discord-role subjects cannot include irreversible actions.** Deleting a
volume, destroying a workload, releasing an IP, and pruning backups are excluded unless an operator
deliberately overrides the default for that binding, having read what it means.

**`requires_mfa` cannot be satisfied by a Discord identity alone.** A grant demanding MFA needs a
Korpis-native authentication factor. Discord's own 2FA state is not visible to Korpis and would not
be trustworthy if it were.

The result is a usable split: routine operations (start, stop, restart, read console, view files)
flow from Discord roles with no friction, which is the whole point of Bet 1. Destructive operations
require a stronger identity or an approval from one (`requires_approval_by`), which is what makes
granting a chat role real authority defensible.

### 5.3 OIDC and SAML

Standard, and the same shape: an issuer plus a claim maps to an `ExternalIdentity`, grants are
issued to it, group claims work exactly like chat roles including the same trust boundary. SSO is a
paid-tier feature in the commercial products and absent from the open ones. Here it is core,
because it is the same mechanism an external chat identity already requires.

---

## 6. Capability tokens: the wire form of a grant

> Resolves open question 2 of `02-architecture.md`, and the emergency-console half of question 1.

The insight that makes this coherent: **a grant and a capability token are the same object in two
representations.** A grant is the stored form; a token is the signed, self-contained wire form.
Identical attenuation algebra, identical evaluation.

```
CapabilityToken
  grant_chain     the attenuation chain, each link signed
  actions, scope, conditions
  issued_at, expires_at
  epoch           the issuing control plane's key epoch
```

An agent holds the control plane's public key and verifies a token **offline**. This is what makes
interactive work possible without a round trip per keystroke:

```
client attaches to console
  → control plane evaluates the grant chain (online, authoritative)
  → issues a capability token: {console.write on workload W, expires in 120s}
  → agent verifies the signature offline
  → keystrokes flow directly
  → renewal every 120s requires the control plane
```

**Revocation latency equals the token lifetime.** That is the honest trade, and it is why the
lifetime is short and configurable rather than convenient. Revoking a grant stops the *next*
renewal, not the current window.

Two things fall out:

**Console survives brief control plane outages**, for exactly one token lifetime, then it stops.
Bounded, predictable, and stated.

**Break-glass access is the same mechanism.** An operator can pre-issue a long-lived token, held
offline, that agents accept when the control plane is unreachable (open question 1 of
`02-architecture.md`). It is a real capability with real risk, so: its use is recorded locally and
replayed to the effect log on reconnect, it is revocable by rotating the control plane key epoch,
and its existence is visible in the interface rather than being a secret an operator forgets they
created.

**A restored control plane rotates the key.**

> Finding 9 of `23-walkthroughs.md`.

Offline verification means a token's validity does not depend on the database that recorded it.
That is the property this whole section is built on, and it has a failure mode: restore an older
backup and a grant that was issued *and revoked* after it is gone from the database while its
token, signed by a key restored alongside, still verifies. Authority in circulation, no row
recording it, and an audit trail that disagrees with what is actually happening.

So key rotation is a mandatory step of control-plane restore, alongside the lease epoch fence (§7
of `03-state.md`). Every token issued after the backup stops verifying and every session
re-authenticates, because a control plane that cannot account for a grant must not honour it.

This is also why token lifetimes being short is a structural decision rather than a tuning knob:
the same number that bounds revocation latency bounds how long a restore's disruption lasts.

**Two keys, and restore rotates only one.**

> Finding 16 of `23-walkthroughs.md`.

"Rotate the signing key" is ambiguous while there are two keys in this design with two different
jobs, and the ambiguity decides whether disaster recovery ends with a working fleet:

| Key | Signs | Rotated on restore |
|---|---|---|
| the **grant signing key** | capability tokens, verified offline by agents | **yes**, mandatorily |
| the **node identity certificate** | the per-node key pinned at enrollment (§3 of `18-operations.md`) | **no** |

They are separate because node identity and delegated authority have different lifetimes and
different revocation stories, which is the same reasoning that separated every other authority in
this design. Rotating both would end a recovery with an operator re-enrolling every node by hand,
during the incident, which converts the outage the restore was fixing into a longer one.

### 6.1 The shareable link

Bet 1's sharpest edge, and the operation RBAC cannot express:

```
https://korpis.example/g/#<token>
```

The fragment **is** the capability token. Holding the URL is holding the authority.

```
Moderator holds: workload.restart on project Survival, expires in 90 days
        attenuates to
Link:            workload.restart on workload W, expires in 24h, max_uses 3
        sent in a Discord DM, no account, no invitation, no provisioning
```

It cannot outlive the moderator's own grant. Revoking theirs kills it. Every use is recorded in the
effect log with the full chain, so "who restarted it" resolves to *this link, attenuated from this
moderator, issued at this time*.

**It is a bearer token, and that is said out loud.** Anyone holding the URL holds the authority.
Mitigations, all of them partial:

- The token is in the **fragment**, never the path or query, fragments are not sent to servers and
  do not appear in access logs or `Referer` headers.
- Short expiry and use caps are defaults, not options. A use cap is redeemed online (§6.2), which
  is a real cost and is stated where the link is created rather than discovered during an outage.
- High-risk actions are excluded from link-form grants by the same default that governs chat roles
  (§5.2).

**Not bound to the IP of first use.**

> External review, finding R14.

An earlier draft offered this as a hardening option. It does not survive contact with the people
the feature exists for: a moderator opening a link on a phone is behind CGNAT or moving between
cell towers, and their address changes minutes after first use. An option that breaks the intended
user in ordinary conditions is not a security control, it is a support ticket that also teaches
people to turn security controls off.

Online redemption (§6.2) is the mechanism that actually bounds a leaked link, and it bounds it by a
number the issuer chose rather than by a network property nobody controls. `source_cidr` remains
available for the case it was designed for, a link issued to a known fixed network.

---

### 6.2 `max_uses` is redeemed online, and says so

> External review, finding R2. Recorded in §16 of `23-walkthroughs.md`.

An agent verifying a token offline can check two of a grant's three limiting conditions. A
signature is self-contained. An expiry is a number compared against a clock. **A use counter is
neither**: it is shared mutable state, and counting it needs either a central point of redemption
or coordination between every agent that might see the token. This design specified neither, which
made `max_uses` a limit the interface displayed and nothing enforced, in the document that tells
every other document not to do that (K-3, P4).

So the rule is stated rather than assumed:

| Condition | Checked where | Survives a control plane outage |
|---|---|---|
| signature and grant chain | agent, offline | **yes** |
| `expires_at`, `not_before` | agent, offline | **yes** |
| `source_cidr` | agent, offline | **yes** |
| `requires_mfa`, `requires_approval_by` | gateway, online | no |
| **`max_uses`** | **gateway, online, redeemed atomically** | **no** |

A token carrying `max_uses` is redeemed at the gateway, which decrements the counter in the same
transaction that authorizes the call. It is therefore **not** usable while the control plane is
unreachable, and a token without `max_uses` is. That is the trade and it is the right way round:
the feature that needs a counter is the one that gives up offline operation, rather than every
token giving up a guarantee to support one condition.

The cost is stated where a user meets it. A share link with a use limit is described in the
interface as requiring the control plane; the console token of §6, which carries no use limit,
keeps its documented survival window. Break-glass tokens (§6, below) never carry `max_uses` at all,
because a credential whose entire purpose is working during an outage cannot depend on a counter
that does not.

**Issuing is also a race, and it is the same race quota had.**

> External review, finding R3.

`child.max_uses ≤ parent.remaining_uses` (§3.2) reads `remaining_uses`, which is the one mutable
field on an otherwise immutable object. Two concurrent attenuations of a parent with three uses
left each observe three and each mint a child with three. This is Finding 1 of `23-walkthroughs.md`
exactly, on the authority side instead of the capacity side, and it takes the same answer:
**`remaining_uses` participates in a compare-and-swap on the parent grant row, in the transaction
that inserts the child.** A losing writer gets `CONFLICT` and re-attenuates against the new figure.

### 6.3 Revoking one thing without revoking everything

> External review, finding R13.

§6 says a break-glass token is "revocable by rotating the control plane key epoch". That is true
and it is a sledgehammer: rotating the key invalidates every token in the system, so killing one
pre-issued credential logs out every console session in the fleet.

Break-glass tokens are few and long-lived, which is exactly the shape a revocation list handles
well. Agents hold a **signed revocation list of token IDs**, distributed on the same channel as the
public key and cached with the same rules. It is checked offline, so it costs nothing that matters,
and it is bounded because only long-lived tokens are ever listed. Short-lived tokens continue to
rely on expiry, because a list entry that outlives its token is waste.

Key rotation remains the answer to *"the key may be compromised"* and to control-plane restore
(§6). It stops being the answer to *"this one credential should stop working"*.

---

## 7. What a provider can see

Bet 4 makes hosting providers parents of their customers' organizations, which raises a question
every panel avoids: **can my host read my files?**

The physical answer is always yes, they own the hardware. The design question is whether the
platform makes that invisible.

| Data | Parent organization's default access |
|---|---|
| Resource usage, quota consumption, metering | **yes**: they are paying for it |
| Workload existence, names, state, health | **yes**: they operate the capacity |
| Placement, node assignment | **yes** |
| Console content | **no**, requires an explicit grant |
| File contents | **no**, requires an explicit grant |
| Backup manifest, the file tree without contents | **no**: requires `backup.browse`, and browse does not imply read (Finding 13 of `23-walkthroughs.md`) |
| Backup contents | **no**, requires an explicit grant, and see §9.1 of `06-storage.md` on key custody |
| Environment variables and secrets | **no**, requires an explicit grant |

Operator access to tenant data is itself a grant. It can be issued (support cases require it) and
when it is used, **the effect is visible to the tenant whose data was accessed**, not only to the
operator's audit log.

This is a stance no competitor takes, and it costs nothing to implement given the model. It
converts "your host can read everything and you will never know" into "your host can read this, and
you will see when they do."

---

## 8. Explanations are filtered by grant

> Resolves open question 5 of `05-scheduling.md`.

The `Explanation` attached to a `Placement` (§2.3 of `05-scheduling.md`) contains node identities,
other nodes' capacity, and why candidates were rejected. Showing that to a customer leaks the
provider's infrastructure and, through utilization figures, information about other tenants.

**Explanations are stored complete and rendered per viewer.** The stored object is the full
derivation. What a subject sees is filtered to what their grants cover:

| Viewer | Sees |
|---|---|
| Operator (root org) | everything, every candidate, every score, every node |
| Tenant | "placed in region EU on a node offering the `microvm` tier; 3 candidates considered" |
| Workload-scoped grant | placement region and tier only |

Same rule applies wherever an object aggregates across a tenancy boundary, node status, cluster
capacity, scheduler recommendations. Filter on read; never store two versions of the truth.

---

## 9. Authentication

| Factor | Notes |
|---|---|
| Passkeys / WebAuthn | **the default and the recommendation**, phishing-resistant, and the reason it is first here |
| TOTP | supported, second class |
| Password | supported, always with a second factor available |
| OIDC / SAML | §5.2 |
| Discord | §5.1, with its stated limits |
| Node enrolment token | single-use, short-lived, itself an attenuated grant (§7 of `02-architecture.md`) |

`requires_mfa` on a grant is satisfied by a passkey or TOTP, never by an external provider's
assertion.

---

## 10. Open questions

1. **Token format.** Biscuit gives offline attenuation and offline verification with a public key,
   in a specification designed for exactly this. Macaroons are simpler and older but their
   third-party caveats are awkward. A custom format is more work and fewer surprises. This choice
   determines whether a *client* can attenuate a token without contacting the control plane, which
   would make §6.1 links issuable entirely offline. → here
2. **Are teams a subject, or just nested organizations?** A "team" that spans projects but is not
   an organization does not exist in the model. Nested organizations cover it awkwardly, a team is
   not a tenancy boundary and does not want a quota. Adding a `Group` subject is simple and adds a
   second grouping concept alongside labels. → here
3. **Organization transfer.** Selling or handing over an organization moves it under a new parent.
   Every grant issued by the old parent must be re-validated against the new one, and some will
   fail attenuation. Whether that revokes them, suspends them, or blocks the transfer is
   unresolved, and it is exactly the operation a hosting business performs when a reseller changes
   hands. → here
4. **Quota consumption by revoked grants.** A revoked grant's workloads keep running and keep
   consuming the tenant's quota. Correct, revoking someone's access should not delete their work,
   but it means quota can be consumed by resources nobody can currently manage. Who reclaims it,
   and after how long? → `05-scheduling.md`
5. **Effect-log visibility across the tenancy tree.** A parent can see that a child's workload was
   restarted. Can it see which of the child's users did it? Operationally useful for support,
   uncomfortable for privacy, and the answer probably differs between a hosting provider and a
   community. → here
