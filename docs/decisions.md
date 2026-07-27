# Architecture decision log

Every significant decision, why it was made, and what was rejected.

**Read this before proposing changes.** If you want to do something that contradicts
an entry here, that's fine — but supersede the entry explicitly rather than working
around it. Silent contradiction is how a codebase loses its shape.

Format: numbered, append-only. To reverse a decision, add a new entry that says so
and mark the old one `SUPERSEDED BY ADR-nn`.

---

## ADR-001 — Scope: design and validate, nothing downstream

**Status:** accepted

**Context.** The manual work this tool replaces spans literature search, binder
selection, construct assembly, sanity checking, codon optimization, cloning design,
and ordering. That's too much surface for one tool to do well.

**Decision.** synbind covers **design and validation of the receptor ectodomain
construct, at the protein level**. Codon optimization, cloning strategy, restriction
site management, and vendor ordering are out of scope.

**Consequences.** Output is a protein sequence plus a validation report. Users take
that into their existing DNA tooling. Keeps the tool small and avoids competing with
mature software that already does the DNA half well.

**Rejected.** Full construct-to-order pipeline. Too large, and the DNA tooling
ecosystem is already good — see `third_party_tools.md`.

---

## ADR-002 — Ligand modality is a first-class property

**Status:** accepted

**Context.** Classical synNotch requires mechanical force from a ligand presented on
an opposing cell; it is largely inert to a soluble ligand. Other scaffolds (SNIPR
variants, MESA-type receptors) tolerate or are designed for soluble input. The lab's
targets include both membrane-bound (CD3ε, CD47) and soluble (VEGF-A, TGF-β1) ligands.

**Decision.** Every target carries a `modality` field (`membrane_bound` | `soluble`).
Every scaffold declares `compatible_modalities`. Mismatch is a **hard validation
failure** with an explanatory message and a suggested alternative scaffold.

**Consequences.** This is one of the tool's primary contributions. A modality/scaffold
mismatch is invisible in any single paper, cheap to encode, and expensive to discover
in the lab. It shapes the data model and the validation stage.

**Rejected.** Assuming membrane-bound ligands throughout, as the original brainstorming
docs did. It would have silently produced non-functional designs for two of the four
initial targets.

---

## ADR-003 — Receptor scaffolds are versioned data, not code

**Status:** accepted

**Context.** The project starts with synNotch and SNIPR but is intended to generalize.
Hardcoding one receptor family's assumptions is how tools become un-generalizable.

**Decision.** Each scaffold is a YAML file in `data/scaffolds/` declaring its ordered
parts, sequences, part provenance, binder constraints, and compatible modalities.
Assembly logic is scaffold-agnostic and consumes this data.

**Consequences.** Adding a receptor family is a YAML file plus tests, not a refactor.
Scaffold data is versioned in git alongside code, so any construct can be reproduced.

**Rejected.** Per-scaffold Python subclasses. More flexible than needed, and it puts
biological reference data in code where non-programmers cannot review it.

---

## ADR-004 — No structure prediction in v1

**Status:** accepted

**Context.** Structure prediction (Boltz, AlphaFold) is the heaviest, slowest, and
most GPU-dependent part of any design pipeline. It is also the least differentiated —
several commercial web frontends already serve it well.

**Decision.** v1 performs **no structure prediction**. Validation is entirely
sequence-level and runs in seconds on a laptop. Structure becomes a v2 enrichment
layer that attaches scores to artifacts that already exist.

**Consequences.** v1 ships far sooner, has no GPU dependency, no MSA server dependency,
and no queue. The validation it does perform is real — see ADR-005.

**Rejected.** Boltz-centric v1, as the original brainstorming docs proposed. It would
have made the first usable version months away and gated on hardware access.

---

## ADR-005 — Validation is sequence-level and rule-based

**Status:** accepted

**Context.** Given ADR-004, "validate" needs a concrete meaning that does not involve
a GPU.

**Decision.** Validation is a set of named, individually-testable rules producing a
report with `pass` / `warn` / `fail` per rule:

1. **Provenance** — is the binder published, citable, and against the right target?
2. **Species** — does the binder target the intended species? An anti-mouse binder for
   a human ligand is a silent, expensive failure.
3. **Modality compatibility** — ADR-002.
4. **Format compatibility** — single-chain, within scaffold size limits, correct
   fusion terminus.
5. **Developability liabilities** — unpaired cysteines, N-glycosylation sequons
   (N-X-S/T, X≠P), deamidation motifs (NG, NS), isomerization motifs (DG).
6. **Immunogenicity indicators** — origin species and framework humanness.

**Consequences.** Every rule is independently testable, explainable to a user in one
sentence, and instant. Rules are data-driven where possible so new ones don't require
restructuring.

**Rejected.** Scattered `assert` statements inside pipeline steps. Not reportable,
not explainable, and invisible to users.

---

## ADR-006 — Core first, web second, CLI later

**Status:** accepted

**Context.** Much bioinformatics software is a CLI with a GUI bolted on afterward.
The result is a UI that exposes CLI concepts — flags, file paths, job IDs — to users
who don't have those concepts. The root cause is a domain model designed for the CLI's
convenience, which the GUI must then reverse-engineer.

**Decision.** Build the typed core first. The web UI and the CLI are both **thin
clients** over it; neither is privileged. The web UI ships in v1; the CLI comes later.

**Consequences.** No business logic in `web/` or `cli.py`. Both call the same
functions and render the same typed results. Primary users are non-technical, so the
web interface gets built and validated first.

**Rejected.** CLI-first with a web layer in v2. It would have baked CLI assumptions
into the core and produced exactly the bolted-on UI this project exists to avoid.

---

## ADR-007 — Curated seed binder data before live search

**Status:** accepted

**Context.** Live search across RCSB, SAbDab, and therapeutic antibody databases is
where most of the implementation risk sits: inconsistent APIs, antibody chain
detection, chain-to-entity mapping, sequence extraction, species annotation.

**Decision.** v1 ships a **hand-curated binder registry** (`data/binders_seed.yaml`)
covering the initial targets. Live search is a later, independently-scoped deliverable
that layers on top.

**Consequences.** v1 is end-to-end real — real models, real assembly, real validation,
real UI — with correct data, at a fraction of the effort. This is a walking skeleton,
not a mock.

The curated registry is **not throwaway**. It remains permanently as the
high-confidence tier; automated search results layer beneath it with lower confidence.

**Rejected.** Live search in v1. High risk, and demo quality would be worse because
automated extraction produces errors that curation does not.

---

## ADR-008 — Recommend one, show the alternatives

**Status:** accepted

**Context.** A ranked table of candidates is a bioinformatician's output. A wet-lab
user needs a decision, with enough visible reasoning to trust or override it.

**Decision.** The interface leads with a **single recommendation and its rationale**.
Alternatives are one click away. Three layers of progressive disclosure:

1. **Recommendation** — one binder, why, caveats, copy/export
2. **Comparison** — all candidates, plus **candidates that were rejected and why**
3. **Raw** — full annotated sequence, all source records, query log, JSON

**Consequences.** Raw data must always be present but never primary. The rejected-
candidates list is deliberate: showing what was excluded, and why, builds more trust
than showing only what survived.

**Rejected.** Table-first UI. It moves the decision back onto the user and gives them
no basis for making it.

---

## ADR-009 — Off-target screening is species-directed, in two directions

**Status:** accepted (design fixed in v1 models; implementation is v2)

**Context.** The lab's application expresses receptors in one species' cells to sense
a ligand from another. This creates an asymmetry no general-purpose tool handles.

Earlier brainstorming proposed searching the proteome for homologs **of the binder**.
This is incorrect: it finds the binder's paralogs, not the binder's unintended targets.
An off-target is something the binder *binds*, not something that *resembles* it.

**Decision.** Two distinct screens with different reference proteomes and different
consequences:

| Screen | Reference proteome | Failure mode |
|---|---|---|
| **trans** | ligand's species | Receptor fires on the wrong cell |
| **cis** | host cell's species | Receptor fires on its *own* host cell — constitutive signaling, dead experiment |

Both search for homologs **of the target ligand**, then evaluate binder engagement.
Structure-based search (Foldseek) is preferred over sequence-based, since
cross-reactivity is a structural property and sequence-dissimilar proteins can share
a fold.

**Consequences.** The cis screen is the novel contribution and the most valuable
single feature planned. Data models reserve space for both from v1 even though
implementation lands in v2.

**Rejected.** Binder-homology search. Biologically incorrect — it answers a question
nobody asked.

---

## ADR-010 — Payload/output module is configurable, never hardcoded

**Status:** accepted

**Context.** The intracellular output module (transactivator and its response element)
is orthogonal to ectodomain design, but hardcoding one blocks a whole class of future
use.

**Decision.** GAL4-UAS is the **default**, declared in scaffold data like any other
part. It is never hardcoded. A payload registry is a backlog item.

**Consequences.** Zero cost now, no migration later.

---

## ADR-011 — Provenance is a product feature, not bookkeeping

**Status:** accepted

**Context.** The trust question — "would a user act on this output?" — is the central
product risk. Users are scientists; they will and should verify claims before
committing lab resources.

**Decision.** Every output carries full lineage: which source, which version, retrieved
when, which tool versions, which parameters. Every sequence shown in the UI links to
its source record. A binder without a citable source **cannot** appear in a
recommendation.

**Consequences.** This serves users and reproducibility with one mechanism. It also
means provenance must be threaded through the data model from the start — it cannot be
retrofitted.

---

## ADR-012 — Contract tests at every external boundary

**Status:** accepted

**Context.** External data sources change response shapes without notice. A pipeline
that silently produces wrong output is worse than one that fails loudly.

**Decision.** Every external source gets a schema definition, a fixture-based test,
and a **scheduled canary** running against the live source in CI. Unit tests never
hit live services.

**Consequences.** Breakage surfaces as a failed scheduled run, not as a user acting on
bad data.

---

## ADR-013 — Deployment: containerized web app, workflow engine deferred

**Status:** accepted

**Context.** Users need a URL they can click. The developer needs the option to run
heavy work on a laptop, a SLURM cluster, or cloud, without rewriting.

**Decision.**

| Layer | Target | When |
|---|---|---|
| Interactive app | Hugging Face Spaces, Docker SDK | v1 |
| Heavy enrichment | Nextflow → local / SLURM / cloud batch | v2 |

Nextflow is **not** adopted in v1, because v1 has no workload that justifies it —
HTTP calls and sequence manipulation, seconds on one machine. Adopting a workflow
engine before there is a workflow to engineer adds ceremony without benefit.

**To earn the option cheaply**, v1 code follows the constraints a workflow engine
would impose anyway: steps are pure functions with declared file inputs and outputs,
no hidden state, no hardcoded paths, external tools behind thin adapters, and Docker
from day one.

**Consequences.** Wrapping the v2 enrichment DAG in Nextflow becomes mechanical
rather than a rewrite.

---

## ADR-014 — Server-rendered frontend: FastAPI + Jinja + HTMX + Tailwind

**Status:** accepted

**Context.** The interface quality is a primary differentiator, but the project must
remain maintainable by a Python developer after the original author moves on.

**Decision.** Server-rendered HTML with HTMX for interactivity. No JavaScript
framework, no build step, no `node_modules`.

**Consequences.** Python owns all logic. Progressive disclosure (ADR-008) maps almost
exactly onto HTMX fragment swaps — each "expand" is a server-rendered partial.
Tailwind gives full visual control without a component framework's default look.

Accepted limitation: lower ceiling for rich client-side interactivity. v1 has none —
it is forms, tables, and disclosure. A structure viewer, if ever needed, drops in as
a web component regardless.

**Rejected.** React/TypeScript SPA. Higher ceiling, but adds a second language and a
build toolchain to a Python codebase, narrowing the pool of people who can maintain it.

---

## ADR-015 — Environment management: pixi

**Status:** accepted

**Context.** v1 dependencies are pure Python. v2 requires conda-only bioinformatics
binaries (Foldseek, MMseqs2, antibody numbering tools and their HMMER dependency).

**Decision.** `pixi` for environment and lock management, configured through
`pyproject.toml`. Standard packaging metadata is preserved, so `pip install -e .`
continues to work.

**Consequences.** One tool spans PyPI and conda-forge/bioconda. Multi-platform
lockfile. Adopted at v1 despite no conda dependencies yet, because retrofitting an
established repo costs more than starting with it. Also relevant: many HPC clusters
disallow container runtimes, and pixi installs userspace binaries without root.

**Rejected.** `uv` alone — excellent for Python, but cannot install bioconda binaries,
which would force those tools into containers even during development, where the
rebuild loop makes iteration slow.

---

## ADR-016 — Pydantic models, no raw dicts across module boundaries

**Status:** accepted

**Context.** Data arrives from external APIs and hand-edited YAML. Both are
untrustworthy in different ways.

**Decision.** Pydantic models for all inter-module data. Validation happens at the
boundary, on ingestion. No raw dicts pass between modules.

**Consequences.** Malformed reference data fails at load with a precise error, not
three steps later. JSON serialization for the web layer is free.

---

## ADR-017 — COMPLIP is not a dependency

**Status:** accepted

**Context.** Early brainstorming assumed a Boltz extension called COMPLIP providing
`--membrane` and `--membrane_thickness` flags for lipid bilayer simulation. This tool
could not be confirmed to exist, and the described interface does not match mainline
Boltz.

**Decision.** No dependency on COMPLIP. Membrane context is out of scope entirely for
v1 and v2. If it returns, established tooling (CHARMM-GUI Membrane Builder, OPM/PPM,
MemProtMD) is preferred over an unverified extension.

**Consequences.** Removes an unfalsifiable assumption from the architecture.
