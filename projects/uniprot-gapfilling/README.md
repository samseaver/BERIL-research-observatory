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
*TBD — add prerequisites and step-by-step instructions after analysis is complete.*

## Authors
- Sam Seaver, KBase / Argonne National Laboratory
