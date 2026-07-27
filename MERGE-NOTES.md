# What the mock produced that `main` does not have

**Branch:** `mocked` · **Written:** 2026-07-27 · **Covers:** commits `9fb3a84` → `7fa9287`

Building the interface mock forced decisions that are not recorded anywhere on
`main`. Some are genuine architecture changes; some are interface principles; one is
an open question with no answer yet.

This file is the handover. Nothing here has been applied to `main` — each item says
what it would touch and whether it is worth carrying over.

---

## Do not merge

| | |
|---|---|
| `mock/index.html` | Disposable prototype. Every sequence, accession, PDB ID and DOI in it is invented. |
| `MOCK.md` | Describes the fabrication exemption, which exists only on this branch. |
| This file, after the decisions are transferred | Its content belongs in `docs/decisions.md` and `CLAUDE.md`. |

The never-fabricate rule was never relaxed outside `mock/`. `data/`, `docs/`,
`synbind/` and `CLAUDE.md` are byte-identical to `main` on this branch — verified with
`git diff main --stat -- data/ docs/ CLAUDE.md` (empty).

---

## 1. The host cell is a missing required input

**This is the significant one.** Everything else on this list is smaller.

ADR-009 specifies two off-target screens against two different reference proteomes —
*trans* against the ligand's species, *cis* against the host cell's. Nothing in the
current data model captures the host cell, so the cis screen has no argument to run
on. It is not a refinement that can be added in v1.2; it is a missing parameter.

The mock made this concrete. Anti-CD3ε in a pig islet cell is a sound design. The
same construct in a Jurkat cell is dead on arrival — Jurkat is a T-cell line, it
displays CD3ε itself, so the binder meets its target on its own membrane and the
receptor is permanently on. Same binder, same scaffold, same target, opposite verdict.
Without the host cell the tool cannot say which situation you are in; the best it can
do is hedge in a caveat, which puts the reasoning back on the user.

A weaker version of the same problem affects the recommendation the tool already
ships: the entire argument for ranking a humanized binder above a murine one is that
the cells go into a recipient. In culture that argument evaporates. `main` currently
asserts that context rather than being told it.

### Proposed ADR-018 — the host cell is a first-class input

> **Status:** proposed
>
> **Context.** ADR-009 defines cis and trans off-target screens against different
> reference proteomes. The cis screen requires the host cell's species, which the data
> model does not carry. Separately, the immunogenicity rule and its ranking weight are
> only meaningful if the engineered cell is destined for a recipient — a fact the tool
> currently assumes rather than knows.
>
> **Decision.** Every design carries a `HostCell`. It is selected by **name from a
> curated list of cell lines and primary cell types**, not by species — species and
> destination are derived from the choice and shown back to the user. A
> `cis.self_display` rule becomes a **blocking failure**: if the target ligand is
> displayed by the host cell itself, the receptor is constitutively on and the design
> cannot work.
>
> **Consequences.** New model, new reference data file, and a new blocking rule.
> `DesignResult` must carry the host cell to remain reproducible. The immunogenicity
> rule and the ranking weight become conditional on `host.transplanted` (see ADR-019).
> The v1.2 cis screen gains the argument it needs, and part of ADR-009's value is
> delivered in v1 rather than v2 — the case where the host displays the *target
> itself* is catchable from curated data alone, with no proteome search.
>
> **Rejected.** Asking for a species. Wet-lab users think in cell lines, not in
> *Sus scrofa*, and species alone is too coarse: a pig fibroblast and a pig T cell
> share a proteome and have completely different surfaces. Same reasoning as
> autocompleting gene names instead of demanding accessions.
>
> **Rejected.** Making it optional. An optional field would be skipped, and the
> blocking check would silently not run.

### What it touches

**New model** in `models.py`:

```python
class HostCell(BaseModel):
    id: str                       # "jurkat", "pig_islet"
    display_name: str
    kind: str                     # "human cell line" | "primary cell"
    species: str | None           # None for "not listed"
    transplanted: bool            # destined for a recipient, or staying in culture
    displays: dict[str, str] | None   # target accession → "yes" | "probably"
    provenance: Provenance        # every marker claim needs a source
```

`displays: None` means unknown, not empty — the check must not run and must say so.
An empty dict means checked and nothing found. Collapsing the two would silently turn
"we did not look" into "it is safe".

**New reference data** `data/host_cells.yaml`, loaded through `registry/host_cells.py`
like every other data file. Keep the list short — the mock uses six, and each earns
its place by producing a different verdict:

| Cell | Why it is in the list |
|---|---|
| Pig islet cell (primary) | The lab's actual chassis. Default. |
| Pig endothelial cell (primary) | Second real chassis. |
| HEK293 | The clean control — displays nothing relevant. |
| Jurkat | T-cell line; displays the CD3 chains. The case that makes the rule visible. |
| K562 | Common synNotch chassis. |
| Something else | The honest fallback. |

**`DesignResult`** gains `host_cell: HostCell`. Without it a stored result cannot be
reproduced, because the verdict depends on it.

**New rule** in the validation table:

| Rule ID | Severity | Checks |
|---|---|---|
| `cis.self_display` | fail | The target is not displayed by the host cell itself |

**Two cautions carried over from building it:**

- **Do not imply completeness.** Surface-marker data per cell line is patchy;
  expression atlases give transcript levels, which is not the same as what is
  displayed and correctly trafficked. The honest v1 is a small hand-checked list of
  *known relevant* markers, with the scope of the claim stated. Same tiering as
  ADR-007 for binders.
- **Label the question precisely.** In synNotch work K562 is usually the *sender*
  cell, not the receiver. Ask "which cell will carry the receptor?", never "cell type".

---

## 2. Immunogenicity is context-dependent, and so is the ranking

### Proposed ADR-019 — immunogenicity applies only in vivo

> **Status:** proposed · depends on ADR-018
>
> **Context.** `immunogenicity.origin` warns on a non-human framework because
> everything on the surface of a transplanted cell is visible to the recipient's
> immune system. For a cell line in a dish, nothing is looking.
>
> **Decision.** `immunogenicity.origin` is a `warn` when `host.transplanted` is true
> and informational otherwise. The corresponding ranking term is applied on the same
> condition.
>
> **Consequences.** The scoring formula in `CLAUDE.md` changes from
> `-0.15 if origin_species not in ("human", "humanized")` to the same term gated on
> `host.transplanted`. The rationale text changes with the context, so the same two
> candidates are separated by different arguments in a dish and in a graft.

**Worth knowing:** in the mock this did *not* reverse any ranking. The murine binder
still sits below the humanized one in culture, because it also carries two
developability flags to the other's one. That is the honest outcome and it was
tempting to engineer a flip for demo value — don't. The visible change is in the
*reasoning*, not the order, and that is enough.

---

## 3. The user can choose a different candidate

ADR-008 leads with a single recommendation so the user is not stranded in a table.
It does not say the user may *select* a different one, and there is currently no
model support for it — which means a user who disagrees has nowhere to go.

**Proposed amendment to ADR-008** rather than a new ADR:

> Alternatives are selectable, not only viewable. Choosing one carries its caveats
> forward into the recommendation view and into the exported record, which notes
> whether the selection was the tool's or the user's.

**Touches:** `DesignResult` gains `selected_by: Literal["synbind", "user"]`, and the
exported record carries it. One line in the model, and it makes every stored result
honest about where the decision came from.

**Second reason to want it:** watching how quickly a user overrides the suggestion is
the cheapest available measure of whether they trust the ranking.

---

## 4. Blocking failures need a typed, actionable remedy

`ValidationFinding.suggestion` is currently `str | None`. The mock surfaced two
blocking failures with structurally different fixes:

| Failure | The fix is to change |
|---|---|
| `modality.compatible` | the **scaffold** (synNotch → SNIPR) |
| `cis.self_display` | the **host cell** (Jurkat → HEK293) |

A free-text string cannot drive a working "fix this" control, and the correct
suggestion has to be computed rather than written — the mock originally hardcoded
"use HEK293", which is wrong for CD47, where *no* listed cell avoids the target.

**Suggested shape:**

```python
class Remedy(BaseModel):
    field: Literal["scaffold", "host_cell", "binder", "target"]
    value: str | None      # None when no valid alternative exists
    message: str           # user-facing, and must work when value is None
```

The `value: None` case matters more than it looks — see the CD47 note below.

---

## 5. Findings should say what they did *not* flag

The sequon rule excludes `N-P-S/T` because a proline blocks attachment. In the mock,
one binder carries both a real `N-G-S` site and a near-miss `N-P-S`, and the finding
explains why the second is not flagged.

This is worth keeping as a behaviour, not just copy. A scientist who spots an
apparent miss and gets no explanation concludes the tool is unreliable; a tool that
says "we saw it, here is why it does not count" earns the opposite. Cheap to
implement — `liabilities.py` already computes the positions it rejects.

---

## 6. CD47 validates its own `role: benchmark`

With `cis.self_display` live, CD47 is blocked in **every** listed cell type, because
it is on essentially every cell. `data/targets.yaml` already says this in a comment;
the rule turns the comment into a demonstrated result. No data change needed — this
is confirmation that the `benchmark` vs `candidate_sensor` distinction is real and
mechanically detectable.

It is also the case that produced the `value: None` remedy above: the honest response
is not to suggest a different cell, but to say that no cell avoids it and that this is
exactly what makes CD47 a poor sensor.

---

## 7. Interface principles worth adding to `CLAUDE.md`

Three that came out of the density pass and are not currently written down:

- **Progressive disclosure applies inside a layer, not just between layers.** ADR-008
  covers recommendation → comparison → raw. Within the recommendation, show the
  warnings and fold the passes behind one line ("9 other checks passed"). Ten stacked
  findings read as a wall; one warning plus a count reads as an answer.
- **State an assumption, do not ask for it.** The host cell defaults to the lab's real
  chassis and is shown back in plain language, so a user gets an answer immediately
  and can correct what was assumed. This is the same pattern as resolving "CD3"
  ambiguity: show the choice, never hide it.
- **Exported artifacts must be self-describing.** A copied sequence or a downloaded
  file leaves the page and loses its context. It has to carry its own identification —
  which is the export-shaped version of ADR-011's argument for provenance.

---

## 8. Open question: the ranking formula has no off-target term

For CD47, a natural receptor ectodomain scores level with an antibody, because the
documented formula rewards structural evidence and penalises liabilities and
non-human origin — and says nothing about off-target risk. But `docs/biology.md`
states plainly that natural receptor ectodomains carry substantially higher risk than
antibody-derived binders.

So the formula and the biology document disagree, and the mock papered over it by
putting the argument in the rationale text and leaving the order to a tie-break.

Three ways out, none obviously right:

1. **Add a term** — `-0.10 if format == "receptor_ectodomain"`. Simple, encodes a real
   ordinal preference, and consistent with how the other weights work.
2. **Leave the formula alone** and keep handling it in the rationale, on the grounds
   that it is a judgement rather than a measurement until the v1.2 screen exists.
3. **Drop the scalar** and rank by explicit dominance across dimensions, which is
   where this formula is heading anyway once structure confidence and off-target
   counts arrive.

Whichever is chosen needs an ADR, because `CLAUDE.md` requires ranking changes to be
recorded.

---

## Suggested order, if any of this is merged

1. **ADR-018** — records the decision; costs nothing until implemented.
2. **ADR-019** and the ADR-008 amendment — both are small and depend on 018.
3. **`CLAUDE.md`** — data model additions, the new rule row, the revised scoring
   term, the three interface principles.
4. **`data/host_cells.yaml`** + `registry/host_cells.py` — real work, and every marker
   claim needs a real source before it can be trusted. Same verification burden as any
   other data file in this repo.
5. **Item 8** — needs a decision before `recommend.py` is written, since it changes
   what that module computes.

Items 1–3 are documentation and can land before any Python exists. Nothing here
changes the v1 build order in `docs/roadmap.md`; ADR-018 adds a step-2 deliverable
(`registry/host_cells.py`) and a step-4 rule.
