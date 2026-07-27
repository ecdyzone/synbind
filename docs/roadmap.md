# Roadmap

Deliverables are **modular and independently pickup-able**. Each has its own scope,
its own definition of done, and no dependency on unfinished work elsewhere. Anyone
should be able to take on any unstarted deliverable without needing the original
author.

---

## v1 — Design and validate (in progress)

**Goal:** a working web application where a user picks a target and a scaffold, and
receives a recommended construct with a validation report.

Runs in seconds on a laptop. No GPU, no cluster, no queue.

### Scope

| In | Out |
|---|---|
| Curated binder registry | Live database search |
| Scaffold registry (synNotch, SNIPR) | Additional scaffold families |
| Construct assembly | DNA-level output |
| Sequence-level validation rules | Structure prediction |
| Recommendation + alternatives | Proteome-wide off-target screening |
| Three-layer web UI | CLI |
| Protein sequence export | Codon optimization, cloning |

### Initial targets

Four, spanning the design space deliberately:

| Target | Modality | Role |
|---|---|---|
| CD3ε | membrane-bound | candidate sensor |
| CD47 | membrane-bound | **benchmark** — well-characterised, both antibody and natural-receptor binder options |
| VEGF-A | soluble | candidate sensor |
| TGF-β1 | soluble | candidate sensor |

Two membrane-bound and two soluble, so the modality/scaffold compatibility logic is
exercised from the start.

### Build order

1. `models.py` — pydantic models. Nothing imports until stable.
2. `registry/` — load and validate `data/` reference files
3. `assembly.py` — scaffold-agnostic construct assembly
4. `validation/` — rules and report
5. `recommend.py` — ranking and rationale
6. `web/` — FastAPI + Jinja + HTMX, three layers
7. Dockerfile, CI, deployment

### Definition of done

- A user selects a target and scaffold and gets a recommendation with rationale
- Selecting a soluble ligand with a force-dependent scaffold produces a clear,
  blocking, explained failure with a suggested alternative
- Every displayed sequence links to its source
- Rejected candidates are visible with reasons
- Tests pass in CI; the app runs from a container

---

## v1.1 — Live binder discovery

**Replaces:** the curated seed registry as the sole source (it remains as the
high-confidence tier).

Query public structural and therapeutic antibody databases for binders against a
given target, extract sequences and chain roles, annotate species, and merge with
curated entries under a confidence ranking.

**Why separate:** this is where most implementation risk sits — inconsistent APIs,
antibody chain detection, chain-to-entity mapping, sequence extraction. Isolating it
keeps v1 shippable.

**Prerequisites:** v1 registry and models.

**Notable risk:** external APIs change without notice. Contract tests and a scheduled
canary are part of this deliverable, not an afterthought (ADR-012).

---

## v1.2 — Cross-species off-target screening

**The most scientifically novel deliverable.** See ADR-009.

Two directional screens against different reference proteomes:

- **trans** — homologs of the target ligand in the ligand's species
- **cis** — homologs of the target ligand in the **host cell's** species, which would
  cause the receptor to fire on its own host

Structure-based search (Foldseek) is preferred over sequence-based, since
cross-reactivity is structural. Predicted proteome-wide structure sets make this
tractable without running any prediction locally.

**Why it matters:** the cis screen catches a failure mode that produces constitutive
signalling and a dead experiment, and no general-purpose tool performs it.

**Prerequisites:** v1 models (which already reserve space for the results).

**Introduces:** the first conda-channel binary dependencies, and therefore the first
real use for containerised tool execution.

---

## v1.3 — Command-line interface

A thin client over the same core functions as the web application (ADR-006). For
batch use and scripting.

**Prerequisites:** stable core API.

**Deliberately late.** Building it earlier would risk core API decisions being shaped
by CLI convenience rather than by the domain.

---

## v2 — Structure enrichment

Attach structural evidence to constructs that already exist. Purely additive — no
existing behaviour changes.

Staged by cost:

1. **Antibody-specific structure prediction** — seconds per structure, sufficient for
   fold sanity checking on scFv and VHH constructs
2. **Complex prediction** — binder–ligand docking confidence, substantially more
   expensive

**Introduces:** a genuine multi-tool, per-candidate parallel workload — the first
point at which a workflow engine is justified (ADR-013). Nextflow, with profiles for
local, SLURM, and cloud batch execution.

**Prerequisites:** v1.2's containerised tool execution.

**Caution:** structural confidence scores are easy to over-trust. Any metric surfaced
in the UI must ship with an honest statement of what it does and does not indicate.

---

## Backlog

Not scheduled. Roughly ordered by expected value per unit effort.

| Item | Notes |
|---|---|
| **Payload/output module registry** | Beyond the default GAL4-UAS (ADR-010). Small and self-contained — a good first contribution |
| **Additional scaffold families** | MESA, GEMS. Each is a YAML file plus tests |
| **Nanobody sources** | Single-domain binders need no assembly; simpler than scFv |
| **Humanness scoring** | Framework identity to human germline; informational only |
| **Epitope binning** | Cluster binders by predicted interface; distinguishes redundant from complementary candidates |
| **Affinity estimation** | Note the window constraint — tighter is not better (see `docs/biology.md`) |
| **Multi-target logic** | AND-gate circuits requiring two receptors with mutually non-cross-reactive binders |
| **DNA output** | Codon optimization and cloning design. Currently out of scope (ADR-001); revisit only if users ask |

Longer-horizon ideas, including de novo binder design, are in [`IDEAS.md`](../IDEAS.md).

---

## Continuity

This project is intended to outlast any individual contributor. Practices that serve
that goal:

- **Decisions are recorded with rationale**, including rejected alternatives, in
  [`docs/decisions.md`](decisions.md). A future contributor should not have to
  re-derive why something is the way it is.
- **Reference data is versioned in git**, human-readable, and reviewable by
  biologists who do not read Python.
- **Deliverables are scoped so that stopping between them leaves working software**,
  never a half-migrated codebase.
- **Tests document intent.** A failing test should explain what was meant to be true.
