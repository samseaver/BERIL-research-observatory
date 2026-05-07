# Rosetta: UniProt-to-ModelSEED Reaction Mapping

## Research Question
What fraction of UniProt (as loaded in BERDL) can be reliably mapped to mass-balanced reactions in the ModelSEED Biochemistry, and what evidence supports each mapping?

## Status
In Progress — research plan created, schema discovery next.

## Overview
Individual mapping approaches (EC matching, database identifier matching, RAST annotation, fuzzy name matching) are each incomplete and noisy. This project systematically combines **all lines of evidence** available in the BERDL lakehouse — KEGG, BioCyc, EC numbers, PaperBLAST curation, InterPro domains, SEED annotations, and name matching — to produce a comprehensive, scored mapping with per-reaction evidence reporting.

A RAST-derived validation set (~2,000 template reactions linked to UniProt proteins) enables F1 scoring per evidence channel and for the integrated mapping.

The final data product is a reusable table mapping UniProt protein IDs to ModelSEED reaction IDs with confidence tiers and evidence provenance.

## Quick Links
- [Research Plan](RESEARCH_PLAN.md) — hypothesis, approach, query strategy
- [Report](REPORT.md) — findings, interpretation, supporting evidence

## Reproduction
*TBD — add prerequisites and step-by-step instructions after analysis is complete.*

## Authors
- Sam Seaver, KBase / Argonne National Laboratory
