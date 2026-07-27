# Reference data

Versioned biological reference data. Loaded and validated through `synbind/registry/`
— never parsed ad hoc elsewhere.

These files are meant to be **reviewed and edited by biologists**, not only by
programmers. Keep them readable, commented, and cited.

---

## ⚠️ Never fabricate a sequence

**This is the most important rule in the repository.**

An invented amino acid sequence that looks plausible is worse than no sequence at
all: it will be assembled, validated, exported, and potentially ordered. Nothing
downstream can detect it.

Any sequence field that is not yet filled in must contain the literal sentinel:

```yaml
sequence: TODO
```

`synbind/registry/` **must refuse to load** any record whose sequence is `TODO`,
raising an error naming the file and record. Incomplete records are visible failures,
never silent ones.

The same applies to accessions, PDB IDs, and DOIs. If you do not have the real value,
write `TODO` — do not guess.

---

## Files

| File | Contents |
|---|---|
| `targets.yaml` | Target ligands: accession, modality, species, role |
| `scaffolds/*.yaml` | Receptor scaffolds: ordered parts, constraints, compatible modalities |
| `binders_seed.yaml` | Hand-curated binders — the high-confidence tier (ADR-007) |
| `linkers.yaml` | Linker sequences for scFv assembly |

Each record carries a `provenance` block. A binder without provenance cannot appear
in a recommendation (ADR-011).

---

## Verification status

Accessions and identifiers in these files were recorded from memory during initial
setup and are marked `verified: false`. **Every one must be checked against the
primary source before the data is used for anything real.**

Set `verified: true` only after checking against UniProt, RCSB, or the cited paper —
and record who checked it in the provenance block.
