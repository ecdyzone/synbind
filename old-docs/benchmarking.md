# Benchmarking

How to validate that the pipeline makes accurate predictions, how to run
the benchmark, interpret the results, and expand the benchmark set over time.

---

## Why benchmark

The pipeline's main value is predicting binding for constructs that have
no experimental data. Before trusting those predictions, you need to know
how accurate it is on cases where the answer is already known.

The benchmark runs the full pipeline on complexes with existing crystal
structures in the PDB, then compares predictions to crystals via iRMSD.
The result is a calibrated expectation: "on this class of targets, the
pipeline gets within X Å of the crystal structure Y% of the time."

Without a passing benchmark, ipTM scores for novel constructs are just
numbers. With it, they have an empirical interpretation.

---

## Benchmark set

Located at `data/benchmark_set.csv`. Committed to the repo.

### Format

```csv
target_uniprot,target_name,pdb_complex_id,binder_name,binder_type,resolution_A,notes
Q08722,CD47,6NZO,Hu5F9,antibody,3.1,anti-phagocytosis; CAR-T context
Q15116,PD-1,5JXE,Pembrolizumab,antibody,2.9,checkpoint inhibitor
P04626,HER2,1N8Z,Trastuzumab,antibody,2.5,breast cancer
P00533,EGFR,1YY9,Cetuximab,antibody,3.3,colorectal cancer
P01375,TNFα,3WD5,Adalimumab,antibody,3.0,autoimmune
P15692,VEGF,2FJG,Bevacizumab,antibody,2.4,angiogenesis
P15391,CD19,6AL5,FMC63,antibody,3.2,B-cell malignancies
P78552,SIRPα,6NZO,CD47 ectodomain,receptor,3.1,natural receptor pair (inverse)
```

### Criteria for adding a complex

A complex is valid for the benchmark if:
1. The PDB entry resolution is ≤ 3.5Å (lower is better)
2. The antibody or receptor sequence is available in SAbDab or UniProt
3. The antigen's UniProt accession is annotated in the PDB entry
4. The antigen is a human surface protein (relevant to synNotch context)
5. The complex involves an extracellular domain interaction (not intracellular)

Verify PDB IDs at https://www.rcsb.org before committing — entries are
occasionally superseded or withdrawn.

---

## Running the benchmark

```bash
# Full benchmark run (all complexes in benchmark_set.csv)
synbind benchmark --set data/benchmark_set.csv

# Single complex (for debugging or adding new entries)
synbind benchmark --pdb 6NZO --target Q08722

# Resume interrupted run
synbind benchmark --set data/benchmark_set.csv --resume

# Use more diffusion samples for higher accuracy
synbind benchmark --set data/benchmark_set.csv --diffusion-samples 5
```

Output: `results/benchmark_{YYYYMMDD}/`

```
results/benchmark_20250722/
├── benchmark_rmsd.csv          ← one row per complex
├── benchmark_summary.txt       ← median, % < 2Å, distribution
└── {pdb_id}/
    ├── construct.fasta
    ├── complex_predicted.pdb
    └── scores.json
```

---

## Output files

### `benchmark_rmsd.csv`

```csv
pdb_id,binder_name,construct_type,iptm,mean_plddt,rmsd_angstrom,n_interface_residues,n_aligned_atoms,resolution_A
6NZO,Hu5F9,scFv_VH_VL,0.91,87.2,1.4,42,42,3.1
5JXE,Pembrolizumab,scFv_VH_VL,0.88,84.1,1.8,38,38,2.9
1N8Z,Trastuzumab,scFv_VH_VL,0.86,82.7,2.1,31,31,2.5
```

### `benchmark_summary.txt`

```
synbind benchmark — 2025-07-22
===============================
Complexes evaluated:  8
Median iRMSD:         1.9 Å
Mean iRMSD:           2.2 Å
Fraction < 2.0 Å:     62%
Fraction < 2.5 Å:     75%

Boltz settings: recycling_steps=3, diffusion_samples=5
Pipeline version: synbind 0.1.0
```

---

## Interpreting results

### What constitutes a passing benchmark

For the pipeline to be considered validated for a given target class:
- **Median iRMSD < 2.5 Å** over the benchmark set
- **At least 60% of complexes with iRMSD < 2.0 Å**

If the benchmark passes, ipTM scores for novel constructs against similar
targets can be interpreted using the thresholds in `docs/metrics.md`.

If the benchmark fails (median iRMSD > 3.5 Å), do not use the pipeline
predictions for that target class without further investigation. Common causes
are listed below.

### Investigating high-RMSD complexes

**Check whether the crystal has multiple bound antibodies:**
Some PDB entries contain two different antibodies bound simultaneously.
The pipeline will try to match the construct to one of them — confirm
which one via sequence alignment.

**Check the resolution of the crystal:**
Complexes with resolution > 3.0 Å have inherent coordinate uncertainty
of ~0.5 Å. High iRMSD may reflect crystal quality, not pipeline failure.

**Check for alternative epitopes:**
The antibody may bind a different epitope in solution than in the crystal
(crystal packing can force a non-native pose). This is not a pipeline bug.

**Check CDR3 length:**
Very long CDR3 loops (> 12 residues, IMGT) are inherently flexible and
difficult to predict. High RMSD for these is expected. Focus on whether
the framework and CDR1/CDR2 contacts are correct.

**Visually inspect in PyMOL:**
```bash
pymol results/benchmark_20250722/6NZO/complex_predicted.pdb data/pdb_cache/6NZO.pdb
```
In PyMOL:
```python
align complex_predicted, 6NZO
color cyan, complex_predicted
color orange, 6NZO
show sticks, (resn * and chain B within 5 of chain A)   # interface residues
zoom
```

---

## Adding a new complex to the benchmark set

### Step 1 — Verify the PDB entry

```bash
synbind verify-pdb --pdb {PDB_ID} --target-uniprot {UNIPROT_ID}
```

This command:
1. Fetches the PDB file from RCSB
2. Runs sequence alignment of all chains against the target
3. Confirms the target is present and identifies the antigen chain ID
4. Reports resolution and method (must be X-ray crystallography or cryo-EM)

### Step 2 — Run the new complex

```bash
synbind benchmark --pdb {PDB_ID} --target {UNIPROT_ID}
```

Check the output iRMSD. If < 3.0 Å, it is a good addition to the set.

### Step 3 — Add to CSV and commit

Add the row to `data/benchmark_set.csv` and copy the PDB file to
`data/pdb_cache/{PDB_ID}.pdb`. Commit both.

---

## Benchmark expansion strategy

The current benchmark set is antibody-heavy. As the pipeline matures,
expand in these directions:

### Natural receptor binders

Add complexes where the binder is a receptor ectodomain rather than an
antibody. Good starting candidates:

| Complex | PDB | Notes |
|---------|-----|-------|
| CD47 + SIRPα | 2WNG | Natural receptor pair; key synNotch target |
| PD-1 + PD-L1 | 4ZQK | Checkpoint pair |
| CD28 + CD80 | 1YJD | T-cell co-stimulation |
| CTLA-4 + CD86 | 1I8L | Immune checkpoint |

### ESM Atlas hit validation

Once the pipeline discovers novel binders via ESM Atlas (`--esmatlas`),
any that get experimentally validated become the most valuable benchmark
entries — they directly validate the Atlas-based discovery workflow.

Track these in a separate `data/atlas_validated.csv` file with the same
format as `benchmark_set.csv` plus a `date_validated` and `validation_method`
column (SPR, ELISA, co-IP, etc.).

### Membrane-context benchmark

When membrane mode (`--membrane`) matures, a separate benchmark set
should be created with complexes from lipid nanodisc or detergent-solubilized
structures. These are rare — maintain as a separate CSV when it becomes relevant.

---

## Reporting the benchmark in a paper

The benchmark section of a methods section should include:

1. The number of complexes tested
2. The PDB IDs and their resolution
3. The Boltz settings used (`recycling_steps`, `diffusion_samples`)
4. The metric used (iRMSD, cutoff distance, atom type)
5. Median and distribution of iRMSD values
6. A figure: scatter plot of predicted ipTM vs observed iRMSD

The scatter plot is the most informative figure — it shows whether ipTM
is a reliable proxy for accuracy. If there is a strong negative correlation
(high ipTM → low iRMSD), the pipeline is well-calibrated.

```python
import pandas as pd
import matplotlib.pyplot as plt

df = pd.read_csv("results/benchmark_20250722/benchmark_rmsd.csv")

fig, ax = plt.subplots(figsize=(6, 5))
ax.scatter(df["iptm"], df["rmsd_angstrom"], c="steelblue", s=60, alpha=0.8)
ax.axhline(2.0, color="gray", linestyle="--", label="2 Å threshold")
ax.set_xlabel("Boltz ipTM")
ax.set_ylabel("iRMSD vs crystal (Å)")
ax.set_title("synbind benchmark: predicted confidence vs accuracy")
ax.legend()
plt.tight_layout()
plt.savefig("figures/benchmark_scatter.pdf")
```
