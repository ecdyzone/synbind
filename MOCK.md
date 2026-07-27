# This branch is a disposable UI prototype

**Branch:** `mocked` — not intended to merge into `main`.

It contains one deliverable, `mock/index.html`: a static walkthrough of the synbind
interface, built to show coworkers and collect feedback before any Python is written.
It is a picture of an application, not an application.

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

No file outside `mock/` and this one is modified on this branch. The four accessions
that do appear (P01137, P15692, P07766, Q08722) are the ones already recorded in
`data/targets.yaml`, and they are still `verified: false` there.

## Guardrails in the page itself

- A persistent header badge, visible in every screenshot.
- Copying any sequence prepends a `# MOCK DATA — invented, not a real sequence` line,
  so a paste into a lab notebook or an order form carries its own warning.

## Do not

- Copy any sequence, identifier, or citation from `mock/` into `data/`. The real
  values must come from primary references, per `data/README.md`.
- Treat the validation findings as real. They are hand-written to demonstrate the
  interface. They *are* internally consistent with the invented sequences — the
  liability counts were computed from them — but the sequences are fiction.

## When feedback is collected

Delete the branch. The real interface is step 6 of the build order in
`docs/roadmap.md`, built on the real core with real data.
