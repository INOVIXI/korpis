# Contributing

There is no code yet. **The most valuable contribution right now is disagreement.**

A specification nobody has argued with has not been reviewed, and this one makes a number of
deliberate departures from how this class of software is normally built. Each of those departures is
a place it could be wrong.

## What helps most

**Find a hole in the reasoning.** If a document asserts something it hasn't earned, or leans on an
assumption it never states, that is worth an issue. Quote the section.

**Find a failure mode the walkthroughs missed.** [`23-walkthroughs.md`](./23-walkthroughs.md) traces
seven scenarios end to end specifically to find seams between documents. It found nine. There are
almost certainly more, and the eighth scenario nobody has traced is where they are.

**Correct the evidence.** [`research/evidence.md`](./research/evidence.md) makes factual claims about
deployed software. If one is wrong, out of date, or unfair, please say so — accuracy matters more
there than being right, and if you maintain one of the projects discussed, your correction takes
precedence over our reading.

**Bring operational experience.** If you run this kind of infrastructure at scale, the thing you know
that no design document can supply is which of these decisions will hurt at 3am. That is the review
this most needs.

**Take an open question.** About forty are recorded across the documents, each filed where it will be
settled. Several are genuine forks in the road rather than details, and are marked as such.

## What to expect

Every issue gets a response. §7 of [`19-governance.md`](./19-governance.md) commits to that, and to
the reason: an untriaged issue is a process failure, not a backlog item. "We're not going to do this,
because —" is a fine outcome. Silence is not.

Changes to the domain model, the protocol, the security boundary, the licence, or the governance
rules go through a written proposal and an explicit decision. Everything else runs on lazy consensus.

## Style

Documents are the specification, not a description of code written elsewhere. They are meant to be
read start to finish by someone who has not seen them before, so:

- State the decision, then the reasoning, then what it costs. Never only the first.
- Cite the evidence when a claim rests on one. "Because X happened, here, with this issue number" is
  worth more than a confident sentence.
- Say what a thing does *not* do. Most of the value in the security and operations documents is in
  the honest negatives.
- Record corrections in place rather than rewriting history. §2 of [`01-model.md`](./01-model.md) is
  the model for this.
- Keep lines under 100 characters, and prefer plain prose to bullet fragments.

## Legal

Contributions are accepted under the **Developer Certificate of Origin**. Sign off your commits:

```
git commit -s
```

There is no contributor licence agreement and no copyright assignment. This is deliberate — it keeps
copyright distributed, which means no single party can ever relicense the project, which is what
makes the promises in §2 of [`19-governance.md`](./19-governance.md) structural rather than a
statement of current intent.

Licensing is split by layer: protocol and SDK material is Apache-2.0, the platform is AGPL-3.0. The
document you are editing tells you which side of that line it describes; when in doubt, ask in the
issue first.
