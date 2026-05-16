# Research Plan — Metabolite Distribution

## Research Question

Can transcript abundances of enzymes in a metabolite's network neighbourhood predict that metabolite's abundance in Arabidopsis leaf tissue across a diurnal time course?

## Hypothesis

- **H1**: Metabolite abundance can be predicted from transcript levels of enzymes in the metabolite's enzyme neighbourhood, with model performance (R²) significantly above a null model using random gene sets.
- **H0**: Transcript abundances of neighbourhood enzymes are no more predictive of metabolite levels than random gene sets of equal size.

**Secondary hypothesis**: Predictive power varies by metabolic context — specialized metabolism metabolites (with fewer producing/consuming enzymes) will be better predicted than central metabolism metabolites (which integrate many pathways).

## Literature Context

Transcript-metabolite correlations are notoriously weak genome-wide, but improve substantially when restricted to functionally related gene sets. Diurnal regulation in Arabidopsis creates coordinated oscillations in both transcripts and metabolites, providing natural variation to exploit for modeling. The enzyme neighbourhood approach avoids the arbitrary boundaries of curated pathways while maintaining biological relevance.

## Approach

### Data

- **Organism**: Arabidopsis thaliana (leaf tissue)
- **Time course**: 28 hours, sampled every 4 hours (8 timepoints)
- **Replicates**: 4 biological replicates per timepoint (32 samples total)
- **Key feature**: Transcriptomics and metabolomics from the same physical sample

### Analysis Plan

#### NB01 — Data Loading and QC
- Load transcriptomics and metabolomics matrices
- Sample metadata: timepoint, replicate, sample pairing
- QC: missing values, distribution shapes, outlier detection
- Normalization assessment (log-transform, scaling)
- PCA/UMAP of both omics layers — do timepoints separate?

#### NB02 — Enzyme Neighbourhood Definition and Exploratory Analysis
- Load user-provided gene-to-metabolite neighbourhood mappings
- Characterize neighbourhood sizes (genes per metabolite, metabolites per gene)
- Pairwise transcript-metabolite correlations within neighbourhoods
- Compare within-neighbourhood vs random correlations
- Identify metabolites with strongest/weakest neighbourhood signal

#### NB03 — Per-Neighbourhood Linear Models
- For each metabolite: fit linear model using neighbourhood enzyme transcripts as predictors
- Handle high-dimensional neighbourhoods: ridge/elastic net regularization when p > n
- Leave-one-timepoint-out cross-validation (grouped by timepoint to avoid replicate leakage)
- Compare against null model: same-sized random gene sets (permutation test, 1000 iterations)
- Output: R², RMSE, p-value vs null for each metabolite

#### NB04 — Global Model Comparison
- Aggregate per-metabolite results: how many metabolites are well-predicted (R² > threshold)?
- Compare central vs specialized metabolism performance
- Feature importance: which enzymes contribute most across metabolites?
- Visualize: volcano plot (R² vs significance), pathway-level summaries

#### NB05 — Temporal Dynamics
- Do time-lagged transcripts improve prediction? (transcript at t-1 → metabolite at t)
- Granger causality tests for well-predicted metabolites
- Diurnal phase analysis: are the best-predicted metabolites those with strong circadian regulation?

### Expected Outcomes

- A ranked list of metabolites by predictability from neighbourhood transcripts
- Evidence for whether enzyme neighbourhood is a useful predictor set definition
- Identification of metabolic contexts where transcript-metabolite coupling is strong vs weak

## Revision History

- **v1** (2026-05-16): Initial research plan
