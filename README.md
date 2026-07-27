# synbind

> Design and validate extracellular binder domains for synthetic receptors.

Synthetic receptors let a cell sense a specific ligand and respond with a
user-defined genetic program. The **binder domain** — the part that recognises the
ligand — is the only component that determines specificity, and choosing it correctly
is slow, manual work.

synbind takes a target ligand, retrieves published binders, assembles a complete
receptor construct, validates the design, and recommends a candidate with visible
reasoning.

Built for wet-lab biologists. No bioinformatics background required.

> **Status: early development.** The interface and data model are still moving.
> Not yet suitable for production use.

---

## What it checks

Some design failures are invisible on paper and expensive in the lab. synbind is
built around catching them:

- **Ligand modality vs. receptor scaffold.** Classical synNotch requires mechanical
  force from a ligand presented on a neighbouring cell. Point it at a soluble ligand
  like TGF-β1 and it will not respond. synbind refuses the combination and suggests
  a scaffold that works.
- **Species mismatch.** A binder raised against the mouse form of a ligand will not
  engage the human form. Easy to miss, costly to discover.
- **Sequence liabilities.** Unpaired cysteines, N-glycosylation sequons, deamidation
  and isomerization motifs — cheap to detect, predictive of poor expression or
  aggregation.
- **Provenance.** Every sequence traces to a published source. Anything uncitable is
  excluded from recommendations.

---

## How it works

```
Target ligand
     │
     ▼
[1] Resolve target ── gene name → accession, modality, species
     │
     ▼
[2] Retrieve binders ── curated registry of published binders
     │
     ▼
[3] Assemble constructs ── binder + linker + scaffold parts
     │
     ▼
[4] Validate ── modality, species, format, liabilities, provenance
     │
     ▼
[5] Recommend ── one candidate + rationale + alternatives + what was rejected
```

Everything runs in seconds on a laptop. No GPU, no cluster, no job queue.

---

## Interface

Three layers, so the depth is there without being in the way:

1. **Recommendation** — one binder, why it was chosen, caveats, copy/export
2. **Comparison** — all candidates side by side, plus what was rejected and why
3. **Raw** — annotated full sequence, source records, query log

---

## Receptor scaffolds

Scaffolds are versioned data, not code. Each declares its parts, its binder
constraints, and which ligand modalities it supports.

| Scaffold | Membrane-bound ligands | Soluble ligands |
|---|---|---|
| synNotch | ✅ | ❌ force-dependent |
| SNIPR | ✅ | ✅ |

Adding a scaffold is a YAML file plus tests.

---

## Development

```bash
git clone <repo>
cd synbind

pixi install
pixi run test
pixi run serve      # http://localhost:8000
```

Requires [pixi](https://pixi.sh). Standard packaging metadata is preserved, so
`pip install -e .` also works for the Python-only dependencies.

---

## Project layout

```
synbind/      core library — models, registry, assembly, validation, web
data/         versioned reference data: targets, scaffolds, binders, linkers
docs/         architecture decisions, biology, roadmap
tests/        unit tests and fixtures
```

---

## Documentation

| Document | Contents |
|---|---|
| [`docs/decisions.md`](docs/decisions.md) | Every architectural decision and what was rejected |
| [`docs/biology.md`](docs/biology.md) | Receptor biology, ligand modality, binder requirements |
| [`docs/roadmap.md`](docs/roadmap.md) | Modular deliverables and their order |
| [`docs/third_party_tools.md`](docs/third_party_tools.md) | Survey of external tools and APIs |
| [`IDEAS.md`](IDEAS.md) | Future directions, explicitly out of current scope |
| [`CLAUDE.md`](CLAUDE.md) | Engineering guide and conventions |

---

## Scope

**In:** design and validation of the receptor ectodomain construct, at the protein
level.

**Out:** codon optimization, cloning design, vendor ordering, structure prediction,
membrane simulation, affinity prediction, de novo binder design. Rationale for each
is in [`docs/decisions.md`](docs/decisions.md).

---

## License

MIT
