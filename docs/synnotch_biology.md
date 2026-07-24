# SynNotch biology

Essential reading before working on any module that touches construct design,
membrane simulation, or the interpretation of output metrics.

---

## What is synNotch?

SynNotch (synthetic Notch) was developed by the Lim lab (UCSF) and published in
Nature in 2016. It repurposes the Notch receptor — a cell-cell signaling protein
conserved across metazoa — as a programmable sensor for any antigen.

The key insight: the Notch receptor's antigen-binding domain and its
transcriptional output are modular and can be independently swapped.

---

## Natural Notch signaling (simplified)

1. Notch is displayed on the surface of the **receiving cell** as a type I
   transmembrane protein
2. Delta-like or Jagged ligands on an adjacent **sending cell** bind Notch's
   extracellular domain
3. Mechanical force from cell-cell contact opens the Notch NRR (negative
   regulatory region), exposing a cleavage site
4. ADAM10/TACE protease cleaves the extracellular region (S2 cleavage)
5. γ-Secretase cleaves within the TMD (S3 cleavage)
6. The released intracellular domain (NICD) translocates to the nucleus and
   activates Notch target genes

**Critical: cleavage requires mechanical force from a ligand on a neighboring cell.**
This is why synNotch only fires in trans (cell-to-cell) — not when the antigen
is soluble in the media. This is a feature, not a bug.

---

## SynNotch architecture

```
EXTRACELLULAR                                    INTRACELLULAR
─────────────────────────────────────────────────────────────
[SP]──[BINDER]──[NRR/HD]──||──[TMD]──||──[RAM]──[ANK]──[TA]
       ↑ WHAT              ↑ HINGE    ↑          ↑ OUTPUT
    synbind designs       region    membrane   transactivator
                                              (e.g. GAL4-VP64,
                                               tTA, dCas9-VP64)
```

- **SP** — signal peptide (for membrane targeting)
- **BINDER** — the antigen-recognition domain. This is what synbind designs.
- **NRR/HD** — Notch negative regulatory region + heterodimerization domain.
  These are preserved from natural Notch and mediate the force-induced
  conformational change that exposes the S2 cleavage site.
- **TMD** — transmembrane domain (from Notch or synthetic)
- **RAM/ANK** — RAM domain and ankyrin repeats. Bind CSL/RBPJ upon nuclear entry.
- **TA** — transactivator. Drives expression of any gene of interest.

---

## Requirements for the binder domain

Understanding these shapes every decision in synbind:

### 1. Single-chain
The binder must be a single polypeptide — it is fused directly to the Notch
core as one continuous chain. This rules out full IgG (two chains, held by
disulfide bonds). Acceptable formats:
- scFv (VH + linker + VL) — ✅ most common
- Nanobody (VHH) — ✅ single domain, smaller, often better folder
- Natural receptor ectodomain — ✅ if single-chain after trimming TM helix

### 2. Extracellular fold under membrane constraint
The binder is not floating freely — it is tethered to the Notch TMD and
displayed at a fixed height above the membrane surface (~10–15 nm for a typical
scFv). This constrains the geometry of antigen engagement.

The antigen is also membrane-anchored on the opposing cell. The binder must
be able to reach across the synaptic cleft (~15–20 nm) and bind the antigen
at its specific epitope.

**Implication for structure prediction:** solution-phase Boltz docking is
a good first approximation. Membrane-mode (COMPLIP) is a better approximation
but slower. For the membrane run, we use the full synNotch ectodomain
(binder + NRR linker), not just the isolated binder.

### 3. Affinity in the right range
- Kd too LOW (< 0.1 nM, very tight): risk of tonic (ligand-independent)
  signaling, and the receptor may not be efficiently transported to the surface
- Kd too HIGH (> 1 µM, very loose): signaling will be weak or absent
- **Target range: ~1–100 nM** for robust synNotch signaling

synbind does not directly predict Kd in v1 (see ideas.md #5 for affinity
estimation plans). ipTM is used as a proxy for binding quality.

### 4. Low off-target binding
An scFv derived from a well-characterized mAb typically has high specificity.
A natural receptor ectodomain (e.g., SIRPα) often has paralogous binding
partners (SIRP-β1, SIRP-γ) — these are off-targets that would cause
the synNotch to fire in the presence of unintended ligands.

---

## The membrane geometry problem

This is worth emphasizing because it distinguishes synbind from generic
antibody-antigen docking tools.

When two cells are in contact, the membranes are separated by a synaptic
cleft. The synNotch on the receiving cell must engage an antigen (e.g., CD47)
on the sending cell surface in **trans**. The geometry is:

```
RECEIVING CELL MEMBRANE
────────────────────────────────────────────────
[TMD]──[NRR]──[BINDER] ──────engages──────▶ [ANTIGEN]──[TM]
                                              (on opposing cell)
────────────────────────────────────────────────
SENDING CELL MEMBRANE   ← ~15–20 nm gap
```

A binder that works in this geometry must:
- Protrude far enough from the membrane to reach across the cleft
- Have its binding site oriented outward (not buried against the Notch stalk)
- Not sterically clash with the NRR domain adjacent to it

The COMPLIP membrane simulation captures this geometry better than
solution-phase docking, which models the two proteins as free-floating.

---

## Notch backbone sequences

Stored in `data/notch_backbone.yaml`. These are the fixed Notch components
that are NOT designed by synbind — they are used only in the membrane
simulation step (step 4b) to build the full construct model.

Sequences are from human Notch1 (UniProt P46531):
- `nrr_hd`: residues 1447–1735 (negative regulatory region + HD domain)
- `tmd`: residues 1756–1800 (transmembrane domain)
- `ram`: residues 1851–1891 (RAM domain)
- `ank`: residues 1892–2183 (ankyrin repeats)

The transactivator (GAL4-VP64, tTA, etc.) is not included — it is intracellular
and irrelevant to the extracellular docking geometry.

---

## Further reading

- Morsut et al. (2016) Nature — original synNotch paper (UCSF Lim lab)
- Roybal et al. (2016) Cell — synNotch for therapeutic applications
- Toda et al. (2018) Science — multi-input synNotch logic gates
- Zhu et al. (2020) Nature Chemical Biology — synNotch in vivo
