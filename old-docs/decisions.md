# Design decisions

A log of every significant architectural and scientific choice, with reasoning.
Update this file when making a non-obvious decision so future Claude Code
sessions understand why things are the way they are and don't "fix" them.

---

## Boltz over AlphaFold3

**Decision:** Boltz-1 is the sole structure prediction and docking engine.
AlphaFold3 is not used and should not be added without explicit user request.

**Reasoning:**
- AlphaFold3 requires a restrictive license (no commercial use, API key from
  Google DeepMind). Boltz-1 is MIT-licensed and runs fully locally.
- Boltz performs folding and docking in a single forward pass — no need to
  chain separate tools.
- Published benchmarks show Boltz achieves comparable accuracy to AF3 on
  antibody–antigen complexes (the primary use case here).
- Local execution means no API rate limits, no data leaves the lab, and
  runs can be batched on institutional HPC.

---

## anarci over IMGT/V-QUEST web API

**Decision:** VH/VL domain extraction uses `anarci` (local Python library).
The IMGT/V-QUEST web API is not called anywhere in the codebase.

**Reasoning:**
- IMGT/V-QUEST has experienced extended downtime periods and has an unpublished
  rate limit that interrupts automated pipelines.
- `anarci` is a local implementation of the same numbering algorithm, runs in
  milliseconds, is deterministic, and produces identical variable domain
  boundaries.
- For a pipeline that may process dozens of antibodies per run, a web dependency
  at step 3 would be a fragile bottleneck.

---

## Both scFv orientations built by default

**Decision:** for every antibody binder, step 3 produces two constructs:
`scFv_VH_VL` (VH + linker + VL) and `scFv_VL_VH` (VL + linker + VH).
Both are passed to Boltz. The winner is the one with higher mean pLDDT.

**Reasoning:**
- Neither orientation is universally better. The optimal orientation depends
  on the specific VH/VL pair and the antigen geometry.
- Running both costs one extra Boltz call per antibody but prevents systematic
  errors from always choosing VH-first.
- In practice, the two orientations can differ by > 0.1 ipTM for the same
  antibody — a difference large enough to change rankings.
- The "winner" is selected automatically. The wet lab user never has to know
  about this distinction.

---

## (G₄S)₃ as the default linker

**Decision:** the default scFv linker is `GGGGSGGGGSGGGGSGGGGS` (15 residues).
Alternative lengths are available in `data/linkers.yaml` but not used by default.

**Reasoning:**
- The 15-residue (G₄S)₃ linker is the most validated in the literature for
  scFv constructs across a wide range of antigen sizes and epitope geometries.
- Shorter linkers (< 10 aa) force VH-VL association and can produce diabodies
  rather than monomeric scFvs — undesirable for synNotch.
- Longer linkers (> 20 aa) are used for sterically challenging epitopes but
  introduce more conformational entropy and are not needed as a default.
- The linker is parameterized in `data/linkers.yaml` so it can be changed
  experimentally without touching code.

---

## iRMSD at 8Å cutoff over Cα atoms only

**Decision:** benchmark RMSD is computed over Cα atoms of interface residues
(residues within 8Å of the partner chain). Not full-chain RMSD, not all atoms.

**Reasoning:**
- Full-chain RMSD is dominated by the arbitrary relative translation and
  rotation of the two proteins in space. Two identical binding poses placed
  differently in the unit cell would show a high full-chain RMSD.
- All-atom RMSD over the interface is noisier than Cα-only — side chain
  conformations are less reliably predicted and add variance without signal.
- iRMSD over Cα at 8Å is the standard metric used in CAPRI (Critical
  Assessment of Predicted Interactions), the international protein docking
  benchmark. Using the same metric makes results directly comparable to
  published Boltz and AF3 evaluations.
- 8Å captures direct contact residues plus the first shell of packing
  residues, which is the biologically relevant region for specificity.

---

## Two-stage off-target filter

**Decision:** step 6 uses mmseqs2 (sequence similarity) as stage 1 and
ESM Atlas similarity search as stage 2, before running Boltz only on the
filtered set. Running Boltz against the full human surface proteome is not done.

**Reasoning:**
- The human surface proteome contains ~5,000 reviewed membrane proteins.
  Boltz docking for all of them per construct = ~100,000 CPU hours. Infeasible.
- Stage 1 (mmseqs2, <30 seconds): catches obvious cross-reactors by sequence
  similarity. Any surface protein with >30% sequence identity to the target
  will likely share the epitope. Flag without Boltz.
- Stage 2 (ESM Atlas, <5 seconds): the Atlas finds functionally similar
  proteins in a much richer space than sequence alone. Its top-50 hits are
  the highest-probability off-targets not already caught by stage 1.
- Running Boltz on ≤50 proteins per construct is feasible (2–4 hours on GPU).
- False negatives exist (a true off-target could be missed). This is acceptable
  for a computational pre-screening tool — the wet lab validation will catch it.
  False positives (flagging harmless proteins) are also acceptable; the cost
  is a conservative prioritization.

---

## Extracellular domain only, not full protein

**Decision:** Boltz receives only the extracellular domain of the target
(and the construct), not the full protein including TM and intracellular regions.

**Reasoning:**
- synNotch binding occurs at the cell surface. The TM and intracellular regions
  are embedded in the membrane and not accessible for binding.
- Including TM regions in Boltz input misleads the model — it would try to fold
  a membrane protein in aqueous context, producing unreliable predictions.
- The extracellular domain is the biologically correct unit of analysis.
- Shorter inputs also reduce Boltz memory requirements and runtime.

---

## Membrane mode is opt-in, not default

**Decision:** `--membrane` (COMPLIP + Boltz lipid bilayer mode) is an optional
flag. The default pipeline runs without membrane context.

**Reasoning:**
- COMPLIP + Boltz membrane support was experimental as of mid-2025.
  Stability and compatibility with Boltz versions beyond 0.4 is not guaranteed.
- Membrane mode adds ~30–60% compute overhead. Using it for initial screening
  of all candidates wastes resources.
- The correct workflow is: run standard mode for ranking → run membrane mode
  only for the top 3–5 candidates before experimental validation.
- Making it opt-in rather than opt-out prevents accidental use in development
  loops where speed matters more than accuracy.

---

## httpx over requests

**Decision:** all HTTP calls use `httpx` through a shared client in `utils/http.py`.
No `import requests` anywhere in the codebase.

**Reasoning:**
- `httpx` supports both sync and async, has a cleaner API for configuring
  retry and timeout behavior, and is the modern standard.
- A shared client (not a new client per call) enables connection pooling —
  relevant when making many sequential calls to the same host (e.g., UniProt).
- Future async support (parallel binder searches in step 2) is easier to
  add with `httpx.AsyncClient` than by retrofitting `requests`.

---

## dataclasses over Pydantic for models

**Decision:** `models.py` uses stdlib `@dataclass`, not Pydantic `BaseModel`.

**Reasoning:**
- No HTTP request/response validation is needed — data comes from internal
  pipeline steps, not user-submitted JSON.
- Pydantic adds a non-trivial dependency and runtime validation overhead that
  buys nothing here.
- `@dataclass` with full type hints and `mypy --strict` gives sufficient
  type safety for this use case.
- If a REST API is added in v2, revisit this decision at that time.

---

## structlog over standard logging

**Decision:** `structlog` is used for all logging. No bare `print()` outside `cli.py`.

**Reasoning:**
- structlog produces structured (JSON-compatible) log output, which is easier
  to parse when debugging multi-step pipeline failures.
- Each log event carries context (e.g., `construct_id`, `step`, `uniprot_id`)
  automatically via the context variable mechanism — no need to manually
  include these in every log call.
- The same logger configuration works for both human-readable terminal output
  (development) and machine-readable output (future CI/CD or remote runs).

---

## ESM Atlas min_similarity = 0.75 for binder discovery

**Decision:** the default `min_similarity` for ESM Atlas calls in step 2C is 0.75.
The off-target pre-filter uses 0.65.

**Reasoning (step 2C):**
- Below 0.75 in SAE feature space, hits become too diverse to be reliable
  candidates — the recall-precision tradeoff favors too many false positives.
- 0.75 was selected empirically by querying known receptor pairs (e.g., CD47/SIRPα)
  and checking that the natural binding partner appears in the top results.

**Reasoning (step 6 off-target pre-filter):**
- For off-target screening we want higher recall (fewer missed off-targets) at
  the cost of more Boltz calls. 0.65 casts a wider net.
- The subsequent Boltz docking filters the noisy hits. The cost of a false
  positive here is one Boltz call; the cost of a false negative is a missed
  safety concern.

---

## run_id is a timestamp, not a hash

**Decision:** results directories are named `results/{YYYYMMDD_HHMMSS}_{target_id}/`
rather than a content hash of the inputs.

**Reasoning:**
- Human-readable. A wet lab collaborator reading the file system can immediately
  tell when a run was made and which target it was for.
- Multiple runs of the same target are expected (parameter tuning) — a hash
  would produce the same directory name and either overwrite or collide.
- The `--resume` flag handles re-use of a specific prior run directory explicitly.
