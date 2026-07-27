# IDEAS.md — long-horizon directions

Speculative directions well beyond current scope. They live here so they are not
lost, and so they are not implemented prematurely.

**Do not implement anything in this file without explicit instruction.**

For planned, scoped work see [`docs/roadmap.md`](docs/roadmap.md). Near-term backlog
items (nanobody sources, humanness scoring, epitope binning, affinity estimation,
payload registry, multi-target logic) live there, not here.

Effort ratings are for a small academic team:
🟢 straightforward · 🟡 substantial · 🔴 multi-year

---

## 1. Protein language models for novel binder discovery

🟡 **Substantial**

### The gap

synbind finds binders that already exist in the literature. It cannot discover
binders that have never been experimentally characterised. For targets with no
published antibody, it has nothing to offer.

### The approach

Protein language models embed sequences in a space organised by learned function
rather than by sequence identity. Searching that space with a target sequence
retrieves proteins that are *functionally* related — including receptor-like
proteins that may be uncharacterised natural binding partners, and including
proteins from organisms nobody has studied for this purpose.

Two query modes:

- **By target** — find proteins functionally related to the target ligand; some
  will be plausible natural binding partners not yet curated in any interaction
  database
- **By known binder** — find functional analogs of an existing binder that may be
  smaller, better expressed, or better folded

Some embedding services expose interpretable features explaining *why* a protein
was retrieved (e.g. "immunoglobulin-like binding domain"). Where available, these
should be stored and surfaced — an unexplained retrieval is not actionable for a
wet-lab user.

### What it unlocks

- Binders for targets with no published antibodies
- Smaller or better-expressed alternatives to known binders
- Natural binding partners from understudied organisms

### Why it is hard

Everything retrieved this way is a **hypothesis with no experimental support**. It
would enter the pipeline at the lowest possible confidence tier, and the interface
would have to make that unmistakable. The provenance requirement (ADR-011) exists
precisely to prevent uncitable sequences appearing in recommendations — this
feature would need an explicit, clearly-labelled exception rather than a loophole.

---

## 2. De novo binder design

🔴 **Separate project**

Rather than searching for existing binders, generate new proteins computationally
designed to bind a chosen epitope.

The established stack is a backbone generator conditioned on a binding site, a
sequence designer for those backbones, and a structure predictor for validation.

Workflow:

1. Define the target epitope (user input or predicted)
2. Generate many backbone candidates
3. Design sequences for each
4. Validate through the existing pipeline

Output would be a binder with no natural analog. High risk — most designs fail
experimentally — but valuable where no natural binder exists or where existing ones
are encumbered.

This is a project in its own right, not a feature. It also inverts the tool's
current premise: synbind is built around published, citable binders, and de novo
designs are neither.

---

## 3. Molecular dynamics for complex stability

🔴 **Requires specialised expertise**

Structure prediction gives a static snapshot. MD would indicate whether a complex
remains stable over time, and — for force-dependent receptors — whether the binder
stays engaged under the mechanical load of cell–cell contact.

Relevant because sustained engagement is a precondition for cleavage in
force-dependent scaffolds, and nothing in the current or planned pipeline addresses
it.

Requires significant compute and expertise that a small team is unlikely to have
in-house. Flagged for a future collaboration rather than an internal deliverable.

---

## 4. Learning from experimental outcomes

🟡 **Substantial — and depends on data that does not exist yet**

Every construct the lab builds and tests produces a data point: this binder, this
scaffold, this target — did it work?

Captured systematically, that becomes a feedback loop. Recommendations could be
weighted by what actually succeeded in this lab, on this cell type, rather than by
sequence-level heuristics alone. Over time it would also reveal which validation
rules are predictive and which are noise.

The prerequisite is boring and unavoidable: **a structured record of experimental
outcomes**. Without it there is nothing to learn from. If this direction is ever
wanted, start collecting the data long before building anything that consumes it.

This is probably the highest-value long-horizon item, precisely because no external
tool can do it — the data would be proprietary to the lab.
