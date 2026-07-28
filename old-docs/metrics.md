# Metrics

What every number in the pipeline output means, where it comes from,
and how to use it to make decisions about which constructs to prioritize.

---

## ipTM — interface predicted TM-score

**Source:** Boltz `confidence_model_0.json → iptm`
**Range:** 0 to 1
**The primary ranking metric.**

ipTM is Boltz's confidence that the predicted binding interface is
geometrically correct. It is computed from the model's internal pairwise
distance predictions between residues across the two chains, compared to
what a correctly docked complex would look like.

A high ipTM does not mean the proteins definitely bind. It means Boltz
is confident about the relative positioning of the two chains at the interface.
Empirically, high ipTM correlates well with experimentally validated binders
in antibody–antigen benchmarks.

| ipTM | Interpretation | Action |
|------|----------------|--------|
| > 0.90 | Excellent — very confident binding pose | High priority; validate experimentally |
| 0.80–0.90 | Good — strong predicted interaction | Standard priority |
| 0.70–0.80 | Moderate — plausible but uncertain | Lower priority; check RMSD if available |
| 0.60–0.70 | Weak — treat with caution | Deprioritize unless no better candidates |
| < 0.60 | Poor | Exclude from candidates list |

**synNotch-specific caveat:** synNotch requires cell-surface contact to trigger
cleavage. A binder with very high ipTM (> 0.95) may indicate an extremely tight
interaction that could cause tonic signaling (receptor firing without antigen
contact). Flag constructs with ipTM > 0.95 for extra scrutiny, not just celebration.

---

## pTM — predicted TM-score

**Source:** Boltz `confidence_model_0.json → ptm`
**Range:** 0 to 1
**Secondary metric; use as a sanity check, not for ranking.**

pTM measures overall structural quality of the complex prediction — both
chains combined, including regions far from the interface.

If `ptm < 0.5`, the overall structure prediction failed and `iptm` is
unreliable. Flag and skip. Otherwise, do not use pTM to compare candidates —
two constructs with the same ipTM but different pTM are not meaningfully
different in binding quality.

---

## mean_pLDDT — mean per-residue confidence

**Source:** Boltz `confidence_model_0.json → plddt` (array, mean taken)
**Range:** 0 to 100
**Use for:** identifying poorly predicted constructs, not for ranking binders.

pLDDT is per-residue structural confidence. We report the mean over all
residues in both chains (target + construct combined).

| mean_pLDDT | Interpretation |
|------------|----------------|
| > 90 | Very high confidence — well-folded prediction |
| 70–90 | Confident — reliable for most purposes |
| 50–70 | Low — flexible regions or uncertain fold |
| < 50 | Very low — likely disordered or wrong; investigate |

**CDR loop exception:** Antibody CDR3 loops (the primary antigen-contact loops,
IMGT positions 95–102) are intrinsically flexible and routinely have pLDDT
of 50–65 even in correct predictions. Do not penalize a construct for low
CDR3 pLDDT. Instead, check whether the CDR3 residues are at the interface
in the predicted structure — if yes, the low confidence is expected.

**How to compute pLDDT for the interface only** (more informative than mean):
```python
from Bio.PDB import PDBParser, NeighborSearch
parser = PDBParser(QUIET=True)
structure = parser.get_structure("complex", pdb_path)
# B-factor column in Boltz PDB output encodes per-residue pLDDT
for residue in structure[0]["B"]:   # chain B = construct
    plddt = residue["CA"].get_bfactor()
```

---

## iRMSD — interface RMSD vs crystal structure

**Source:** `steps/s5_benchmark.py`, Biopython `Superimposer`
**Units:** Ångströms (Å)
**Only available when `binder.pdb_complex_id` is not None.**

iRMSD measures how closely the predicted binding pose matches a known
crystal structure of the same complex. It is the gold-standard validation
metric — when available, it overrides ipTM as the primary confidence signal.

Computed over Cα atoms of interface residues only (residues within 8Å of
the partner chain in the predicted structure). We do not use full-chain RMSD
because the relative orientation of the two proteins in space is arbitrary
and uninformative.

| iRMSD (Å) | Interpretation |
|-----------|----------------|
| < 1.5 | Excellent — near-crystallographic accuracy |
| 1.5–2.5 | Good — binding mode correctly captured |
| 2.5–4.0 | Acceptable — rough pose; binding site correct |
| > 4.0 | Poor — incorrect binding pose |

**How to use iRMSD together with ipTM:**

```
ipTM high + iRMSD low  → highest confidence; use as benchmark positive
ipTM high + iRMSD high → Boltz is confident but wrong; investigate why
                          (alternative epitope? crystal packing artifact?)
ipTM low  + iRMSD low  → Boltz found the right pose but was uncertain
                          (unusual; usually means high structural flexibility)
ipTM low  + iRMSD high → poor prediction; deprioritize
```

---

## n_interface_residues

**Source:** `steps/s5_benchmark.py`
**Units:** count
**Use as:** quality control for iRMSD, not as a ranking signal.

Number of residues in the predicted interface (within 8Å of the partner chain).
A very small interface (< 10 residues) may produce a misleadingly low iRMSD
by chance — chance alignment of a few atoms can give 0.8Å RMSD for completely
wrong poses if only 5 atoms are aligned. Flag these.

Normal antibody–antigen interfaces: 15–40 residues.
Large receptor–ligand interfaces: 30–80 residues.

---

## similarity_score (ESM Atlas hits)

**Source:** ESM Atlas API `similar_proteins[i].similarity_score`
**Range:** 0 to 1 (cosine similarity in SAE feature space)
**Only present for binders with `source="esmatlas"`.**

This is not sequence identity. It is cosine similarity between the
sparse autoencoder feature vectors of the query (target extracellular sequence)
and the hit. Two proteins can have 0% sequence identity and score 0.85 here
if they share the same functional features in the ESM model's learned
representation.

| Similarity | Interpretation |
|------------|----------------|
| > 0.90 | Functionally very close; likely known binding partner |
| 0.80–0.90 | Strong functional similarity; strong candidate |
| 0.75–0.80 | Moderate; worth investigating |
| < 0.75 | Filtered out by default (`min_similarity=0.75`) |

Do not confuse this with ipTM. Similarity score tells you how functionally
related the hit is to the target (step 2 output). ipTM tells you how
confidently it docks to the target (step 4 output). Both are needed.

---

## sequence_identity_to_target (off-target screen)

**Source:** mmseqs2 in `steps/s6_offtarget.py`
**Units:** percentage (0–100)
**Used for:** stage 1 off-target flagging only.

Sequence identity between the construct and a surface proteome protein.
Any hit above 30% is flagged as a potential cross-reactor without running Boltz.

This is a conservative threshold — 30% identity often means structural
similarity, especially in the immunoglobulin fold. False positives are
acceptable here; false negatives (missing a real off-target) are not.

---

## n_offtargets

**Source:** `steps/s6_offtarget.py`, count of `OffTargetResult` entries
**Units:** count
**Used in:** composite ranking score

The number of human surface proteins that the construct is predicted to
bind (`iptm > 0.65`) or that share significant sequence similarity (>30%)
with the target. Lower is better for synNotch applications.

A construct with 0 predicted off-targets that cleared both the mmseqs2 and
Boltz screens is a **clean binder** — high value for synNotch safety.

---

## rank_score — composite ranking metric

**Source:** `report.py`
**Formula:**

```python
rank_score = iptm - (0.3 * min(n_offtargets, 10) / 10)
```

The `0.3` weight is a tunable parameter in `report.py`. It means:
a construct with 10+ off-targets is penalized by 0.3 points on the ipTM scale.
A construct with `iptm=0.88` and 0 off-targets (rank_score=0.88) outranks
one with `iptm=0.92` and 10+ off-targets (rank_score=0.62).

This reflects the synNotch design requirement that specificity is at
least as important as binding strength.

**Adjust this weight** based on application context:
- Tumor-specific targets with no healthy-tissue expression → `0.1` (off-targets matter less)
- Widely-expressed targets → `0.5` or higher (off-target specificity critical)

---

## Quick decision table for wet lab prioritization

| Condition | Recommended action |
|-----------|-------------------|
| `rank_score > 0.80`, `n_offtargets = 0` | Top priority — validate experimentally |
| `rank_score > 0.80`, `n_offtargets > 0` | Validate off-targets first; check tissue expression of off-target proteins |
| `iRMSD < 2.0` (benchmark complex) | Pipeline validated for this target class; trust ipTM predictions |
| `iRMSD > 4.0` (benchmark complex) | Investigate before trusting — may be wrong epitope |
| `mean_pLDDT < 50` | Poor structure prediction; check construct sequence; consider alternate orientation |
| `construct_type = scFv_VL_VH` ranked higher than `scFv_VH_VL` | Both orientations built; VL-VH works better for this antibody — expected |
| All candidates `iptm < 0.70` | No good antibody found in SAbDab; switch to `--esmatlas` for novel discovery |
