# IDEAS.md — extensions and variations

Possible directions for synbind beyond v1. These are not in scope for v1 —
they live here so they don't get lost and so Claude Code sessions don't
accidentally implement them prematurely.

Each idea has a rough feasibility rating for a grad-student-scale project:
🟢 straightforward addition | 🟡 substantial effort | 🔴 multi-year project

---

## 1. ESM Atlas integration — novel binder discovery

🟡 **Effort: medium — 2–4 weeks on top of v1**

### The gap v1 has

synbind v1 finds binders that are **already in the literature** (SAbDab, UniProt).
It cannot discover binders that have never been experimentally characterized.

### What ESM Atlas enables

ESM Atlas (biohub.ai/esm/protein) is a map of 6.8 billion proteins organized by
functional similarity as learned by the ESMC protein language model. The key API:

```
GET /similarity-search?sequence={target_ectodomain}&topk_results=100
```

This returns proteins that are **functionally similar to the target** — not by
sequence homology, but by what the model learned about protein function from
evolution. Among those similar proteins, some will be receptor-like and some will
be previously uncharacterized binders.

### Integration plan

Add a new source in `binders.py`:

```python
class ESMAtlasSource:
    """
    Queries ESM Atlas with the TARGET sequence to find proteins
    that are functionally related to the target — these are candidates
    for natural binding partners that haven't been curated into UniProt
    interaction databases yet.

    Also queries with known BINDER sequences to find similar binders
    (functional analogs to a known scFv that may fold better or be
    smaller).
    """
    def search_by_target(self, target: Target) -> list[Binder]: ...
    def search_by_known_binder(self, binder: Binder) -> list[Binder]: ...
```

The SAE (sparse autoencoder) features returned by the Atlas API explain *why*
a protein was retrieved — features like "complement control protein module" or
"immunoglobulin-like binding domain" are directly interpretable. These should be
stored in `Binder.esm_features` for display in the v2 GUI.

### What this unlocks

- Binders for targets with NO known antibodies in SAbDab
- Smaller or better-expressed alternatives to known scFvs
- Discovery of natural binding partners from understudied organisms
  (the Atlas covers all of life, not just human proteins)

### Relationship to the ESM Atlas IC project

This is the bridge between the IC project (exploring ESM Atlas for
imunomodulatory protein discovery) and synbind (engineering synthetic
receptor binders). The IC project could be framed as a feasibility study
for this integration.

---

## 2. De novo binder design

🔴 **Effort: high — separate project**

Instead of searching for existing binders, generate entirely new proteins
computationally designed to bind a target.

Stack:
- **RFdiffusion** — diffusion model that generates protein backbones conditioned
  on a binding site
- **ProteinMPNN** — designs sequences that fold into those backbones
- **Boltz** — validates the designed sequences

The workflow would be:
1. Define the binding epitope on the target (user input or predicted by the pipeline)
2. RFdiffusion generates 100s of backbone candidates
3. ProteinMPNN designs sequences for each backbone
4. synbind validates each designed sequence through the existing structure + benchmark pipeline

The output is a **de novo scFv-like binder** with no natural analog.
High risk (most designs fail in the lab), but potentially high reward when
natural binders don't exist or are IP-encumbered.

---

## 3. Multi-antigen logic gating

🟡 **Effort: medium — 3–6 weeks**

synNotch circuits can implement AND gates: cell activates only when it detects
**both** antigen A and antigen B. This requires two separate synNotch receptors,
each with a different binder domain.

Extension: `synbind run --target Q08722 --and-target P15391`

The pipeline would:
1. Find binders for both targets independently
2. Check for cross-reactivity between the two binder sets
   (binder for target A must NOT bind target B, and vice versa)
3. Return two ranked lists with a combined compatibility score

Critical check: if binder-A also binds target-B, the AND gate leaks.
The off-target module in v1 already has this logic — just extend it to
check binder-A against target-B as a mandatory off-target check.

---

## 4. Nanobody library integration

🟢 **Effort: low — 1 week**

Nanobodies (VHH domains from camelid antibodies) are single-domain, small (~15 kDa),
and thermostable. They are increasingly used in synthetic receptor engineering.

Databases to integrate:
- **NbMiner** (https://nbminer.onlinebioinformatics.org) — searchable nanobody database
- **SAbDab** already includes some nanobody entries (filter by `nb:true`)
- **PubChem BioAssay** — some nanobody sequences are deposited there

In `binders.py`: add `NanobodySource` alongside `SAbDabSource`.
In `construct.py`: nanobodies don't need VH+VL assembly — they're already single-chain.
No linker needed. Construct building is simpler.

This is a quick win — add it in v1.1.

---

## 5. Affinity estimation

🟡 **Effort: medium — requires Rosetta or FoldX**

Boltz gives a confidence score (ipTM) but not a binding affinity (Kd).
For synNotch, Kd matters: too tight → tonic signaling (always on); too loose → no signal.
Optimal Kd range is roughly 1–100 nM for synNotch.

Possible approach:
- Use **FoldX** `AnalyseComplex` on the Boltz-predicted structure → ΔΔG estimate
- Use **Rosetta** `ddg_monomer` → more accurate but much slower
- Use **ESMFold2 + pairwise scoring** if Biohub exposes it

Add as an optional `--affinity` flag. Slow (minutes per complex), so run only
on top-ranked candidates.

---

## 6. Epitope mapping and blocking analysis

🟡 **Effort: medium**

If multiple binders are found for the same target, some may compete for the
same binding site (same epitope) while others bind distinct epitopes.

Competing binders: redundant — keep only the best one.
Non-competing binders: could be used together in a bispecific construct or
stacked synNotch design.

Implementation: cluster candidates by predicted interface residues on the TARGET.
Binders whose interface residues overlap > 50% are in the same epitope bin.

---

## 7. Web GUI (v2)

🟡 **Effort: 1–2 months for MVP**

Frontend: React + Tailwind
Backend: FastAPI wrapping the synbind pipeline
Structure viewer: [Mol*](https://molstar.org/) (MIT license, excellent, used by RCSB)

Key UI components:
- **Target search box** — autocomplete from UniProt (suggest gene names, not IDs)
- **Results table** — sortable, filterable, with tooltips explaining every metric
- **Structure viewer** — embedded Mol*, loads the predicted PDB directly
- **Off-target panel** — expandable per natural receptor binder
- **Export** — one-click FASTA download for top N constructs (ready for synthesis order)
- **Run history** — past runs stored per user session

Architecture: v1 CLI modules require zero changes. FastAPI calls `pipeline.run()`
directly and streams logs via Server-Sent Events (SSE) to the browser.

Deployment: Docker + any VPS. Runs fine on a $20/month machine for the team's usage.
GPU for Boltz: deploy separately (RunPod, vast.ai) and call via API.

---

## 8. Binder humanization scoring

🟢 **Effort: low — 1–2 days, mostly a lookup table**

For clinical-track applications, antibody-derived scFvs need to be "humanized"
— their framework regions need to match human germline sequences to avoid
immunogenicity when infused into patients.

Add a `humanization_score` column to `candidates.csv`:
- Use `anarci` to identify framework vs CDR residues
- Compare framework residues against human germline sequences (IMGT database)
- Report % framework identity to nearest human germline

This is informational only — the humanization itself is a wet-lab step
(or a separate computational tool). But flagging it is useful.

---

## 9. Integration with the IC ESM Atlas project

🟢 **Effort: low — these are already two separate codebases**

The IC project (`esm_atlas_xeno_search.py`) generates a ranked list of
candidate proteins similar to known immunomodulatory proteins.

Those candidates are protein sequences — exactly what synbind's step 2 returns.

Bridge: add a `--from-esm-atlas` flag to synbind:
```bash
synbind run --target Q08722 --from-esm-atlas results/atlas_raw_results.json
```

This reads the ESM Atlas output (the `atlas_raw_results.json` file produced by
the IC pipeline), treats the top-N candidates as binder candidates, and passes
them directly to construct building (step 3). No SAbDab / STRING search needed.

This makes the two projects a two-stage pipeline:
```
ESM Atlas IC project  →  novel candidate proteins
        ↓
     synbind          →  structural validation + off-target screening
        ↓
    wet lab team      →  experimental validation
```

---

## 10. Molecular dynamics — long-term stability

🔴 **Effort: very high — separate project, specialized expertise needed**

Boltz gives a static snapshot of the complex. MD simulations would show
whether the complex is stable over time and whether the binder stays engaged
during the mechanical force of cell-cell contact (relevant for synNotch, which
requires sustained engagement for Notch cleavage).

Tools: GROMACS, AMBER, or OpenMM. Timescale: microseconds requires
significant HPC resources.

Not realistic for a single IC. Flag for a future postdoc project.
