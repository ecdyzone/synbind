# Roadmap

Version plan for synbind: from CLI tool to web application for wet lab users.

---

## v1 — CLI pipeline (current)

**Target users:** the developer and bioinformatically capable collaborators.
**Delivery:** Python package, pip install, runs on Linux.
**Status:** in development.

### v1 scope (all of this is in scope now)

- `synbind run --target {uniprot_id}` — full pipeline
- Binder discovery: SAbDab + UniProt/STRING
- Optional ESM Atlas binder discovery (`--esmatlas`)
- scFv construction (both orientations) and ectodomain extraction
- Boltz structure prediction and docking
- iRMSD benchmark against PDB crystal structures
- Off-target screening (two-stage: mmseqs2 + ESM Atlas + Boltz)
- Output: `candidates.csv`, `offtargets.csv`, PDB files
- Optional membrane context (`--membrane`)
- `--resume` for interrupted runs

### v1 explicit non-scope

Everything in this list is deferred to v2 or later:

- Any web interface, REST API, or GUI
- User accounts, authentication, job history
- Affinity prediction (Kd)
- Molecular dynamics
- Multi-target logic gates (AND/OR synNotch circuits)
- Sequence optimization / affinity maturation
- De novo protein design
- Automated report generation (PDF, Word)

---

## v1.1 — Hardening (post-first-paper)

Short iteration after the pipeline produces its first validated result.

- Expanded benchmark set (target: 20+ complexes, including natural receptors)
- `synbind verify-pdb` command for benchmark entry validation
- Proper test coverage (target: 80% on step modules)
- GitHub Actions CI: lint + type check + unit tests on push
- `pyproject.toml` with pinned dependency versions
- Docker image for reproducible runs (`docker run synbind run --target Q08722`)
- Basic documentation site (mkdocs or sphinx, auto-deployed from `docs/`)

---

## v2 — Web application for wet lab users

**Target users:** wet lab collaborators with no coding experience.
**Delivery:** web application, hosted internally or on a cloud instance.
**Trigger:** when multiple wet lab users need the pipeline regularly enough
that teaching them the CLI is slower than building a UI.

### What wet lab users need from v2

Based on the user context (colleagues who don't code), v2 must hide all
pipeline complexity behind a clean interface. A wet lab researcher should be able to:

1. Enter a gene name or UniProt ID in a search box
2. Click "Find binders"
3. See a ranked table of candidates with visual indicators (colored scores)
4. Click a candidate to view the 3D structure in-browser
5. Download a one-page PDF summary for lab meetings
6. Export the candidate list as CSV for Excel

They should never see: FASTA files, command flags, iRMSD numbers without
explanation, PDB file paths, or error tracebacks.

### v2 architecture plan

**Backend:** FastAPI wrapping the existing synbind Python package.
The CLI pipeline becomes a set of API endpoints. No business logic is
rewritten — the v1 code runs unchanged behind the API.

```
POST /api/runs              → create a new run (returns run_id)
GET  /api/runs/{id}         → poll status (queued | running | done | failed)
GET  /api/runs/{id}/candidates → ranked candidate list
GET  /api/runs/{id}/offtargets → off-target results
GET  /api/runs/{id}/structure/{construct_id} → PDB file for 3D viewer
```

**Job queue:** Celery + Redis. Boltz runs are slow — the user submits a job
and gets notified (email or in-app) when done. No synchronous requests.

**Frontend:** React. Three views:
- **Search:** gene name / UniProt ID input, run options (checkboxes for
  `--esmatlas`, `--membrane`), submit button
- **Results:** ranked table with colored ipTM badges, off-target count,
  RMSD badge (green/yellow/red). Click row → opens structure viewer.
- **Structure viewer:** 3D molecule viewer (Mol* or NGL Viewer, both open source,
  embed via iframe or JS library). Show predicted complex with interface
  residues highlighted.

**Auth:** institution SSO (if deployed internally) or simple invite-only
email+password for external collaborators. Nothing public-facing.

**Hosting options (in order of preference):**
1. USP HPC cluster with a GPU node — best for Boltz runtime; IT support needed
2. AWS EC2 with a GPU instance (g5.xlarge ~$1/hr) — flexible; pay per use
3. Google Colab Pro as a stopgap — works but not suitable for lab-wide use

### v2 feature mapping

| v1 CLI flag | v2 UI equivalent |
|-------------|-----------------|
| `--target Q08722` | Gene name / UniProt search box |
| `--esmatlas` | Checkbox: "Include novel binders (ESM Atlas)" |
| `--membrane` | Checkbox: "Include membrane context (slower)" |
| `--no-offtarget` | Checkbox: "Skip off-target screening (faster)" |
| `--diffusion-samples 5` | Radio: "Fast (draft)" / "Accurate (final)" |
| `--resume` | Not exposed — handled automatically via job ID |
| `candidates.csv` | Results table + CSV download button |
| `offtargets.csv` | Off-targets tab in results view |
| `complex_predicted.pdb` | In-browser 3D viewer + PDB download button |

### v2 non-scope (defer to v3 or never)

- Public access / SaaS model
- Multi-user collaboration on the same run
- Automated experimental ordering (oligo synthesis integration)
- Natural language query ("find me a binder for the thing that mediates phagocytosis")

---

## v3 — Advanced features (speculative)

These are ideas that require significant new capabilities. No timeline.

- **Multi-target circuit design:** given targets A and B, find binder pairs
  for AND-gate synNotch (fire only when both antigens are present).
  Requires combinatorial search and a circuit-level validation layer.

- **Affinity tuning:** given a validated binder, suggest point mutations to
  adjust Kd into the moderate range optimal for synNotch (not too tight,
  not too weak). Requires integration with a ΔΔG prediction tool.

- **In silico CDR grafting:** take CDR loops from one antibody and graft
  them onto a different framework — useful when the original framework
  has expression or immunogenicity issues.

- **Experimental feedback loop:** when wet lab results come in (SPR affinity,
  ELISA binding, synNotch activation assay), automatically re-calibrate
  the pipeline ranking weights (`decisions.md → rank_score formula`).

---

## Migration path from v1 to v2

The v1 CLI must be designed with v2 in mind. Specific requirements:

1. **No side effects in step functions.** Each `s{N}_*.py` function must be
   pure (input → output) with no global state. The FastAPI layer will call
   these functions directly.

2. **Structured output from every step.** The dataclasses in `models.py`
   must be JSON-serializable (add `asdict()` or use `dataclasses.asdict()`).
   The API will return these as JSON responses.

3. **Progress reporting.** `pipeline.py` must emit progress events that can
   be forwarded to a Celery task status. Use a callback pattern:
   ```python
   def run_pipeline(target_id, on_progress=None):
       ...
       if on_progress:
           on_progress(step=2, message="Found 14 binders")
   ```

4. **Idempotent runs.** The `--resume` / cache system already handles this.
   Ensure every step checks the cache before doing work — the API will
   restart jobs on server restart.

5. **No `print()` in library code.** Already a convention. The web backend
   captures structlog output; bare prints are lost.

If these are followed throughout v1 development, the v1 → v2 migration is
wrapping the existing code in a FastAPI app, not a rewrite.
