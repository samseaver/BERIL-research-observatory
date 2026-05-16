# Metabolite Distribution — Predicting Metabolite Abundance from Enzyme Neighbourhood Transcripts

## Research Question

Can transcript abundances of enzymes neighbouring a metabolite in the metabolic network predict that metabolite's abundance across a diurnal time course in Arabidopsis leaf tissue?

## Status

In Progress — awaiting data upload, project scaffolded.

## Overview

Paired transcriptomics and metabolomics data were collected from Arabidopsis leaf tissue every 4 hours over 28 hours (8 timepoints, 4 biological replicates per timepoint, 32 samples total). Both omics layers come from the same physical samples, removing inter-sample variability as a confounder.

Rather than using rigid metabolic pathway boundaries, we define "enzyme neighbourhoods" — sets of enzymes that are topologically close to a metabolite in the metabolic network. For each metabolite, we build linear models using transcript abundances of its neighbourhood enzymes as predictors. This tests whether local enzymatic context is sufficient to explain metabolite variation, and whether this relationship holds differently for central vs specialized metabolism.

## Data Sources

- Arabidopsis leaf diurnal time course (user-provided)
  - Transcriptomics: gene-level abundance (8 timepoints × 4 replicates)
  - Metabolomics: metabolite abundance (matched samples)
- Enzyme neighbourhood mappings (user-provided)

## Quick Links

- [Research Plan](RESEARCH_PLAN.md) — hypothesis, approach, analysis strategy
- [Report](REPORT.md) — findings (TBD)

## Reproduction

TBD — will be updated after analysis is complete.

## Authors

- Sam Seaver, Argonne National Laboratory / KBase
