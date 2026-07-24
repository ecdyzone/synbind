# synbind

> In silico pipeline for designing extracellular binder domains for synthetic Notch receptors.

Given a target antigen, **synbind** finds known binders, builds deployable scFv or
ectodomain constructs, predicts complex structures with Boltz, optionally simulates
membrane geometry, screens for off-target binding, and delivers a ranked candidate
list ready for experimental validation.

---

## Background

[Synthetic Notch (synNotch)](https://www.nature.com/articles/nature18622) is an
engineered receptor system that allows cells to sense a specific antigen and
respond with a user-defined genetic program. The only target-specific part is the
extracellular **binder domain** — designing this correctly determines the
specificity of the entire circuit.

synbind automates the computational steps for binder domain selection and
validation, reducing weeks of manual literature search and structure analysis
to a single command.

---

## Quick start

```bash
pip install synbind

# Find and validate binder candidates for CD47
synbind run --target Q08722

# Include membrane simulation for top 3 candidates
synbind run --target Q08722 --membrane

# Full run with off-target screening
synbind run --target Q08722 --membrane --check-offtargets
```

Results land in `./results/{run_id}/`:
- `candidates.csv` — ranked binder list with all metrics
- `offtargets.csv` — off-target risk per natural receptor binder
- `{construct_id}/complex_predicted.pdb` — predicted complex structure
- `{construct_id}/complex_membrane.pdb` — membrane-context structure (if `--membrane`)

---

## Installation

```bash
# From source
git clone https://github.com/yourhandle/synbind
cd synbind
pip install -e ".[dev]"

# Install Boltz (structure prediction engine)
pip install boltz && boltz download

# Install local BLAST (for off-target screening)
sudo apt install ncbi-blast+

# Install DeepTMHMM (transmembrane topology)
pip install pydeeptyhmm
```

Python 3.11+ required. Boltz runs on CPU (slow) or GPU (fast). A CUDA GPU is
recommended for full runs with `--diffusion-samples 5`.

---

## Example output

```
$ synbind run --target Q08722

Target: CD47 (Q08722) — Signal-regulatory protein
Extracellular domain: residues 19–139 (121 aa) · membrane protein (1 TM helix)

Searching SAbDab...          12 antibodies found
Searching UniProt/STRING...  3 natural receptors found (off-target risk: HIGH)
Total candidates:            15

Building constructs...
  12 × scFv (VH-VL and VL-VH orientations = 24 constructs)
  3  × ectodomain
  Total: 27 constructs

Running Boltz (27 complexes) ─────────────────── [27/27] done

┌──────────────────────┬─────────┬───────┬────────┬──────────┬───────────┐
│ Binder               │ Type    │ ipTM  │ pLDDT  │ RMSD (Å) │ OT risk   │
├──────────────────────┼─────────┼───────┼────────┼──────────┼───────────┤
│ Hu5F9 (VH-VL)        │ scFv    │ 0.91  │ 87.2   │ 1.4      │ —         │
│ CC-90002 (VH-VL)     │ scFv    │ 0.88  │ 85.1   │ —        │ —         │
│ SIRPα D1             │ ectodm  │ 0.85  │ 83.7   │ 1.8      │ HIGH      │
│ ...                  │ ...     │ ...   │ ...    │ ...      │ ...       │
└──────────────────────┴─────────┴───────┴────────┴──────────┴───────────┘

Off-target analysis for SIRPα D1:
  SIRP-β1 (P78552)  →  ipTM 0.81  →  HIGH risk
  SIRP-γ  (O00241)  →  ipTM 0.74  →  MEDIUM risk

Results: results/cd47_20240722_143201/
```

---

## Pipeline overview

```
UniProt ID
    │
    ▼
[1] Target preparation   — sequence fetch, TM topology (DeepTMHMM)
    │
    ▼
[2] Binder search        — SAbDab (antibodies), UniProt/STRING (receptors)
    │
    ▼
[3] Construct building   — scFv assembly (anarci), ectodomain trimming
    │
    ▼
[4] Structure prediction — Boltz folding + docking
    │ (optional)
    ├─[4b] Membrane simulation — COMPLIP + Boltz with lipid bilayer
    │
    ▼
[5] Off-target screening — BLAST → Boltz docking on top hits (receptors only)
    │
    ▼
[6] Benchmark            — iRMSD vs PDB crystal (when co-crystal exists)
    │
    ▼
[7] Ranked output        — candidates.csv + PDB files
```

---

## Key metrics

| Metric | What it means | Threshold |
|--------|--------------|-----------|
| `ipTM` | Boltz confidence in binding pose (0–1) | > 0.8 |
| `pLDDT` | Per-residue structure confidence (0–100) | > 70 |
| `RMSD (Å)` | Predicted vs crystal interface deviation | < 2.0 |
| `OT risk` | Off-target binding risk (natural receptors) | LOW preferred |

See [`docs/metrics.md`](docs/metrics.md) for full interpretation.

---

## Data sources

| Source | Purpose |
|--------|---------|
| UniProt REST API | Target sequences, topology, interaction partners |
| SAbDab | Antibody VH/VL sequences and PDB co-crystals |
| anarci | Local VH/VL domain extraction (no API call) |
| RCSB PDB | Crystal structures for benchmarking |
| STRING | Experimental interaction partners (natural receptors) |
| BioGRID | Cell-type expression of receptor binders |
| DeepTMHMM | Transmembrane topology prediction |
| NCBI BLAST | Off-target sequence homology search |
| Boltz + COMPLIP | Structure prediction and membrane simulation |

---

## Project structure

```
synbind/
├── synbind/              source code
├── docs/                 extended documentation
├── data/
│   ├── benchmark_set.csv
│   ├── linkers.yaml
│   └── notch_backbone.yaml
├── tests/
└── results/              gitignored; created at runtime
```

---

## Roadmap

**v1 (current)** — CLI tool for researchers comfortable with a terminal.
Full pipeline: search → construct → predict → off-targets → benchmark → rank.

**v2 (planned)** — Web interface for wet-lab biologists. Same core pipeline,
React + FastAPI frontend, embedded Mol* structure viewer, one-click export.

See [`docs/roadmap.md`](docs/roadmap.md) and [`IDEAS.md`](IDEAS.md) for extensions.

---

## License

MIT
