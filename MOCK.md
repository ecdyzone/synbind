# This branch is a disposable UI prototype

**Branch:** `mocked` — not intended to merge into `main`.

It contains one deliverable, `mock/index.html`: a static walkthrough of the synbind
interface, built to show coworkers and collect feedback before any Python is written.
It is a picture of an application, not an application.

## What it covers

Two targets have candidates attached. Between them they exercise every state worth
reacting to:

| Pick this | To see |
|---|---|
| **TGF-β1** + synNotch | The blocking failure — a secreted ligand cannot pull, so the receptor never fires. Suggests SNIPR. |
| **TGF-β1** + SNIPR | A passing result across four binder formats: scFv in both orientations, a nanobody, a natural receptor ectodomain. |
| **CD3ε** + synNotch | A passing result on the *original* scaffold, where the tool ranks the famous antibody (UCHT1) below a humanized one — because these cells go into a recipient. |
| Type **CD3** | Ambiguity resolved rather than guessed: ε, δ and γ are offered. |

VEGF-A, CD3δ, CD3γ and CD47 resolve but have no binders, which is its own state.

---

## The fabrication exemption, and its exact boundary

`CLAUDE.md` and `data/README.md` forbid inventing sequences, accessions, PDB IDs, and
DOIs. That rule exists because an invented sequence that looks plausible will be
assembled, validated, exported, and potentially ordered, and nothing downstream can
detect it.

**On this branch only, `mock/` is exempt.** Everything in it is invented: binder
names, sequences, PDB IDs, DOIs, affinities, and curator names. None of it was taken
from a paper or a database.

The exemption stops there:

| | |
|---|---|
| ✅ Invented data allowed | `mock/index.html` |
| ❌ Rule still absolute | `data/`, `docs/`, `synbind/`, and all of `main` |

No file outside `mock/` and this one is modified on this branch.

Four of the accessions shown (P01137, P15692, P07766, Q08722) are the ones already
recorded in `data/targets.yaml`, still `verified: false` there. Two more (P04234,
P09693, for CD3δ and CD3γ) exist only in the mock, to demonstrate resolving the
"CD3" ambiguity — they are unchecked like the rest.

## Guardrails in the page itself

- A persistent header badge, visible in every screenshot.
- Copying a sequence, or downloading the FASTA or JSON, prepends a
  `# MOCK DATA — invented, not a real sequence` line, so a paste into a lab notebook
  or an order form carries its own warning. Downloaded filenames end in `_MOCK`.
- A build tag in the footer, so feedback can be tied to the version it was written
  against.

## Do not

- Copy any sequence, identifier, or citation from `mock/` into `data/`. The real
  values must come from primary references, per `data/README.md`.
- Treat the validation findings as real. They are hand-written to demonstrate the
  interface. They *are* internally consistent with the invented sequences — the
  liability counts were computed from them — but the sequences are fiction.

## When feedback is collected

Delete the branch. The real interface is step 6 of the build order in
`docs/roadmap.md`, built on the real core with real data.
