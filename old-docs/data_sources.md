# Data sources

Every external service and local tool used by synbind, with endpoints,
rate limits, authentication, caching strategy, and known quirks.

---

## UniProt REST API

**Base URL:** `https://rest.uniprot.org/uniprotkb/`
**Auth:** none
**Rate limit:** ~3 req/s recommended; be polite

### Endpoints used

```
GET /uniprotkb/{id}.fasta
    → raw FASTA sequence (single entry)

GET /uniprotkb/{id}.json
    → full entry: gene name, organism, features (SIGNAL, TM, TOPO_DOM, BINDING...)

GET /uniprotkb/search
    ?query=interacts_with:{id} AND reviewed:true AND organism_id:9606
    &format=json
    &fields=accession,gene_names,sequence,go,cc_interaction
    → interaction partners for step 2B
```

### Features we read from the JSON entry

```json
"features": [
  {"type": "Signal",        "location": {"start": 1, "end": 21}},
  {"type": "Transmembrane", "location": {"start": 22, "end": 44}},
  {"type": "Topological domain",
   "description": "Extracellular",
   "location": {"start": 45, "end": 135}}
]
```

Used as fallback when DeepTMHMM subprocess fails.

### Caching

Cache both FASTA and JSON to `results/{run_id}/cache/uniprot_{id}.fasta`
and `results/{run_id}/cache/uniprot_{id}.json`. TTL: not enforced (sequences
don't change often); re-run with `--no-cache` to force refresh.

**Docs:** https://www.uniprot.org/help/api

---

## SAbDab (Structural Antibody Database)

**Base URL:** `https://opig.stats.ox.ac.uk/webapps/sabdab-sabdab/`
**Auth:** none
**Rate limit:** not documented; use 0.5 req/s

### Endpoint used

```
GET /search/
    ?antigen_uniprot={uniprot_id}
    &format=json
```

### Response structure

```json
[
  {
    "pdb": "6NZO",
    "Hchain": "H",
    "Lchain": "L",
    "antigen_chain": "A",
    "antibody_name": "Hu5F9",
    "resolution": 3.1,
    "species": "Homo sapiens"
  }
]
```

### How we get sequences

SAbDab returns chain IDs in the PDB file, not the sequences directly.
We then fetch the PDB from RCSB and extract sequences by parsing chain records.
Use `utils/pdb.py → extract_chain_sequence(pdb_path, chain_id)`.

**Quirk:** some SAbDab entries have `Lchain: null` (heavy-chain-only antibodies
like nanobodies). Handle gracefully — these still proceed as scFv if the single
chain contains a complete VH.

**Bulk download option:** SAbDab publishes a weekly summary TSV at
`https://opig.stats.ox.ac.uk/webapps/sabdab-sabdab/summary/`
For repeated development runs against the same target, download this once
and query it locally to avoid hammering their server.

**Docs:** https://opig.stats.ox.ac.uk/webapps/sabdab-sabdab/about/

---

## STRING

**Base URL:** `https://string-db.org/api/`
**Auth:** none
**Rate limit:** 1 req/s

### Endpoints used

```
GET /json/interaction_partners
    ?identifiers={uniprot_id}
    &species=9606
    &limit=50
    → list of interaction partners with sub-scores

GET /json/get_string_ids
    ?identifiers={uniprot_id}
    &species=9606
    → maps UniProt accession to STRING identifier
```

### Response fields we use

```json
[
  {
    "stringId_A": "9606.ENSP00000398632",
    "stringId_B": "9606.ENSP00000367469",
    "preferredName_A": "CD47",
    "preferredName_B": "SIRPA",
    "experimentally_determined_interaction": 821,
    "combined_score": 905
  }
]
```

**Filter:** keep only entries where `experimentally_determined_interaction > 400`.
Do not use `combined_score` alone — it includes text mining and co-expression,
which are not evidence of direct physical binding.

**Caching:** cache to `results/{run_id}/cache/string_{id}.json`

**Docs:** https://string-db.org/help/api/

---

## RCSB PDB

**Base URL:** `https://files.rcsb.org/` and `https://search.rcsb.org/`
**Auth:** none
**Rate limit:** none published; be courteous

### Endpoints used

```
GET https://files.rcsb.org/download/{PDB_ID}.pdb
    → PDB format structure file

POST https://search.rcsb.org/rcsbsearch/v2/query
    Content-Type: application/json
    → search for structures containing a given UniProt accession
```

### Search query to find co-crystals

```json
{
  "query": {
    "type": "group",
    "logical_operator": "and",
    "nodes": [
      {
        "type": "terminal",
        "service": "text",
        "parameters": {
          "attribute": "rcsb_polymer_entity_container_identifiers.uniprot_ids",
          "operator": "in",
          "value": ["Q08722", "P78552"]
        }
      }
    ]
  },
  "return_type": "entry",
  "request_options": {"results_content_type": ["experimental"]}
}
```

Use this in step 2B to check whether a UniProt-derived receptor binder
has a co-crystal with the target.

### Caching

PDB files for the fixed benchmark set: committed to `data/pdb_cache/`.
PDB files fetched at runtime: cached to `results/{run_id}/cache/pdb_{id}.pdb`.

**Docs:** https://data.rcsb.org/

---

## ESM Atlas API

**Base URL:** `https://biohub.ai/esm/protein/api/v1alpha1/`
**Auth:** none (alpha API, no key required as of v1)
**Rate limit:** not documented; use 1 req/s; add `DELAY_BETWEEN_CALLS = 1.5`

### Endpoints used

#### Similarity search (steps 2C and 6 pre-filter)

```
GET /similarity-search
    ?sequence={amino_acid_sequence}   ← max 800 residues
    &topk_results=50                  ← max 100
    &topk_features=20
    &min_similarity=0.75
    &include_cluster_info=true
```

**Important:** the API accepts a raw amino acid sequence and runs ESMC
embedding + SAE similarity search server-side. You do not generate
embeddings locally. The result is ranked by SAE feature cosine similarity,
not raw sequence identity — this is what makes it useful for finding
functionally similar but sequentially distant proteins.

**Response:**
```json
{
  "similar_proteins": [
    {
      "protein_accession": "uniprotkb:P78552",
      "protein_name": "SIRPA",
      "similarity_score": 0.84,
      "mean_plddt": 82.1,
      "cluster_size": 47,
      "protein_hash": "abc123..."
    }
  ],
  "top_features_across_results": [
    {
      "feature_index": 1234,
      "occurrence_count": 8,
      "mean_activation": 0.71
    }
  ]
}
```

**`restricted_count`:** number of results withheld by the biosecurity filter.
This is expected and normal — log it but do not treat as an error.

#### Protein detail lookup (optional enrichment)

```
GET /proteins/{protein_hash}
    → full entry: sequence, predicted structure (PDB string), SAE features
```

Use this to retrieve the predicted structure of an ESM Atlas hit
without running Boltz — saves compute for initial screening. Parse the
`pdb` field (PDB format string) and write to disk.

### Sequence length constraint

The API rejects sequences > 800 residues. For targets with longer
extracellular domains (e.g., large receptor ectodomains):
- Use the most distal 800 aa (N-terminal of the extracellular portion)
- Document the truncation in logs
- Flag these results as `truncated=True` in the output

### Caching

Cache results to `results/{run_id}/cache/esmatlas_{seq_hash}.json`
where `seq_hash = hashlib.md5(sequence.encode()).hexdigest()[:12]`.
The response is deterministic per sequence.

**Docs:** https://biohub.ai/esm/protein/docs/

---

## DeepTMHMM

**Type:** local tool (subprocess call), NOT an HTTP API
**Install:** `pip install pydeeptyhmm` or via Docker

### Usage

```python
import subprocess, tempfile
from pathlib import Path

def run_deeptmlhmm(sequence: str, id: str) -> Path:
    fasta = f">{id}\n{sequence}\n"
    with tempfile.NamedTemporaryFile(suffix=".fasta", mode="w", delete=False) as f:
        f.write(fasta)
        fasta_path = f.name
    out_dir = Path(f"/tmp/tmhmm_{id}/")
    out_dir.mkdir(exist_ok=True)
    subprocess.run(
        ["deeptmhmm", "--fasta", fasta_path, "--output", str(out_dir)],
        check=True, capture_output=True
    )
    return out_dir / f"{id}.gff3"
```

### Output format (GFF3)

```
##gff-version 3
target_id  DeepTMHMM  signal     1   21   .  .  .  .
target_id  DeepTMHMM  TMhelix    22  44   .  .  .  .
target_id  DeepTMHMM  outside    45  135  .  .  .  .
target_id  DeepTMHMM  TMhelix    136 158  .  .  .  .
target_id  DeepTMHMM  inside     159 220  .  .  .  .
```

Feature types: `signal`, `TMhelix`, `inside`, `outside`

Parse with:
```python
from BCBio import GFF   # pip install bcbio-gff
```
or with a simple line-by-line parser (GFF3 is tab-separated, no external lib needed).

### Docker alternative

```bash
docker run --rm -v $(pwd):/data ghcr.io/biolib/deeptmlhmm:latest \
  --fasta /data/target.fasta --output /data/tmhmm_out/
```

**Docs:** https://dtu.biolib.com/DeepTMHMM

---

## anarci

**Type:** local Python library (no HTTP)
**Install:** `pip install anarci`

Used in step 3 to identify and extract VH and VL domains from raw antibody
sequences, trim constant regions, and annotate CDR loops.

```python
import anarci

sequences = [("Ab_H", heavy_chain_sequence), ("Ab_L", light_chain_sequence)]
numbered, alignment_details, hit_table = anarci.run_anarci(
    sequences, scheme="imgt", output=False
)
```

`numbered` is a list of residue-number tuples: `[((1, ' ', 'Q'), ...), ...]`
Reconstruct the variable domain sequence by taking all residues from
position 1 through 128 (IMGT end of framework 4).

**Note:** `anarci` will return `None` for sequences it cannot number
(e.g., the sequence is too short, too diverged, or is a constant domain).
Handle this gracefully — skip the construct and log a warning.

**Docs:** https://github.com/oxpig/ANARCI

---

## Boltz

**Type:** local CLI tool (subprocess call)
**Install:** `pip install boltz && boltz download`
**Model weights:** ~2 GB, downloaded once to `~/.boltz/`

### Input format (multi-chain FASTA)

```fasta
>target|chain_A
MGNKACLS...
>construct|chain_B
QVQLVQSG...
```

### CLI call

```bash
boltz predict input.fasta \
  --out_dir ./out/ \
  --recycling_steps 3 \
  --diffusion_samples 5
```

### Output files

```
out/
├── predictions/
│   ├── complex_model_0.pdb          ← best ranked model
│   ├── complex_model_1.pdb          ← additional samples
│   └── confidence_model_0.json      ← confidence scores
└── msa/                             ← MSA files (can be ignored)
```

### Confidence JSON

```json
{
  "iptm": 0.87,
  "ptm": 0.91,
  "plddt": [82.1, 79.3, 91.2, ...]
}
```

`plddt` is a per-residue array over all residues in both chains concatenated.
Compute `mean_plddt = sum(plddt) / len(plddt)`.

### Selecting the best model

When `diffusion_samples > 1`, Boltz produces multiple models.
`complex_model_0.pdb` is the highest-ranked by Boltz's internal scoring.
Use it as the primary model. If you want to inspect alternatives, they are
`complex_model_1.pdb`, `complex_model_2.pdb`, etc.

**Docs:** https://github.com/jwohlwend/boltz

---

## mmseqs2

**Type:** local CLI tool
**Install:** `pip install mmseqs2` or `conda install -c bioconda mmseqs2`

Used in step 6 (off-target stage 1) for fast sequence similarity search
against the surface proteome.

```bash
mmseqs2 easy-search \
  construct.fasta \
  data/surface_proteome.fasta \
  results/offtarget_stage1.tsv \
  /tmp/mmseqs_tmp \
  --min-seq-id 0.30 \
  --cov-mode 0 \
  -c 0.5 \
  --format-output "query,target,pident,alnlen,evalue"
```

Output TSV: `query | target_accession | percent_identity | alignment_length | e-value`

Flag any hit with `pident > 30` as a potential cross-reactor.

**Docs:** https://github.com/soedinglab/MMseqs2

---

## COMPLIP (membrane context, optional)

**Type:** local tool, preprocessing for Boltz
**Install:** `pip install complip`
**Required only when:** `--membrane` flag is passed

COMPLIP adds lipid bilayer representation to protein inputs for structure
prediction models that support it. It identifies which chain is the
membrane protein, computes membrane embedding, and adds special tokens
to the Boltz input.

```bash
complip prepare input.fasta \
  --output input_membrane.fasta \
  --membrane-protein chain_A
```

**Status note:** COMPLIP + Boltz membrane support was experimental as of
mid-2025. Verify current compatibility with the installed Boltz version
before using in production. If COMPLIP is not installed, the `--membrane`
flag should raise a clear `DependencyNotInstalled` error, not a cryptic
subprocess failure.

**Docs:** https://github.com/COMPLIP (verify current URL)

---

## Caching summary

All cache operations go through `utils/cache.py`. The pattern everywhere is:

```python
cached = cache.get(key)
if cached:
    return cached
result = expensive_call()
cache.set(key, result)
return result
```

| Data | Cache key | Location |
|------|-----------|----------|
| UniProt FASTA | `uniprot_{id}.fasta` | `results/{run_id}/cache/` |
| UniProt JSON | `uniprot_{id}.json` | `results/{run_id}/cache/` |
| SAbDab results | `sabdab_{id}.json` | `results/{run_id}/cache/` |
| STRING results | `string_{id}.json` | `results/{run_id}/cache/` |
| ESM Atlas results | `esmatlas_{seq_hash}.json` | `results/{run_id}/cache/` |
| DeepTMHMM output | `tmhmm_{id}.gff3` | `results/{run_id}/cache/` |
| PDB files (runtime) | `pdb_{id}.pdb` | `results/{run_id}/cache/` |
| PDB files (benchmark) | `{id}.pdb` | `data/pdb_cache/` ← committed |
| Boltz output | `{construct_id}/complex_model_0.pdb` | `results/{run_id}/` |
