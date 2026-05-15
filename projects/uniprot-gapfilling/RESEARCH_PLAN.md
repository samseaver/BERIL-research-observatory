# Research Plan: UniProt-Guided Gap-Filling via Enzyme and Reaction Embeddings

## Research Question
Can enzyme-similarity and reaction-similarity embeddings, combined with Rosetta's UniProt-to-ModelSEED mappings, propose biologically plausible gap-filling solutions that enable metabolic reconstructions to grow?

## Hypothesis
- **H0**: Embedding-guided gap-filling does not improve biological plausibility over standard cost-minimization gap-filling (comparable or worse growth restoration rate with no improvement in evidence support)
- **H1**: Embedding-guided gap-filling produces solutions with stronger biological evidence (higher fraction of gap-filled reactions supported by enzyme similarity to the organism's proteome) while maintaining comparable growth restoration

## Literature Context
*TBD — to be expanded with literature review*

## Approach
1. Start from the Rosetta evidence-scored UniProt→ModelSEED reaction mappings
2. Use enzyme embeddings to identify proteins in a target genome that are similar to enzymes catalyzing candidate gap-fill reactions
3. Use reaction embeddings to identify reactions similar to those already present in the draft reconstruction
4. Score candidate gap-fill reactions by combining enzyme similarity, reaction similarity, and Rosetta evidence tiers
5. Compare against standard gap-filling in terms of growth restoration and evidence support

## Data Sources

### From Rosetta Project
| Source | Purpose |
|---|---|
| `evidence_integration_summary.parquet` | Scored UniProt→reaction mappings with evidence tiers |
| `ec_to_reaction.parquet` | EC number to ModelSEED reaction bridge |
| `template_coverage_summary.tsv` | Template reaction coverage baseline |

### From BERDL
| Table | Purpose | Filter Strategy |
|---|---|---|
| `kbase_msd_biochemistry.reaction` | Reaction definitions and status | `status = 'OK'` for mass-balanced |
| `kbase_msd_biochemistry.compound` | Compound definitions | As needed |
| `refdata_uniprot.protein` | Protein annotations | By organism |
| `kbase_ke_pangenome.gene` | Gene functional annotations | By species |

### User-Provided
| File | Purpose |
|---|---|
| *Initial notebook (to be uploaded)* | Existing prototype of the embedding-based approach |

## Analysis Plan

### Notebook 1: Existing Prototype (uploaded)
- **Goal**: Establish baseline approach from prior work
- **Expected output**: Initial gap-filling candidates and scoring

### Subsequent Notebooks: TBD
- Will be defined after reviewing the uploaded notebook and identifying next steps

## Expected Outcomes
- **If H1 supported**: Embedding-guided gap-filling provides biologically grounded solutions — potential integration into KBase model reconstruction pipeline
- **If H0 not rejected**: Embeddings add complexity without improving biological plausibility — standard gap-filling remains preferred
- **Potential confounders**: Embedding quality varies by protein family; training data bias in embeddings

## Revision History
- **v1** (2026-05-15): Initial plan — awaiting notebook upload to refine

## Authors
- Sam Seaver, KBase / Argonne National Laboratory
