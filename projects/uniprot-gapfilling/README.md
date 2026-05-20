# UniProt-Guided Gap-Filling via Enzyme and Reaction Embeddings

## Research Question
Can enzyme-similarity and reaction-similarity embeddings, combined with Rosetta's UniProt-to-ModelSEED mappings, propose biologically plausible gap-filling solutions that enable metabolic reconstructions to grow?

## Status
Complete — see [Report](REPORT.md) for findings. H0 not rejected: ESM-2 cosine similarity in the pretrained embedding space does not provide discriminative gap-filling evidence after pool-size correction and permutation testing.

## Overview
Standard gap-filling adds reactions with little biological evidence. This project uses ESM-2 protein embeddings to ask: does the genome encode an enzyme similar to known catalysts of each gap-filled reaction? [Rosetta mappings](../rosetta/) (UniProt → ModelSEED reactions with evidence tiers) identify which Swiss-Prot proteins catalyze each reaction; the `llm_homology_api` provides ESM-2 embeddings; FAISS cosine similarity scores genome proteins against candidates. The result is a gap-fill proposal grounded in sequence-level evidence.

## Data Sources
- Rosetta project mappings (`projects/rosetta/data/`)
- `kbase_msd_biochemistry` — ModelSEED reactions, compounds, stoichiometry
- `refdata_uniprot` — UniProt protein annotations and cross-references
- `kbase_ke_pangenome` — pangenome gene clusters and functional annotations

## Quick Links
- [Research Plan](RESEARCH_PLAN.md) — hypothesis, approach, query strategy
- [Report](REPORT.md) — findings, interpretation, supporting evidence

## Reproduction

### Prerequisites
- Python 3.10+
- Dependencies: `pip install -r requirements.txt`
- NB02 requires a live Spark session via `berdl_notebook_utils` (on-cluster only, or off-cluster with `.venv-berdl`)
- NB03 requires a live `llm_homology_api` connection, or use the cached `user_data/candidate_embeddings.json`
- Rosetta project outputs must exist at `projects/rosetta/data/` (run the Rosetta project first)

### Steps
1. **NB01** — Gap-fill landscape characterization. Reads Rosetta parquets (local, ~10s)
2. **NB02** — Candidate assembly. Requires Spark for `reaction_similarity` query (~2 min)
3. **NB03** — Embedding retrieval. Calls `llm_homology_api` in batches of 500 (~5 min with API)
4. **NB03b** — Embedding calibration. Local FAISS operations (~30s)
5. **NB04** — Similarity scoring. FAISS search across 48 genomes (~2 min)
6. **NB05** — Initial evaluation. Local computation (~15s). Note: NB05b supersedes the verdict
7. **NB05b** — Proxy inflation correction with permutation tests. Local computation (~30s)

### Cached Data
If the `llm_homology_api` is unavailable, NB03 can be skipped — the cached embeddings at `user_data/candidate_embeddings.json` are sufficient for NB03b onward. Similarly, NB02's Spark query results are cached in `user_data/proxy_candidates.parquet`.

## Authors
- Sam Seaver, KBase / Argonne National Laboratory
