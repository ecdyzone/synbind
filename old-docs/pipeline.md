# Pipeline

Detailed logic for each step: what goes in, what comes out, how failures
are handled, and what to watch out for.

The six steps map to the six modules in `synbind/steps/`. Each step receives
typed dataclass objects and returns typed dataclass objects — never raw dicts.
See `models.py` for the full type definitions.

---

## Overview

```
UniProt ID
    │
    ▼
[S1] prepare_target()
    → Target (extracellular_sequence, tm_segments)
    │
    ▼
[S2] find_binders()
    → list[Binder]   (SAbDab + UniProt/STRING + ESM Atlas)
    │
    ▼
[S3] build_construct()          ← one Binder → one or two Constructs
    → list[Construct]  (scFv VH-VL, scFv VL-VH, or ectodomain)
    │
    ▼
[S4] predict_structure()        ← target + construct → Boltz
    → Structure  (pdb_path, iptm, ptm, mean_plddt)
    │
    ├──► [S5] benchmark_against_crystal()   ← only if pdb_complex_id exists
    │         → dict  (rmsd, n_interface_residues)
    │
    └──► [S6] screen_offtargets()           ← only if --no-offtarget not set
              → list[OffTargetResult]

[report] assemble candidates.csv + offtargets.csv
```

---

## Step 1 — Target preparation (`steps/s1_target.py`)

**Function:** `prepare_target(uniprot_id: str) -> Target`

### What it does

1. **Fetch from UniProt** (`utils/uniprot.py`):
   - `GET https://rest.uniprot.org/uniprotkb/{id}.fasta` → raw sequence
   - `GET https://rest.uniprot.org/uniprotkb/{id}.json` → metadata + features
   - Cache both to `results/{run_id}/cache/uniprot_{id}_{fasta|json}`

2. **Run DeepTMHMM** on the full sequence:
   ```bash
   deeptmhmm --fasta /tmp/target.fasta --output /tmp/tmhmm_out/
   ```
   Parse the output GFF3. Collect all segments annotated as `outside`
   (extracellular). Pick the longest one.

3. **Signal peptide:** if UniProt JSON contains a `SIGNAL` feature,
   record `signal_peptide_end`. Exclude the signal peptide from
   `extracellular_sequence` (it is cleaved in the mature protein).

4. **Fallback:** if DeepTMHMM subprocess fails (not installed, timeout),
   fall back to UniProt `TOPO_DOM` features with `description: Extracellular`.
   Log a warning but do not crash.

5. Return `Target` with both `full_sequence` and `extracellular_sequence`.

### Edge cases

| Situation | Behaviour |
|-----------|-----------|
| UniProt ID not found | Raise `TargetNotFoundError` with clear message |
| No TM segments detected | `is_membrane_protein=False`; use mature sequence (minus signal peptide) as `extracellular_sequence`; warn |
| Protein is entirely intracellular | Warn that synNotch context may not apply; use full sequence; continue |
| Multiple extracellular segments | Use the longest; log all segments and their lengths |
| Sequence > 800 aa after extraction | Log warning — ESM Atlas has an 800 aa limit; Boltz handles longer sequences fine |

---

## Step 2 — Binder search (`steps/s2_binders.py`)

**Function:** `find_binders(target: Target, use_esmatlas: bool = False) -> list[Binder]`

Results from all sub-sources are merged and deduplicated. Deduplication:
sequences with >95% identity (computed via simple k-mer comparison or
`mmseqs2 easy-cluster` on the combined FASTA) are collapsed — keep the
one with a known `pdb_complex_id`, or the SAbDab one if both lack PDB.

### 2A — Antibodies via SAbDab

```
GET https://opig.stats.ox.ac.uk/webapps/sabdab-sabdab/search/
    ?antigen_uniprot={target.uniprot_id}
    &format=json
```

Response is a list of antibody entries. For each entry:
- Extract `pdb` (PDB complex ID), `Hchain`, `Lchain` (chain IDs in the PDB)
- Fetch the PDB file from RCSB: `GET https://files.rcsb.org/download/{pdb}.pdb`
- Extract heavy and light chain sequences from the PDB by chain ID
- Create `Binder(type="antibody", source="sabdab", pdb_complex_id=pdb, ...)`

**Note on SAbDab sequences:** the sequences in the PDB include constant regions.
Step 3 (`anarci`) will trim these to variable domains only.

### 2B — Natural receptors via UniProt + STRING

**UniProt interaction search:**
```
GET https://rest.uniprot.org/uniprotkb/search
    ?query=interacts_with:{uniprot_id} AND reviewed:true AND organism_id:9606
    &format=json&fields=accession,gene_names,sequence,go
```

**STRING confidence filter:**
```
GET https://string-db.org/api/json/interaction_partners
    ?identifiers={uniprot_id}&species=9606&limit=50
```
Keep only partners where `experimentally_determined_interaction > 400`.
Cross-reference UniProt results against STRING — keep only proteins that
appear in both (UniProt says "interacts", STRING has experimental evidence).

For each surviving partner:
- Create `Binder(type="receptor", source="uniprot", heavy_seq=None, light_seq=None, sequence=partner_seq)`
- `pdb_complex_id=None` by default (no guarantee of a co-crystal)

**Also check RCSB for co-crystals:** query RCSB search API for structures
containing both `target.uniprot_id` and the partner accession. If found,
set `pdb_complex_id` for that binder. This enables step 5 for natural receptors too.

### 2C — Novel binders via ESM Atlas (opt-in, `--esmatlas` flag)

```
GET https://biohub.ai/esm/protein/api/v1alpha1/similarity-search
    ?sequence={target.extracellular_sequence}
    &topk_results=50
    &topk_features=20
    &min_similarity=0.75
    &include_cluster_info=true
```

For each hit:
- Check `protein_accession` against UniProt to get GO terms
- Keep hits with at least one GO term matching:
  `receptor activity`, `binding`, `immune`, `cell surface`, `signaling`
- Discard hits already found by 2A or 2B (deduplicated by accession)
- Create `Binder(type="atlas_hit", source="esmatlas", similarity_score=hit.similarity_score)`

**Important:** ESM Atlas hits will almost never have a `pdb_complex_id`.
Step 5 (benchmark) is skipped for them. Step 4 (Boltz) is the only validation.

---

## Step 3 — Construct building (`steps/s3_construct.py`)

**Function:** `build_construct(binder: Binder) -> list[Construct]`

Returns a list because antibodies produce two constructs (VH-VL and VL-VH);
receptors and atlas hits produce one.

### For antibodies

1. **Extract variable domains** using `anarci` (local, no HTTP call):
   ```python
   import anarci
   numbered, _, hit_table = anarci.run_anarci(
       [(f"{binder.id}_H", binder.heavy_seq)], scheme="imgt"
   )
   ```
   Repeat for light chain. Extract the IMGT-numbered residues from
   framework 1 through framework 4 (the full variable domain, no constant region).

2. **Read linker** from `data/linkers.yaml`:
   ```yaml
   default: GGGGSGGGGSGGGGSGGGGS   # (G4S)3 — 15 residues
   long:    GGGGSGGGGSGGGGSGGGGSGGGGS  # (G4S)4 — 20 residues, for large antigens
   ```

3. **Assemble two constructs:**
   - `scFv_VH_VL`: `{VH}{linker}{VL}`
   - `scFv_VL_VH`: `{VL}{linker}{VH}`

4. **Validate each construct:**
   - Length: 220–320 aa (warn if outside range, but continue)
   - No internal stop codons (check if any `*` in translated back-translation — skip this if working purely at AA level)
   - `anarci` must successfully number both VH and VL; if not, skip that construct

5. Write FASTA to `results/{run_id}/{construct_id}/construct.fasta`

### For receptors and ESM Atlas hits

1. Run DeepTMHMM on `binder.sequence` (same logic as step 1)
2. Extract extracellular domain if `is_membrane_protein=True`
3. If not a membrane protein: use full sequence
4. Validate: length 50–600 aa

### Construct ID scheme

`{binder_id}_{construct_type}` — e.g., `sabdab_6nzo_scfv_vh_vl` or `uniprot_P78552_ecto`

---

## Step 4 — Structure prediction (`steps/s4_structure.py`)

**Function:**
```python
def predict_structure(
    target: Target,
    construct: Construct,
    with_membrane: bool = False,
    diffusion_samples: int = 1,
) -> Structure
```

### Standard mode (no membrane)

1. **Write input FASTA** (Boltz multi-chain format):
   ```
   >target|chain_A
   MGNKACLS...
   >construct|chain_B
   QVQLVQSG...
   ```

2. **Cache check:** if `results/{run_id}/{construct.id}/complex_model_0.pdb`
   exists, skip Boltz and read scores from `confidence_model_0.json` directly.

3. **Run Boltz:**
   ```bash
   boltz predict input.fasta \
     --out_dir results/{run_id}/{construct.id}/ \
     --recycling_steps 3 \
     --diffusion_samples {diffusion_samples}
   ```
   Development default: `diffusion_samples=1` (fast, less accurate).
   Final runs: `diffusion_samples=5`.

4. **Parse output:**
   ```
   out/
   ├── predictions/
   │   ├── complex_model_0.pdb        ← best model
   │   └── confidence_model_0.json    ← scores
   ```
   From JSON: extract `iptm`, `ptm`, `plddt` (array → take mean).

5. Return `Structure(pdb_path=..., iptm=..., ptm=..., mean_plddt=...)`

### Membrane mode (`--membrane`)

Before calling Boltz, preprocess the input FASTA through COMPLIP:
```bash
complip prepare input.fasta --output input_membrane.fasta --membrane-protein chain_A
```
COMPLIP adds lipid bilayer embedding tokens to the input. Pass the modified
FASTA to Boltz as usual.

Add `with_membrane=True` to the returned `Structure`.

**Runtime note:** membrane mode adds ~30–60% runtime overhead. Use only
for the top candidates after a standard run, not for initial screening.

### Runtime expectations

| Mode | CPU (per complex) | GPU A100 (per complex) |
|------|-------------------|------------------------|
| `--diffusion-samples 1` | ~20 min | ~2 min |
| `--diffusion-samples 5` | ~90 min | ~8 min |
| `--membrane --diffusion-samples 5` | ~2.5 h | ~12 min |

---

## Step 5 — Benchmark (`steps/s5_benchmark.py`)

**Function:**
```python
def benchmark_against_crystal(structure: Structure, pdb_complex_id: str) -> dict
```

Only called when `binder.pdb_complex_id is not None`.

1. **Fetch crystal structure** from RCSB (with cache):
   `GET https://files.rcsb.org/download/{pdb_complex_id}.pdb`
   Cache to `data/pdb_cache/{pdb_complex_id}.pdb`

2. **Identify chains** in the crystal by sequence alignment:
   - BLAST the construct sequence against all chains in the crystal PDB
   - BLAST the target extracellular sequence against all chains
   - Assign the best-matching chains as `binder_chain` and `target_chain`

3. **Find interface residues** in the predicted structure:
   - For each residue in chain B (construct), check if any atom is within 8Å
     of any atom in chain A (target)
   - Collect these as `interface_residues_construct`
   - Repeat for chain A residues near chain B → `interface_residues_target`

4. **Superimpose** using `Biopython.PDB.Superimposer`:
   - Fixed atoms: Cα of `interface_residues_target` in crystal
   - Moving atoms: corresponding Cα in predicted structure
   - After superimposition, compute RMSD over the same interface atoms

5. **Return:**
   ```python
   {
       "rmsd": float,                    # Ångströms
       "n_interface_residues": int,
       "n_aligned_atoms": int,
       "pdb_complex_id": str,
   }
   ```

6. **Append** one row to `results/{run_id}/benchmark_rmsd.csv`

---

## Step 6 — Off-target screening (`steps/s6_offtarget.py`)

**Function:**
```python
def screen_offtargets(
    construct: Construct,
    top_n: int = 20,
    iptm_threshold: float = 0.65,
) -> list[OffTargetResult]
```

### Why two stages

The human surface proteome has ~5,000 reviewed membrane proteins.
Running Boltz on all of them per construct is infeasible (days of compute).
Two fast pre-filters reduce this to a tractable set.

### Stage 1 — Sequence similarity (mmseqs2)

```bash
mmseqs2 easy-search construct.fasta surface_proteome.fasta \
  results/offtarget_hits.tsv /tmp/mmseqs_tmp \
  --min-seq-id 0.30 --cov-mode 0 -c 0.5
```

Any hit with sequence identity >30% to the target is a **structural analog** —
flag it immediately as a likely cross-reactor without running Boltz.
These are reported in `offtargets.csv` with `stage=sequence_similarity`.

### Stage 2 — ESM Atlas pre-filter

Run `similarity-search` on the construct sequence (not the target):
```
GET /similarity-search?sequence={construct.sequence}&topk_results=50&min_similarity=0.65
```

Collect the returned accessions. These are proteins that the ESM model
considers functionally similar to the construct's binding surface — higher
prior probability of cross-reaction. Run Boltz on this set (up to 50 proteins).

### Stage 3 — Boltz docking

For each protein from stage 2 (that was not already flagged in stage 1):
- Run `predict_structure(target=offtarget_protein, construct=construct, diffusion_samples=1)`
- If `iptm > iptm_threshold`: create `OffTargetResult` and add to the list

**Total Boltz calls per construct for off-target:** up to 50.
At ~20 min/call on CPU this is ~17 hours; on GPU ~2 hours.
Run off-target screening as a background job or on a GPU node.

---

## Final report (`pipeline.py` → `report.py`)

### `candidates.csv`

One row per construct, sorted by `iptm` descending:

| Column | Source |
|--------|--------|
| `construct_id` | step 3 |
| `binder_name` | step 2 |
| `binder_source` | step 2 |
| `construct_type` | step 3 |
| `construct_length` | step 3 |
| `iptm` | step 4 |
| `ptm` | step 4 |
| `mean_plddt` | step 4 |
| `with_membrane` | step 4 |
| `pdb_complex_id` | step 2 |
| `rmsd_angstrom` | step 5 (null if no crystal) |
| `n_offtargets` | step 6 count |
| `pdb_path` | step 4 |

### `offtargets.csv`

One row per off-target hit per construct:

| Column | Source |
|--------|--------|
| `construct_id` | |
| `offtarget_uniprot` | step 6 |
| `offtarget_name` | step 6 |
| `detection_stage` | `sequence_similarity` or `boltz` |
| `iptm` | step 6 (null for stage 1 hits) |
| `sequence_identity_to_target` | stage 1 |

### Prioritization heuristic

Candidates are ranked by a composite score:
```
rank_score = iptm - (0.3 * normalized_n_offtargets)
```
A construct with `iptm=0.88` and 3 off-targets scores lower than one with
`iptm=0.85` and 0 off-targets. This is intentional: specificity is
as important as affinity in the synNotch context.
