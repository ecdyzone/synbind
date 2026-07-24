# CLAUDE.md — synbind

Read this file in full before writing any code, editing any module, or proposing
architecture changes. It contains the biological context, design decisions, and
full module map. When in doubt, ask the developer rather than inferring.

---

## What this project is

**synbind** is a CLI tool (v1) and eventual web application (v2) that designs and
validates extracellular binder domains for **synthetic Notch (synNotch) receptors**.

Given a target antigen (UniProt ID), it:
1. Finds all known binders in the literature (antibodies + natural receptors)
2. Builds deployable constructs (scFv or ectodomain)
3. Predicts the 3D structure of the binder–target complex
4. Optionally simulates the complex in a membrane bilayer context
5. Predicts off-target binding risk for non-antibody binders
6. Benchmarks predictions against PDB crystal structures
7. Delivers a ranked, annotated candidate list ready for experimental validation

The end users are wet-lab biologists (v2 GUI) and the developer himself (v1 CLI).
Do not over-engineer the CLI — keep it usable. Plan module boundaries so the GUI
layer (Flask/FastAPI + React, planned v2) can be added without refactoring the core.

---

## Critical biological context — read carefully

### What is synNotch?

Synthetic Notch (synNotch) is an engineered receptor system for mammalian cells.
It works by swapping the extracellular antigen-recognition domain of the Notch
receptor with a custom binder (scFv, nanobody, or natural receptor ectodomain).

**The full construct architecture is:**
```
[signal peptide]──[BINDER DOMAIN]──[Notch NRR/HD]──[TMD]──[RAM]──[ANK]──[transactivator]
                   ↑
             THIS IS WHAT synbind DESIGNS
```

When the binder domain engages its target antigen (on a neighboring cell or in the
extracellular matrix), the Notch core is proteolytically cleaved (by ADAM10/TACE
then γ-secretase), releasing the intracellular transactivator, which drives
user-defined gene expression.

### Why the extracellular domain is everything

The binder domain is the only part of synNotch that determines **specificity**.
The rest of the construct (Notch core, TMD, transactivator) is modular and
off-the-shelf. Getting the binder domain right means:
- It must fold correctly when anchored to a membrane (not free in solution)
- It must bind the target antigen on a **cell surface** (trans interaction)
- It must NOT bind strongly enough to trigger tonic (ligand-independent) signaling
- It should have low off-target binding (unintended gene activation is dangerous)

This is why membrane simulation matters in this project in a way it doesn't
for a generic antibody design tool.

### Binder domain options (in order of preference)

1. **scFv from known mAb** — best characterized, most data, easiest to validate
   against PDB. Convert VH+VL to single chain.
2. **Natural receptor ectodomain** — e.g., SIRPα binds CD47. Physiologically
   relevant. Higher off-target risk because natural receptors evolved to bind
   multiple partners.
3. **Nanobody (VHH)** — single domain, small, excellent folder. Search in
   databases like NbMiner or published datasets. Add to pipeline as v1.1.
4. **De novo designed binder** — RFdiffusion + ProteinMPNN. Out of scope for v1
   but a natural v2 extension. See ideas.md.

### What "membrane context" means for structure prediction

A free scFv in solution and the same scFv tethered to a Notch transmembrane domain
behave differently. The TMD imposes geometric constraints: the binder is displayed
at a fixed distance and orientation from the membrane surface, and it must engage
an antigen on an **opposing cell membrane** (trans, not cis).

COMPLIP (Boltz extension) allows placing proteins in a lipid bilayer for structure
prediction. This is not just a nice-to-have — for synNotch, the membrane geometry
affects whether the binder can physically engage the antigen. A construct that docks
perfectly in solution may fail in trans.

**However:** membrane simulation is computationally expensive and the API is
experimental. Implement it as an optional `--membrane` flag. The default pipeline
runs without it (solution-phase docking). Wet lab collaborators will use the
default; the developer uses `--membrane` for top-ranked candidates only.

### Off-target risk

For antibody-derived binders: off-target risk is generally low because mAbs are
engineered for specificity. Still worth checking.

For natural receptor ectodomains: HIGH risk. SIRPα, for example, binds CD47 but
also has isoforms that interact with other ligands. PD-1 binds PD-L1 and PD-L2.
Every natural receptor ectodomain must be screened against the human proteome.

Off-target pipeline:
1. Run BLAST of binder sequence against human proteome (UniProt human reviewed)
2. For hits with > 40% sequence identity in the binding region: flag as structural
   off-target candidates
3. Run Boltz docking for the top 5 off-target candidates
4. Report ipTM for each — any > 0.7 is a risk flag

---

## Architecture

```
synbind/
├── CLAUDE.md                      ← you are here
├── README.md
├── IDEAS.md
├── pyproject.toml
├── docs/
│   ├── pipeline.md                ← full step descriptions
│   ├── synnotch_biology.md        ← deep dive on synNotch biology
│   ├── data_sources.md            ← all external APIs
│   ├── metrics.md                 ← what each number means
│   ├── benchmarking.md            ← how to run and interpret benchmark
│   ├── membrane_simulation.md     ← COMPLIP + Boltz setup
│   ├── off_targets.md             ← off-target prediction logic
│   ├── decisions.md               ← architecture decision log
│   └── roadmap.md                 ← v1 → v2 transition plan
├── synbind/
│   ├── __init__.py
│   ├── cli.py                     ← typer app; entry point
│   ├── pipeline.py                ← orchestrates all steps end-to-end
│   ├── target.py                  ← step 1: UniProt fetch + DeepTMHMM
│   ├── binders.py                 ← step 2: SAbDab + UniProt/STRING search
│   ├── construct.py               ← step 3: scFv conversion, ectodomain trim
│   ├── structure.py               ← step 4: Boltz folding + docking
│   ├── membrane.py                ← step 4b (optional): COMPLIP membrane run
│   ├── offtarget.py               ← step 5: off-target prediction
│   ├── benchmark.py               ← step 6: RMSD vs PDB crystal
│   ├── report.py                  ← assembles CSV + summary
│   ├── models.py                  ← all dataclasses: Target, Binder, Construct, etc.
│   └── utils/
│       ├── http.py                ← shared httpx client with retry logic
│       ├── uniprot.py             ← UniProt API helpers
│       ├── sabdab.py              ← SAbDab API helpers
│       ├── pdb.py                 ← RCSB PDB helpers
│       ├── anarci_utils.py        ← VH/VL extraction with anarci
│       ├── blast.py               ← NCBI BLAST for off-target search
│       └── cache.py               ← disk cache helpers
├── data/
│   ├── benchmark_set.csv          ← 15-20 complexes with known PDB structures
│   ├── linkers.yaml               ← scFv linker sequences
│   ├── notch_backbone.yaml        ← Notch NRR/TMD/RAM/ANK sequences for context
│   └── pdb_cache/                 ← committed PDB files for benchmark set
├── tests/
│   ├── test_target.py
│   ├── test_binders.py
│   ├── test_construct.py
│   ├── test_structure.py
│   ├── test_offtarget.py
│   ├── test_benchmark.py
│   └── fixtures/
│       ├── cd47_uniprot.json
│       ├── sirpa_sabdab.json
│       └── sample_boltz_output/
└── results/                       ← gitignored; created at runtime
    └── {run_id}/
        ├── run_config.json        ← reproducibility: all params stored
        ├── candidates.csv         ← final ranked table
        ├── offtargets.csv         ← off-target risks per natural binder
        ├── benchmark_rmsd.csv     ← if benchmark mode was run
        └── {construct_id}/
            ├── construct.fasta
            ├── complex_predicted.pdb
            ├── complex_membrane.pdb   ← optional, if --membrane
            └── scores.json
```

---

## Data models (`models.py`)

All inter-module data is passed as typed dataclasses. No dicts between modules.

```python
@dataclass
class Target:
    uniprot_id: str
    gene_name: str
    full_sequence: str
    extracellular_sequence: str
    is_membrane_protein: bool
    tm_segments: list[tuple[int, int]]
    signal_peptide: tuple[int, int] | None
    organism: str

@dataclass
class Binder:
    id: str                        # e.g. "sabdab_6nzo" or "uniprot_P78552"
    name: str
    type: Literal["antibody", "receptor", "nanobody"]
    source: Literal["sabdab", "uniprot", "string", "manual"]
    heavy_seq: str | None          # antibodies only
    light_seq: str | None          # antibodies only
    sequence: str | None           # receptors / nanobodies
    pdb_complex_id: str | None     # if co-crystal exists
    interacting_cells: list[str]   # cell types known to express this binder
    off_target_risk: Literal["low", "medium", "high"] | None  # set in step 5

@dataclass
class Construct:
    id: str
    parent_binder_id: str
    sequence: str
    fasta_path: Path
    construct_type: Literal["scFv_VH_VL", "scFv_VL_VH", "ectodomain", "nanobody"]
    length: int

@dataclass
class Structure:
    construct_id: str
    pdb_path: Path
    membrane_pdb_path: Path | None
    iptm: float
    ptm: float
    mean_plddt: float
    interface_residues: list[int]

@dataclass
class OffTargetResult:
    construct_id: str
    off_target_uniprot: str
    off_target_name: str
    sequence_identity: float       # BLAST hit identity
    iptm_predicted: float | None   # from Boltz docking (top hits only)
    risk_level: Literal["low", "medium", "high"]

@dataclass
class BenchmarkResult:
    construct_id: str
    pdb_id: str
    rmsd_angstrom: float
    n_interface_residues: int
    n_aligned_atoms: int
```

---

## Pipeline steps

### Step 1 — `target.py`
1. `GET https://rest.uniprot.org/uniprotkb/{id}.json` → full entry
2. Extract canonical sequence, gene name, organism, topology annotations
3. Run DeepTMHMM subprocess → GFF3 output
4. Parse: identify extracellular segments, TM helices, signal peptide
5. Select longest extracellular segment as `extracellular_sequence`
6. Return `Target`

### Step 2 — `binders.py`
Two parallel searches, results merged and deduplicated by sequence similarity
(collapse binders with > 95% sequence identity):

**SAbDab** (antibodies + nanobodies):
```
GET https://opig.stats.ox.ac.uk/webapps/sabdab-sabdab/search/?antigen_uniprot={id}&format=json
```
Extract: PDB ID, heavy/light sequences, antigen chain, resolution.
Flag high-resolution structures (< 2.5Å) for benchmark priority.

**UniProt + STRING** (natural receptors):
```
GET https://rest.uniprot.org/uniprotkb/search?query=interacts_with:{id}&reviewed:true&organism_id:9606
GET https://string-db.org/api/json/interaction_partners?identifiers={id}&species=9606
```
Keep only `experimentally_determined_interaction > 400`.
For each receptor hit: flag `off_target_risk = "high"` by default (natural
receptors are promiscuous). Off-target module will refine this.

**Cell type annotation** (enriches the UI for v2):
For each natural receptor binder: query `BioGRID` or UniProt tissue expression
to annotate which cell types express the binder. This tells the researcher
"SIRPα is expressed on macrophages and dendritic cells" — critical for
synNotch circuit design (you need to know who's doing the signaling).

### Step 3 — `construct.py`
For antibody binders:
1. Run `anarci` locally to number VH and VL, extract variable domains only
2. Try both orientations: VH-linker-VL and VL-linker-VH
3. Store both as separate `Construct` objects — Boltz will select the better folder
4. Default linker: `(GGGGS)₃` from `data/linkers.yaml`

For natural receptor binders:
1. Run DeepTMHMM on the receptor sequence (it may also be a membrane protein)
2. Extract ectodomain(s)
3. If multidomain: create one construct per domain + one construct with all ectodomains
   concatenated (some receptors need multiple domains for binding)

### Step 4 — `structure.py`
Write two-sequence FASTA (target ectodomain + construct), run Boltz:
```bash
boltz predict input.fasta \
  --out_dir {out_dir} \
  --recycling_steps 3 \
  --diffusion_samples 5
```
Parse `confidence_model_0.json` → `Structure`.

**Parallelization note:** when processing multiple constructs, run Boltz
calls in a subprocess pool. Each call is independent. See `pipeline.py`
for the `concurrent.futures.ProcessPoolExecutor` pattern.

### Step 4b — `membrane.py` (optional, `--membrane` flag)

Only for the top N constructs by ipTM (default N=3, configurable).

Uses COMPLIP (Boltz extension for membrane proteins):
```bash
boltz predict input.fasta \
  --out_dir {out_dir} \
  --membrane \
  --membrane_thickness 40   # Å, typical mammalian plasma membrane
```

IMPORTANT: COMPLIP adds the membrane context but the binder itself is extracellular.
What it actually simulates is the full synNotch construct (binder + Notch TMD) in
the bilayer, engaging the target on an opposing membrane. This requires:
- The Notch TMD sequence (stored in `data/notch_backbone.yaml`)
- The target sequence INCLUDING its TM anchor (the antigen is also membrane-bound)

The input FASTA for membrane mode is therefore 3 chains:
```
>notch_with_binder|chain_A   ← binder + Notch NRR/TMD (no intracellular)
>target_full|chain_B         ← full target including TM anchor
>membrane|chain_X            ← COMPLIP special token (see docs/membrane_simulation.md)
```

### Step 5 — `offtarget.py`

Run only for `binder.type == "receptor"` (natural receptors have inherent off-target risk).
For antibody binders: run only if `--check-offtargets` flag is passed (slower).

1. BLAST binder sequence against UniProt human reviewed:
   ```
   POST https://blast.ncbi.nlm.nih.gov/blast/Blast.cgi (async, poll for results)
   ```
   Or use local BLAST database for speed (see `docs/off_targets.md`).

2. Collect hits with `e_value < 1e-5` and sequence identity > 40%.

3. For top 5 hits by sequence identity: run Boltz docking against the off-target.

4. Classify risk:
   - `iptm > 0.75` → high risk
   - `iptm 0.6-0.75` → medium risk
   - `iptm < 0.6` → low risk

5. Store in `OffTargetResult` list, write to `offtargets.csv`.

### Step 6 — `benchmark.py`
Only runs when `binder.pdb_complex_id` is set.
Fetch PDB, align predicted interface vs crystal, compute Cα iRMSD at 8Å cutoff.
See `docs/benchmarking.md` for full protocol.

---

## CLI interface

```bash
# Full run (all steps except membrane + offtargets)
synbind run --target Q08722

# With membrane simulation for top candidates
synbind run --target Q08722 --membrane --membrane-topk 3

# With off-target prediction for all binder types
synbind run --target Q08722 --check-offtargets

# Full run, everything on
synbind run --target Q08722 --membrane --check-offtargets --diffusion-samples 5

# Search only (no structure prediction)
synbind search --target Q08722

# Benchmark against known PDB structures
synbind benchmark --set data/benchmark_set.csv

# Show what's cached for a target
synbind cache --target Q08722

# Clear cache for a target
synbind cache --target Q08722 --clear
```

---

## v2 GUI plan (do not implement in v1, but design for it)

v2 is a Flask + React web app for wet-lab users. To make v1 → v2 smooth:

- **Keep all business logic in `synbind/` modules** (not in `cli.py`)
- `cli.py` only calls functions from `pipeline.py` and formats output
- `pipeline.py` orchestrates steps and returns typed results
- No `print()` in modules — use `structlog` with levels; the web app can
  subscribe to a log stream
- `results/` directory structure is already web-friendly (JSON + PDB files)
- The final `candidates.csv` maps 1:1 to the table the GUI will show

When building v2, the pattern is:
```
FastAPI route → pipeline.py functions → same results/ output → React reads JSON
```

The v2 GUI must be usable by a wet-lab biologist with no bioinformatics background:
- Target entry: autocomplete from UniProt search (not raw UniProt IDs)
- Results: visual table with explanatory tooltips on every metric
- PDB viewer: embedded Mol* or NGL viewer for structure visualization
- One-click: "Export top 3 constructs for synthesis"

---

## Environment and dependencies

```toml
[project]
name = "synbind"
requires-python = ">=3.11"
dependencies = [
    "httpx>=0.27",
    "biopython>=1.83",
    "pandas>=2.2",
    "typer>=0.12",
    "rich>=13",
    "structlog>=24",
    "boltz>=0.4",
    "anarci>=1.3",
    "pydeeptyhmm",
    "pyyaml",
    "biopython>=1.83",
]

[project.optional-dependencies]
dev = ["pytest", "pytest-httpx", "ruff", "mypy"]
web = ["fastapi", "uvicorn", "jinja2"]
```

BLAST: use local `ncbi-blast+` via subprocess OR the NCBI web API.
Local is faster and avoids rate limits. Install separately:
```bash
sudo apt install ncbi-blast+
```

---

## Coding conventions

- Type hints on all public functions. Return typed dataclasses, never raw dicts.
- `structlog.get_logger()` at module top. Use `log.info()`, `log.warning()` etc.
  No bare `print()` except `cli.py` via `rich`.
- All HTTP calls through `utils/http.py` shared client (retry + timeout configured).
- Disk cache in `results/{run_id}/cache/`. Check before every network call.
- `pipeline.py` is the only module allowed to call multiple step modules in sequence.
  Step modules do not import each other.
- Tests mock all HTTP with `pytest-httpx`. Never hit live APIs in tests.
- `ruff` for linting + formatting. Run before committing.

---

## What is out of scope for v1 (do not implement)

These are in `IDEAS.md` for future reference. Do not add them to v1:

- De novo binder design (RFdiffusion + ProteinMPNN)
- ESM Atlas integration for novel binder discovery
- Affinity prediction (Kd)
- Molecular dynamics
- Multi-antigen logic gating (AND/OR gates in synNotch circuits)
- PROTAC or degrader design variants
- Anything requiring a GPU in the critical path (GPU should be optional for Boltz)

---

## Benchmark set (starter, verify PDB IDs at rcsb.org before committing)

| Target | UniProt | Binder | Type | PDB |
|--------|---------|--------|------|-----|
| CD47 | Q08722 | Hu5F9 | antibody | 6NZO |
| PD-1 | Q15116 | Pembrolizumab | antibody | 5JXE |
| PD-L1 | Q9NZQ7 | Atezolizumab | antibody | 5XXY |
| HER2 | P04626 | Trastuzumab | antibody | 1N8Z |
| EGFR | P00533 | Cetuximab | antibody | 1YY9 |
| TNFα | P01375 | Adalimumab | antibody | 3WD5 |
| VEGF | P15692 | Bevacizumab | antibody | 2FJG |
| CD19 | P15391 | FMC63 scFv | antibody | 6AL5 |
| CD47 | Q08722 | SIRPα D1 | receptor | 6BKM |
| PD-1 | Q15116 | PD-L1 ecto | receptor | 4ZQK |

---

## Status at project start

Nothing implemented. Recommended start order:
1. `models.py` — define all dataclasses first, nothing else imports until this works
2. `utils/http.py` — shared client
3. `target.py` + `cli.py` stub — `synbind search --target Q08722` prints Target
4. `binders.py` — adds binder list to output
5. `construct.py` — scFv conversion
6. `structure.py` — Boltz integration
7. `benchmark.py` — RMSD pipeline
8. `offtarget.py` — BLAST + off-target docking
9. `membrane.py` — COMPLIP integration
10. `report.py` — final CSV assembly
