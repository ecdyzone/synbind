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
| **CD3ε** + **Jurkat** | The one to show first. Jurkat is a T-cell line, so it carries CD3ε itself — the binder finds its target at home and the receptor is stuck on. Same construct, different cell, opposite verdict. |
| **TGF-β1** + synNotch | The other blocking failure — a released signal cannot pull, so the receptor never fires. Suggests SNIPR. |
| **CD3ε** + pig islet cell | The tool ranks the famous antibody (UCHT1) *below* a humanized one, because these cells go into a recipient. Switch the cell to HEK293 and the reasoning visibly changes. |
| **TGF-β1** + SNIPR | Four binder formats compared: scFv both ways round, a nanobody, a natural receptor piece. |
| Type **CD3** | Ambiguity resolved rather than guessed: ε, δ and γ are offered. |
| Any **Use this** button | Overriding the suggestion. The choice is carried through to the sequence and the exported record. |

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

The page says "mock" in exactly three places — the banner, the footer label, and the
build tag. Repeating it in every panel was noise; the banner does the job.

- A persistent header badge, visible in every screenshot.
- Copying or downloading a sequence prepends a
  `# MOCK DATA — invented, not a real sequence` line, and filenames end in `_MOCK`.
  This is the one place the warning must be repeated: exported content travels
  somewhere the banner cannot follow it.
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
