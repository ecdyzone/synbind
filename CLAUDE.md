# CLAUDE.md — synbind

Read this file in full before writing code or proposing architecture changes.
Then read [`docs/decisions.md`](docs/decisions.md) — it records every significant
decision and what was rejected. Do not silently contradict it.

When in doubt, ask the developer rather than inferring.

> ## ⚠️ Never fabricate a biological sequence
>
> Not an amino acid sequence, not an accession, not a PDB ID, not a DOI.
>
> An invented sequence that looks plausible is worse than none: it will be
> assembled, validated, exported, and potentially ordered for synthesis. Nothing
> downstream can detect it.
>
> If a real value is unavailable, write the literal sentinel `TODO`. The registry
> loader **must refuse** to load records containing `TODO` and raise an error naming
> the file and record. Incomplete data fails loudly; it never passes silently.
>
> This applies to test fixtures too — use obviously-synthetic strings
> (`AAAA...`, `TESTSEQ`), never something that could be mistaken for real data.

---

## What this project is

**synbind** designs and validates extracellular binder domains for **synthetic
receptors** — engineered receptors that let a cell sense a specific ligand and
respond with a user-defined genetic program.

Given a target ligand, it:

1. Retrieves published binders for that ligand
2. Assembles a complete receptor construct against a chosen scaffold
3. Validates the design against rules that catch silent, expensive failures
4. Recommends one candidate, with visible reasoning and alternatives

Users are **wet-lab biologists with no bioinformatics background**. The primary
interface is a web application. Design accordingly — see [Interface principles]
(#interface-principles).

The project starts with synNotch and SNIPR scaffolds but generalizes: receptor
scaffolds are versioned data, not code.

### Lab context

The originating application is xenotransplantation. Donor cells carry a synthetic
receptor that senses a recipient immune signal and conditionally expresses an
immunomodulatory transgene — locally and only when needed, rather than
constitutively.

This context matters for one reason above all: **the receptor is expressed in one
species' cells and senses a ligand from another.** That asymmetry drives the
off-target model (ADR-009) and is not something general-purpose tools handle.

---

## Biology you must understand first

Read [`docs/biology.md`](docs/biology.md) before touching assembly, validation, or
anything that interprets a metric. The essentials:

- The **binder domain is the only part that determines specificity.** Everything
  else in the construct is modular and off-the-shelf.
- The binder must be a **single polypeptide** — it is fused directly to the receptor
  core. This rules out full IgG. Acceptable: scFv, nanobody (VHH), single-chain
  natural receptor ectodomain.
- **Ligand modality decides which scaffolds can work.** Classical synNotch requires
  mechanical force from a ligand on an opposing cell and is largely inert to a
  soluble ligand. This is the single most important check the tool performs.
- **Binding too tightly is a failure mode**, not a success. Excessive affinity risks
  ligand-independent (tonic) signaling.
- A binder that engages something on its **own host cell** causes constitutive
  signaling and a dead experiment.

---

## Scope

### In scope (v1)

Design and validation of the receptor ectodomain construct, **at the protein level**.

### Out of scope

| Excluded | Why |
|---|---|
| Codon optimization, cloning, ordering | ADR-001 — mature tooling exists |
| Structure prediction | ADR-004 — v2 enrichment layer |
| Membrane simulation | ADR-017 |
| Affinity (Kd) prediction | Backlog; see `IDEAS.md` |
| De novo binder design | Separate project; see `IDEAS.md` |
| Molecular dynamics | Out of scope indefinitely |

Do not implement anything from `IDEAS.md` without explicit instruction.

---

## Architecture

```
synbind/
├── CLAUDE.md
├── README.md
├── IDEAS.md
├── pyproject.toml              ← project metadata + [tool.pixi]
├── pixi.lock                   ← committed
├── Dockerfile
├── docs/
│   ├── decisions.md            ← ADR log; read before changing architecture
│   ├── biology.md              ← receptor biology, modality, binder requirements
│   ├── roadmap.md              ← modular deliverables
│   └── third_party_tools.md    ← external tool survey (all unverified)
├── synbind/
│   ├── __init__.py
│   ├── models.py               ← pydantic models; nothing imports until stable
│   ├── registry/               ← loads + validates versioned reference data
│   │   ├── targets.py
│   │   ├── scaffolds.py
│   │   └── binders.py
│   ├── assembly.py             ← construct assembly, scaffold-agnostic
│   ├── validation/
│   │   ├── rules.py            ← rule registry + dispatch
│   │   ├── modality.py         ← scaffold/modality compatibility
│   │   ├── liabilities.py      ← sequence liability motifs
│   │   └── report.py           ← findings → ValidationReport
│   ├── recommend.py            ← ranking + rationale
│   ├── provenance.py           ← lineage capture
│   └── web/
│       ├── app.py              ← FastAPI application
│       ├── routes.py           ← thin; no business logic
│       └── templates/          ← Jinja + HTMX partials
├── data/
│   ├── README.md               ← read before editing any data file
│   ├── targets.yaml
│   ├── scaffolds/
│   │   ├── synnotch.yaml
│   │   └── snipr.yaml
│   ├── binders_seed.yaml       ← hand-curated; the high-confidence tier
│   └── linkers.yaml
└── tests/
    ├── test_models.py
    ├── test_registry.py
    ├── test_assembly.py
    ├── test_validation.py
    ├── test_recommend.py
    ├── test_web.py
    └── fixtures/
```

`cli.py` is deliberately absent from v1 — see ADR-006.

---

## Data models

Pydantic, not dataclasses (ADR-016). Validation happens at ingestion. No raw dicts
cross module boundaries.

```python
Modality    = Literal["membrane_bound", "soluble"]
BinderFormat= Literal["antibody", "nanobody", "receptor_ectodomain"]
ChainRole   = Literal["VH", "VL", "scFv", "VHH", "ectodomain"]
Severity    = Literal["pass", "warn", "fail"]


class Provenance(BaseModel):
    source: str                   # "PDB" | "UniProt" | "literature" | "curated"
    identifier: str               # PDB ID, accession, DOI
    url: str | None
    retrieved_at: datetime | None
    curated_by: str | None
    note: str | None


class Target(BaseModel):
    accession: str                # UniProt
    gene: str
    aliases: list[str]            # so a user typing "CD3" resolves to CD3E
    species: str
    modality: Modality            # ADR-002
    role: Literal["candidate_sensor", "benchmark"]
    rationale: str
    provenance: Provenance


class ScaffoldPart(BaseModel):
    name: str                     # signal_peptide | hinge | core | tmd | output
    sequence: str
    species_origin: str           # drives immunogenicity flagging
    provenance: Provenance


class Scaffold(BaseModel):
    id: str                       # "snipr_v1"
    display_name: str
    compatible_modalities: list[Modality]     # ADR-002
    parts: list[ScaffoldPart]                 # ordered, N→C
    binder_insertion_index: int
    accepted_formats: list[BinderFormat]
    max_binder_length_aa: int | None
    default_linker_id: str
    provenance: Provenance


class Binder(BaseModel):
    id: str
    name: str                     # "UCHT1", "bevacizumab", "SIRPa_D1"
    target_accession: str
    target_species: str           # anti-human vs anti-mouse — a silent killer
    format: BinderFormat
    chain_role: ChainRole
    origin_species: str           # murine | humanized | human
    sequences: dict[ChainRole, str]
    provenance: Provenance        # no provenance → cannot be recommended (ADR-011)


class ConstructSegment(BaseModel):
    name: str
    start: int                    # 0-based, half-open
    end: int
    kind: Literal["scaffold_part", "binder", "linker"]


class Construct(BaseModel):
    id: str
    binder_id: str
    scaffold_id: str
    sequence: str
    segments: list[ConstructSegment]     # drives the annotated sequence view
    orientation: str | None              # "VH-VL" | "VL-VH"
    length: int


class ValidationFinding(BaseModel):
    rule_id: str                  # "modality.compatible"
    severity: Severity
    title: str                    # one line, user-facing, no jargon
    detail: str
    evidence: dict[str, str]
    suggestion: str | None        # e.g. an alternative scaffold


class ValidationReport(BaseModel):
    construct_id: str
    findings: list[ValidationFinding]

    @property
    def blocking(self) -> bool: ...


class Recommendation(BaseModel):
    construct: Construct
    report: ValidationReport
    score: float
    rationale: str                # why THIS one, in plain language
    caveats: list[str]


class RejectedCandidate(BaseModel):
    binder_id: str
    reason: str                   # shown in the UI (ADR-008)


class DesignResult(BaseModel):
    target: Target
    scaffold: Scaffold
    recommended: Recommendation | None
    alternatives: list[Recommendation]
    rejected: list[RejectedCandidate]
    generated_at: datetime
    synbind_version: str
    data_version: str             # git SHA of data/
```

Note `DesignResult` carries enough to reproduce itself. That is deliberate (ADR-011).

---

## Validation rules

Each rule is independently testable and explainable in one sentence (ADR-005).

| Rule ID | Severity on failure | Checks |
|---|---|---|
| `provenance.present` | fail | Binder has a citable source |
| `species.match` | fail | Binder targets the intended species |
| `modality.compatible` | fail | Scaffold supports the ligand's modality |
| `format.single_chain` | fail | Binder is a single polypeptide |
| `format.length` | warn | Within scaffold's size limit |
| `liability.unpaired_cys` | warn | Odd cysteine count in the binder |
| `liability.nglyc_sequon` | warn | N-X-S/T motif, X≠P |
| `liability.deamidation` | warn | NG / NS motif |
| `liability.isomerization` | warn | DG motif |
| `immunogenicity.origin` | warn | Non-human framework origin |

Rules must never raise on unexpected input — they return a finding. A rule that
crashes takes down the whole report.

---

## Assembly behaviour

`assembly.py` is scaffold-agnostic. It reads the scaffold's ordered `parts` and
`binder_insertion_index`, inserts the binder, and records a `ConstructSegment` for
every region so the UI can render an annotated sequence.

Per binder `chain_role`:

| `chain_role` | Constructs produced |
|---|---|
| `VH_VL` (two chains supplied) | **Two** — one `VH-linker-VL`, one `VL-linker-VH` |
| `scFv` (already fused) | One — used as-is, no linker inserted |
| `VHH` | One — single domain, no linker |
| `ectodomain` | One — no linker |

Both scFv orientations are generated because they do not always behave identically
and there is no reliable sequence-level way to predict which is better. They are
presented as distinct candidates; the user or a later structural stage decides.

---

## Ranking and recommendation

`recommend.py` produces an ordered list. Two principles govern it:

**1. Blocking failures are not ranked, they are excluded.** Any construct with a
`fail` finding becomes a `RejectedCandidate` with a user-facing reason. It never
receives a score. Ranking applies only to constructs that passed every blocking rule.

**2. The score is a tiebreaker, not the explanation.** The rationale shown to users
is generated from the findings themselves, in plain language — never from the number.

```python
score = 1.0
score -= 0.10 * n_liability_warnings        # capped contribution of -0.30
score -= 0.15 if origin_species not in ("human", "humanized") else 0.0
score -= 0.10 if not has_structural_evidence else 0.0
score += 0.10 if has_cocrystal_with_target else 0.0
score -= 0.05 * max(0, (length - 300) // 50)   # gentle length penalty
score = clamp(score, 0.0, 1.0)
```

Weights are deliberately coarse. They encode ordinal preferences — fewer liabilities
is better, human framework is better, direct structural evidence is better — and
nothing more.

**Do not present the raw score as a headline number in the UI.** It implies a
precision that does not exist. Use it to order candidates; explain with findings.

When adding a factor, add it here and in `docs/decisions.md`. A ranking that drifts
without record becomes unexplainable, and an unexplainable recommendation is one no
user should act on.

---

## Interface principles

These are requirements, not decoration. Non-technical users are the primary audience.

- **Never require an accession.** Autocomplete on gene names and aliases. Ambiguity
  (e.g. "CD3" → CD3ε/δ/γ) is shown and resolved, never silently guessed.
- **Lead with a recommendation**, not a table. A ranked list moves the decision back
  onto a user who has no basis for making it.
- **Three layers** (ADR-008): recommendation → comparison → raw. Raw data is always
  present, never primary.
- **No bare numbers.** Every metric ships with what it means and whether this value
  is good — inline and visible, not hidden in a hover tooltip.
- **Provenance is clickable everywhere.** Users must be able to check the work.
- **Failures in the user's language.** "No published human-specific binders for this
  target" — never "registry returned 0 records".
- **Show what was rejected and why.** This builds more trust than showing only
  survivors.
- **Loudest warnings for the expensive mistakes** — modality mismatch, wrong species,
  host-cell cross-reactivity.

---

## Coding conventions

- Type hints on all public functions. Return pydantic models, never raw dicts.
- `structlog.get_logger()` at module top. No bare `print()`.
- **No business logic in `web/`.** Routes call core functions and render results.
  The same functions must be callable from a future CLI without modification.
- Steps are **pure functions**: declared inputs, declared outputs, no hidden state,
  no hardcoded paths. This is what makes the v2 workflow-engine migration mechanical
  (ADR-013).
- Reference data in `data/` is loaded and validated through `registry/`. Never parse
  YAML ad hoc elsewhere. The loader **must reject** any record containing the `TODO`
  sentinel, with an error naming the file and record.
- Tests never hit live services. External sources get fixtures plus a separate
  scheduled canary (ADR-012).
- `ruff` for lint and format; `mypy` for types. Both run in CI.

---

## Environment

`pixi` manages environments through `pyproject.toml` (ADR-015). Standard packaging
metadata is preserved, so `pip install -e .` works.

```bash
pixi install
pixi run test
pixi run serve
```

v1 dependencies are pure Python: `fastapi`, `uvicorn`, `jinja2`, `pydantic`,
`pyyaml`, `structlog`, plus `pytest`, `ruff`, `mypy` for development.
Conda-channel bioinformatics binaries arrive in v2.

---

## Current state

**Keep this section accurate.** Sessions do not resume; this is how the next one
learns where things stand. Update it whenever a step's status changes.

_Last updated: 2026-07-27_

| # | Step | Status |
|---|---|---|
| 0 | Docs, decisions, project config, data schemas | ✅ done |
| 1 | `models.py` — pydantic models | ⬜ not started |
| 2 | `registry/` — load + validate `data/` | ⬜ not started |
| 3 | `assembly.py` — construct assembly | ⬜ not started |
| 4 | `validation/` — rules + report | ⬜ not started |
| 5 | `recommend.py` — ranking + rationale | ⬜ not started |
| 6 | `web/` — FastAPI + Jinja + HTMX | ⬜ not started |
| 7 | Dockerfile, CI, deployment | ⬜ not started |

**Next action:** step 1. Write `synbind/models.py` from the [Data models](#data-models)
section above, with tests. Nothing else imports it until it is stable.

### Known blockers

- **All sequences in `data/` are `TODO`.** Scaffold parts and binder sequences must
  be sourced from primary references before anything can be assembled end to end.
  Steps 1–5 can be built and tested against synthetic fixtures without them.
- **All accessions in `data/targets.yaml` are `verified: false`** — recorded from
  memory, unchecked against UniProt.
- **`docs/third_party_tools.md` is entirely unverified.** Nothing in v1 depends on
  it; relevant from v1.1 onward.

### Acceptance scenario for v1

The single behaviour that must work, and the one to demonstrate first:

1. User selects target **TGF-β1** and scaffold **synNotch**
2. Tool blocks with a plain-language explanation — a secreted ligand cannot
   generate the mechanical force this scaffold requires — and suggests **SNIPR**
3. User switches to SNIPR; validation passes; the construct assembles and the
   annotated sequence is shown with every source clickable

If that path works end to end, v1 has demonstrated its central claim. Build toward
it, not around it.

See [`docs/roadmap.md`](docs/roadmap.md) for deliverables beyond v1.
