# UniProt-Guided Gap-Filling via Enzyme and Reaction Embeddings

## Research Question
Can enzyme-similarity and reaction-similarity embeddings, combined with Rosetta's UniProt-to-ModelSEED mappings, propose biologically plausible gap-filling solutions that enable metabolic reconstructions to grow?

## Status
In Progress — initial notebook uploaded, research plan drafted.

## Overview
Standard gap-filling approaches rely on database-wide reaction pools and cost minimization, often adding reactions with little biological evidence. This project builds on the [Rosetta mapping](../rosetta/) (UniProt → ModelSEED reactions with evidence tiers) to constrain gap-filling candidates using embedded enzyme similarities and embedded reaction similarities. By scoring candidate reactions against a genome's proteome, gap-fill proposals are grounded in sequence-level and functional evidence rather than purely stoichiometric feasibility.

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
