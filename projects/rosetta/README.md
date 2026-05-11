# Rosetta: UniProt-to-ModelSEED Reaction Mapping

## Research Question
What fraction of UniProt (as loaded in BERDL) can be reliably mapped to mass-balanced reactions in the ModelSEED Biochemistry, and what evidence supports each mapping?

## Status
Complete — see [Report](REPORT.md) for findings.

## Overview
Individual mapping approaches (EC matching, database identifier matching, RAST annotation, fuzzy name matching) are each incomplete and noisy. This project systematically combines **all lines of evidence** available in the BERDL lakehouse — KEGG, BioCyc, EC numbers, PaperBLAST curation, InterPro domains, SEED annotations, and name matching — to produce a comprehensive, scored mapping with per-reaction evidence reporting.

A RAST-derived validation set (~2,000 template reactions linked to UniProt proteins) enables F1 scoring per evidence channel and for the integrated mapping.

The final data product is a reusable table mapping UniProt protein IDs to ModelSEED reaction IDs with confidence tiers and evidence provenance.

## Quick Links
- [Research Plan](RESEARCH_PLAN.md) — hypothesis, approach, query strategy
- [Report](REPORT.md) — findings, interpretation, supporting evidence

## Data Collections
`kbase_msd_biochemistry`, `refdata_uniprot`, `kbase_ke_pangenome`, `kescience_fitnessbrowser`, `refdata_interpro`, `u_seaver__msd_biochemistry`

## Reproduction
1. Ensure access to the BERDL lakehouse (on-cluster or off-cluster with proxy)
2. Place user data files in `user_data/` (see RESEARCH_PLAN.md for file descriptions)
3. Run notebooks 01-08 in order; each is self-contained with saved outputs
4. Requires: pandas, numpy, matplotlib, upsetplot

## Authors
- Sam Seaver, KBase / Argonne National Laboratory
