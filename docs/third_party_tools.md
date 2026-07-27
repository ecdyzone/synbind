# Third-party tools and APIs to build on

Survey of existing open-source software and free APIs that synbind can glue together
instead of reimplementing.

**Guiding principle:** write almost no algorithms. Write a well-tested pipeline that
calls other people's algorithms and produces a trustworthy, reproducible artifact.

Scope decisions referenced below are recorded in [`decisions.md`](decisions.md).

---

## Status of this document

⚠️ **Nothing here has been verified against live docs yet.** This was assembled from
model knowledge with a May 2026 cutoff. Versions, API shapes, endpoint paths, and
licenses all need checking before we take a hard dependency on anything.

Verification status per entry:

- 🔴 **unverified** — from memory, needs checking before we depend on it
- 🟡 **partially verified** — exists for sure, but API/license details unconfirmed
- 🟢 **verified** — checked against current docs, date noted

Everything is 🔴 at time of writing.

**License caveat:** several entries are academic-use-only, which constrains what can
be shipped in a public repository. Flagged per entry where known; all need confirming.

---

## Headline finding

**No tool does this end-to-end.** There is no software that takes a target antigen and
returns a deployable synthetic-receptor construct. The gap is real.

But roughly **80% of the internals are already solved.** So the project's value is
glue, curation, and the few things nobody else does — see
[What is actually ours](#what-is-actually-ours) at the bottom.

---

## 1. Closest existing things to what we're designing

| Tool | What it is | Verdict | Status |
|---|---|---|---|
| **Addgene** | Plasmid repository. Published synNotch/SNIPR backbones deposited with full sequences. Has an API. | **Use it.** This is where the scaffold registry gets real sequences instead of transcribing from papers. | 🔴 |
| **SynBioHub / SBOL** | Open standard + repository for synthetic biology parts (`pySBOL3`, `sbol_utilities`) | Adopt the *data model* for construct representation. Interoperability for free, and good engineering taste. | 🔴 |
| **Benchling** | Commercial ELN, free academic tier, has an API. Wet lab may already live here. | **Integrate, don't replace.** Exporting into their existing tool beats making them learn ours. | 🔴 |
| **Tamarind Bio / Neurosnap** | Commercial web frontends running Boltz, ABodyBuilder, RFdiffusion etc. with a GUI | Competitor for the *structure-prediction UI* niche. Reinforces cutting Boltz from v1 — that space is taken. | 🔴 |

None of these do antigen → binder → assembled receptor.

---

## 2. Antibody handling (mAb → scFv)

**Worth stating plainly: scFv conversion is not a hard problem.** It is
`VH + linker + VL`. The difficulty is entirely in (a) getting correct VH/VL domain
boundaries and (b) sourcing the sequence in the first place.

| Tool | Purpose | Notes | Status |
|---|---|---|---|
| **ANARCI** (Oxford OPIG) | Antibody numbering, CDR/framework assignment | The standard. Conda install, requires HMMER. Not cleanly pip-installable. | 🔴 |
| **RIOT / `riot-na`** (NaturalAntibody) | Modern numbering, positioned as faster ANARCI | **Evaluate first** — pip-installable, much better install story for the laptop constraint. | 🔴 |
| **AbNumber** | Friendlier wrapper over ANARCI | Conda. | 🔴 |
| **Thera-SAbDab** | Therapeutic antibodies indexed by INN name, with sequences | **Probably the highest-value single source.** Maps "pembrolizumab" → actual VH/VL. This is exactly the manual lookup the wet lab does by hand. | 🔴 |
| **BioPhi** (Prihoda et al.) | Open-source humanization + "humanness" scoring | Directly addresses immunogenicity of a murine-framework scFv displayed on a transplanted pig cell. | 🔴 |
| **IgBLAST** (NCBI) | Germline assignment | Local, free. | 🔴 |
| **ABodyBuilder2 / 3** (OPIG) | Antibody structure prediction | Seconds, not hours. If we ever want structure in v1, **this** is the affordable version — not Boltz. | 🔴 |

---

## 3. Binder discovery

| Source | Purpose | Notes | Status |
|---|---|---|---|
| **RCSB PDB Search API** | Structures by UniProt accession | REST + GraphQL, well documented. **Use this instead of scraping SAbDab.** More reliable, and gets sequences from the same place. | 🔴 |
| **SAbDab** (OPIG) | Structural antibody database | Richer antibody-specific annotation, but access is a TSV dump, not an API. Gives PDB + chain IDs, *not* sequences. | 🔴 |
| **INDI / NbMiner** | Nanobody databases | For the VHH path. | 🔴 |
| **UniProt REST** | Target sequences, topology, interactions | Good API, CORS-enabled (matters if we ever want browser-side lookups). | 🔴 |
| **IntAct / PDBe-KB** | Protein interactions | Better curated for *direct physical* binding than STRING, which mixes in functional association. | 🔴 |

---

## 4. Off-target screening — the highest-value section

| Tool | Purpose | Notes | Status |
|---|---|---|---|
| **MMseqs2** | Sequence search | Use instead of BLAST. Orders of magnitude faster, trivially local, no NCBI rate limits. | 🔴 |
| **DIAMOND** | Sequence search | Alternative to MMseqs2, same reasoning. | 🔴 |
| **Foldseek** | *Structural* similarity search | **Strictly better than sequence search for off-targets** — cross-reactivity is a structural property, and sequence-dissimilar proteins can share a fold. | 🔴 |
| **AlphaFold DB** | Predicted structures, proteome-wide | **Includes the *Sus scrofa* proteome.** | 🔴 |

### Why this combination matters

Foldseek + pig AlphaFold DB makes the **cis-activation screen** tractable on a laptop:

> Does this binder hit anything on the *pig* cell surface? Foldseek the human target
> against the pig proteome, get structural homologs, flag them.

No GPU. Minutes, not hours. This is likely the single most valuable thing the software
can do, and the tooling already exists — we just have to point it in the right direction.

See **ADR-009** in [`decisions.md`](decisions.md): trans-screening against the ligand's
species and cis-screening against the host cell's species are *different screens with
different consequences*, and the cis one is the failure mode that silently kills
experiments. Scheduled as deliverable v1.2 in [`roadmap.md`](roadmap.md).

---

## 5. DNA output (if in scope)

| Tool | Purpose | Notes | Status |
|---|---|---|---|
| **DNA Chisel** (Edinburgh Genome Foundry) | Codon optimization as constraint solving | Optimize for *Sus scrofa* codon usage while avoiding restriction sites, GC extremes, repeats. Open, pip-installable. | 🔴 |
| **DnaCauldron** (EGF) | Assembly simulation | Same group. | 🔴 |
| **DnaWeaver** (EGF) | Ordering strategy | Same group. | 🔴 |
| **GeneBlocks** (EGF) | Sequence block manipulation | Same group. | 🔴 |
| **pydna** | Cloning simulation | | 🔴 |
| **Biopython** | Sequence handling | Baseline dependency. | 🔴 |

**If the wet lab's actual pain is "make me an orderable sequence," then this suite plus
the scaffold registry *is* the product — and it needs no GPU at all.**

---

## 6. Immunogenicity

Relevant because the construct is displayed on a pig cell to a human immune system.

| Tool | Purpose | Notes | Status |
|---|---|---|---|
| **IEDB** | Epitope database + T-cell epitope prediction | Free API. | 🔴 |
| **NetMHCpan** | MHC binding prediction | Standard. Free for academics — license needs checking for a public repo. | 🔴 |
| **BioPhi** | Humanness scoring | Also listed under antibody handling. | 🔴 |

---

## 7. Structure prediction

Deprioritized for v1 — heaviest, slowest, GPU-bound, least differentiated, and the
niche where commercial GUIs already compete.

| Tool | Notes | Status |
|---|---|---|
| **Boltz-2** | Now includes affinity prediction, which addresses the Kd-window concern (synNotch needs ~1–100 nM; too tight causes tonic signaling). | 🔴 |
| **Chai-1** | Open weights. | 🔴 |
| **ESMFold** | Single-sequence, fast, no MSA needed. | 🔴 |
| **ColabFold** | Free notebooks + MMseqs2 MSA API. Note: the public MSA server is rate-limited and is a single point of failure for batch runs. | 🔴 |
| **AlphaFold Server** | Free web, daily quota, zero install. **Good enough for a PI demo.** | 🔴 |
| **ABodyBuilder3** | Antibody-specific, seconds per structure. Best cost/benefit if we want any structure in v1. | 🔴 |

---

## 8. Membrane context

| Tool | Notes | Status |
|---|---|---|
| **CHARMM-GUI Membrane Builder** | The academic standard for building membrane systems. Free for academics. | 🔴 |
| **OPM / PPM server** | Precomputed / computed membrane positioning of structures. | 🔴 |
| **MemProtMD** | Coarse-grained MD of membrane proteins. | 🔴 |
| **Martini / martinize2** | Coarse-grained force field + tooling. | 🔴 |
| ~~**COMPLIP**~~ | ⚠️ **Existence unconfirmed** — the described `boltz predict --membrane` interface does not match mainline Boltz. **Rejected as a dependency (ADR-017).** Membrane context is out of scope. If it ever returns, prefer the established tools above. | 🔴 |

---

## What is actually ours

If most components exist, the honest question is: what is *ours*?

Three things nobody appears to be doing:

1. **Species-directed cis/trans off-target screening.**
   Trans against the human proteome, cis against the pig proteome, with different
   consequences for each. Not seen anywhere else, and it is the failure mode that
   kills experiments — a binder that engages its own host cell surface causes tonic
   signaling and the transgene is always on.

2. **The receptor scaffold registry.**
   Turning "which parts, in which order, with which constraints" into versioned data
   that generalizes across synNotch, SNIPR, and whatever comes next. Adding a new
   receptor family should be a YAML file plus tests, not a refactor.

3. **Curation and delivery for people who don't code.**
   Everything listed above is a CLI for bioinformaticians. None of it is usable by a
   wet-lab biologist. The last mile is the product.

Everything else should be a dependency, not code.

---

## Next actions

- [ ] Verify the load-bearing dependencies before committing to them. Priority order
      follows the roadmap: binder-discovery sources (Thera-SAbDab access model, RCSB
      Search API shape) for v1.1, then Foldseek + predicted proteome availability for
      v1.2
- [ ] Confirm licenses for anything shipped in a public repo (NetMHCpan, ANARCI, and
      IMGT-derived data are the likely friction points)
- [x] ~~Get a source for COMPLIP or drop it~~ — dropped, ADR-017
- [x] ~~Decide whether DNA output is in scope~~ — out of scope, ADR-001. The Edinburgh
      Genome Foundry suite stays on the backlog, not a dependency

**Note:** v1 has no third-party dependencies from this document at all — it is pure
Python over a curated data registry (ADR-007). The first real external tool
dependencies arrive with v1.1 and v1.2.
