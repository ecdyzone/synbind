# Receptor biology

Essential reading before working on construct assembly, validation rules, or the
interpretation of any output.

---

## Synthetic receptors, generally

A synthetic receptor is an engineered protein that lets a cell detect a specific
ligand and respond by expressing genes the designer chooses. The general architecture
is modular:

```
EXTRACELLULAR                                     INTRACELLULAR
──────────────────────────────────────────────────────────────
[SP]──[BINDER]──[CORE]──||──[TMD]──||──[OUTPUT MODULE]
       ↑                    ↑                ↑
   synbind designs      membrane        transactivator
                                        (e.g. GAL4-VP64)
```

- **SP** — signal peptide, for surface trafficking
- **BINDER** — the ligand-recognition domain. **This is what synbind designs.**
- **CORE** — receptor-family-specific machinery that converts ligand engagement into
  a proteolytic or conformational event
- **TMD** — transmembrane domain
- **OUTPUT MODULE** — released on activation; drives expression of a gene of interest

Only the binder determines *what* the cell senses. Everything else is off-the-shelf
and swappable. That is why binder selection is the whole problem.

---

## Receptor families

synbind treats receptor families as **data**, not code (ADR-003). Each family differs
in its core machinery, and — critically — in what kind of ligand it can detect.

### synNotch

Developed by the Lim lab (UCSF), published 2016. Repurposes the Notch receptor.

Natural Notch signalling, simplified:

1. Notch is displayed on the **receiving cell** as a type I transmembrane protein
2. A ligand on an adjacent **sending cell** engages the Notch ectodomain
3. **Mechanical force** from cell–cell contact opens the negative regulatory region
   (NRR), exposing a cleavage site
4. ADAM10/TACE cleaves the extracellular region (S2)
5. γ-Secretase cleaves within the TMD (S3)
6. The released intracellular domain enters the nucleus and drives transcription

**Step 3 is the constraint that matters.** Cleavage requires mechanical force, which
requires the ligand to be anchored to something — normally a neighbouring cell. This
is why synNotch fires in *trans* and is largely inert to a soluble ligand.

For natural Notch signalling this is a feature. For a designer choosing a scaffold,
it is a hard limit.

### SNIPR

Synthetic Intramembrane Proteolysis Receptors. A later generation with a redesigned
core, offering improved dynamic range, lower ligand-independent signalling, and
greater tolerance of ligand presentation format — including reported responsiveness
to soluble factors under some configurations.

### Others

MESA and GEMS-type receptors were designed for soluble ligand input from the start.
Not implemented; noted so the data model accommodates them.

---

## Ligand modality — the central design constraint

**This is the most important concept in the project.**

| Modality | Ligand is | Example targets |
|---|---|---|
| `membrane_bound` | anchored on a neighbouring cell | CD3ε, CD47 |
| `soluble` | free in the extracellular space | VEGF-A, TGF-β1 |

A force-dependent scaffold paired with a soluble ligand produces a receptor that
looks correct on paper, assembles fine, expresses fine, and never fires.

This failure is:

- invisible in any single paper, because papers report what worked
- cheap to encode once someone bothers
- expensive to discover experimentally

Hence: every target declares a modality, every scaffold declares which modalities it
supports, and mismatch is a blocking validation failure with a suggested alternative.

Two second-order notes:

- **Soluble ligands are often multimeric.** VEGF-A and TGF-β1 are both dimeric, so
  avidity and valency affect activation in ways a monomeric analysis misses.
- **Presentation matters as much as solubility.** A nominally soluble ligand
  immobilised on matrix or captured on a cell surface may support force-dependent
  signalling. Modality is a useful approximation, not a law.

---

## Requirements for the binder domain

These shape every rule in `synbind/validation/`.

### 1. Single polypeptide

The binder is fused directly to the receptor core as one continuous chain. Full IgG
(two chains held by disulfide bonds) cannot be used. Acceptable formats:

- **scFv** — VH + linker + VL. Most common. Both orientations (VH-VL, VL-VH) are
  worth generating; they do not always behave identically.
- **Nanobody (VHH)** — single domain, small, often an excellent folder. No assembly
  needed.
- **Natural receptor ectodomain** — usable if single-chain after removing the TM
  anchor. Physiologically relevant, but see off-target risk below.

### 2. Geometry under membrane constraint

The binder is not free in solution. It is tethered to the receptor and displayed at
a fixed distance and orientation from the membrane. For a membrane-bound ligand, it
must reach across the intercellular gap and engage its epitope without clashing with
the adjacent core domain.

synbind does not model this in v1 (ADR-004). It is noted because it explains why
solution-phase reasoning about binder–ligand pairs is an approximation.

### 3. Affinity in a window, not maximised

Counterintuitive and important: **tighter is not better.**

- Too tight — risk of ligand-independent (tonic) signalling; the receptor may also
  traffic poorly to the surface
- Too loose — weak or absent signal

Roughly 1–100 nM is the commonly cited productive range for this receptor class.
synbind does not predict affinity in v1; the constraint is documented so that no one
later adds a rule that rewards maximum affinity.

### 4. Specificity in both directions

Two distinct risks, with different consequences (ADR-009):

| | What it means | Consequence |
|---|---|---|
| **trans off-target** | Binder engages an unintended ligand on another cell | Receptor fires on the wrong cell |
| **cis off-target** | Binder engages something on its **own** host cell | Constitutive signalling — receptor is always on |

The cis case is the more dangerous and the less considered. It is also
species-dependent: when a receptor is expressed in one species' cells to detect
another species' ligand, host-cell paralogs of the target ligand are exactly the
proteins most likely to cause it.

Antibody-derived binders are generally specific by construction. **Natural receptor
ectodomains carry substantially higher risk** — they evolved in a context with
multiple binding partners, and their paralogs are widespread.

### 5. Immunogenicity of what is displayed

If the engineered cell is transplanted, everything on its surface is visible to the
recipient's immune system — including the receptor scaffold and the binder framework.
A murine-framework scFv is a liability in a way it would not be for a purely in vitro
application. This is why binder `origin_species` and scaffold part `species_origin`
are tracked.

---

## Sequence liabilities

Cheap to detect from sequence alone, and predictive of poor behaviour:

| Motif | Risk |
|---|---|
| Unpaired cysteine | Incorrect disulfide pairing, aggregation |
| N-X-S/T (X≠P) | N-glycosylation site; may block the binding interface |
| NG, NS | Deamidation — changes charge over time |
| DG | Isomerization |

These are warnings, not blockers. A liability inside a CDR matters far more than one
in a framework loop, and synbind does not currently make that distinction — so the
findings inform rather than decide.

---

## Terminology

| Term | Meaning |
|---|---|
| **Ectodomain** | The extracellular portion of a membrane protein |
| **scFv** | Single-chain variable fragment: VH + linker + VL |
| **VHH / nanobody** | Single-domain antibody fragment, camelid-derived |
| **VH / VL** | Heavy / light chain variable domain |
| **CDR** | Complementarity-determining region — the loops that contact the ligand |
| **Framework** | The structural scaffold surrounding the CDRs |
| **Tonic signalling** | Ligand-independent activation; the receptor fires without input |
| **cis / trans** | Interaction on the same cell / between two cells |
| **NRR** | Negative regulatory region — the force-sensitive element in Notch |

---

## Further reading

- Morsut et al. (2016) *Nature* — original synNotch
- Roybal et al. (2016) *Cell* — synNotch for therapeutic applications
- Toda et al. (2018) *Science* — multi-input synNotch logic
- Zhu et al. (2022) *Cell* — SNIPR

Citations are provided for orientation and have not been re-verified against the
current literature.
