# Research Plan: Rosetta — UniProt-to-ModelSEED Reaction Mapping

## Research Question
What fraction of UniProt (as loaded in BERDL) can be reliably mapped to mass-balanced reactions in the ModelSEED Biochemistry, and what evidence supports each mapping?

## Hypothesis
- **H0**: No single combination of evidence channels achieves >50% coverage of mass-balanced reactions with F1 >0.7 against RAST validation.
- **H1**: Combining multiple evidence channels (KEGG, BioCyc, EC, PaperBLAST, InterPro, name matching) achieves substantially higher coverage and accuracy than any single channel, reaching >70% reaction coverage with F1 >0.8 on the RAST validation set.

## Literature Context
Genome-scale metabolic model (GEM) reconstruction depends on mapping genes/proteins to biochemical reactions. Automated tools (RAST, ModelSEED, CarveMe, gapseq) each use different annotation pipelines and identifier systems, producing incomplete and partially overlapping mappings. The ModelSEED Biochemistry provides a unified reaction namespace, but bridging from UniProt — the central protein reference — to ModelSEED reactions remains fragmented across EC numbers, KEGG, MetaCyc, and curated databases. No systematic assessment of multi-evidence mapping coverage exists.

## Approach
Systematically extract every available evidence channel for UniProt→reaction mapping from BERDL, combine them with confidence scoring, and validate against RAST annotations.

## Data Sources

### Primary
| Database | Table(s) | Purpose |
|----------|----------|---------|
| `kbase_msd_biochemistry` | `reaction`, `molecule`, `reagent` | Target reaction set (mass-balanced filter via `status`) |
| `refdata_uniprot` / `kbase_uniprot_kb` | `uniprot_identifier` (~2.5B rows) | UniProt cross-references (KEGG, BioCyc, Reactome, etc.) |
| `u_seaver__msd_biochemistry` | 6 tables (TBD) | User's augmented biochemistry (may have EC mappings) |

### Evidence Channels
| Database | Table(s) | Evidence Type |
|----------|----------|---------------|
| `kbase_ke_pangenome` | `eggnog_mapper_annotations` (93M) | EC, KEGG_ko, KEGG_Reaction |
| `kbase_ke_pangenome` | `bakta_annotations`, `bakta_db_xrefs` | EC, UniRef50 bridge |
| `kescience_paperblast` | `curatedgene` (255K) | Literature-curated gene→function |
| `kescience_interpro` | `protein2ipr` (1.18B), `interproscan_pathways` (287M) | Domain→EC→reaction |
| `kescience_fitnessbrowser` | `seedannotation`, `seedclass` | SEED role→reaction |

### Validation
- User-provided RAST annotation file (~2,000 reactions linked to UniProt proteins via RAST)
- Located in `user_data/`

## Query Strategy

### Tables Required
| Table | Purpose | Estimated Rows | Filter Strategy |
|---|---|---|---|
| `kbase_msd_biochemistry.reaction` | Mass-balanced reaction set | 56K | Filter by `status` |
| `kbase_msd_biochemistry.reagent` | Stoichiometry | 263K | Join on `reaction_id` |
| `kbase_msd_biochemistry.molecule` | Compound details | 46K | Join on `molecule_id` |
| `*.uniprot_identifier` | UniProt cross-refs | 2.5B | Filter by `db` column, one type at a time |
| `eggnog_mapper_annotations` | EC/KEGG annotations | 93M | Aggregate EC→reaction |
| `paperblast.curatedgene` | Curated annotations | 255K | Full scan OK |
| `interpro.protein2ipr` | Domain annotations | 1.18B | Filter by `ipr_id` list |

### Performance Plan
- **Tier**: JupyterHub Spark SQL (on-cluster)
- **Estimated complexity**: Moderate-High (multiple large table joins)
- **Known pitfalls**:
  - Must `SET spark.sql.autoBroadcastJoinThreshold = -1` before any join on `uniprot_identifier` (2.5B rows)
  - `reaction.abbreviation` holds KEGG R-numbers for ~80% of reactions only
  - No EC column on `reaction` (per pitfalls.md) — bridge via abbreviation patterns or user's augmented DB
  - ModelSEED IDs use `seed.reaction:rxnNNNNN` format, not bare `rxnNNNNN`
  - Process one `uniprot_identifier.db` type at a time to manage memory

## Analysis Plan

### Notebook 1: Schema Discovery (Spark)
- **Goal**: Characterize all relevant table schemas, enumerate cross-reference types, identify mass-balanced filter, find differences in user's augmented biochemistry
- **Expected output**: Schema inventory CSVs, cross-reference type counts

### Notebook 2: Database Identifier Matching (Spark)
- **Goal**: Map UniProt proteins to reactions via KEGG and BioCyc cross-references
- **Expected output**: Per-channel mapping TSVs

### Notebook 3: EC Number Evidence (Spark)
- **Goal**: Collect EC numbers from all sources (UniProt, eggNOG, bakta, user DB) and map to reactions
- **Expected output**: EC→reaction lookup, UniProt→EC→reaction mappings

### Notebook 4: PaperBLAST Evidence (Spark)
- **Goal**: Extract literature-curated gene→reaction links from PaperBLAST
- **Expected output**: PaperBLAST evidence TSV

### Notebook 5: InterPro Domain Evidence (Spark)
- **Goal**: Map InterPro domains with EC associations to reactions
- **Expected output**: InterPro evidence TSV

### Notebook 6: Fuzzy Name Matching (local)
- **Goal**: Match UniProt protein descriptions to reaction names
- **Expected output**: Name match TSV (lowest confidence)

### Notebook 7: Evidence Integration (local)
- **Goal**: Combine all evidence, assign confidence tiers, compute coverage statistics
- **Expected output**: Integrated mapping table, coverage statistics, evidence contribution analysis

### Notebook 8: RAST Validation (local)
- **Goal**: Compute F1 scores per evidence channel and combined against RAST ground truth
- **Expected output**: Validation results, precision-recall analysis

### Notebook 9: Summary Visualizations (local)
- **Goal**: Generate final data product and publication-quality figures
- **Expected output**: Final mapping table, all figures

## Expected Outcomes
- **If H1 supported**: Multi-evidence approach achieves >70% reaction coverage with F1 >0.8 — demonstrates the value of systematic evidence integration for metabolic model reconstruction
- **If H0 not rejected**: Coverage remains below 50% or F1 below 0.7 — identifies which evidence channels are most limiting and what additional data sources are needed
- **Potential confounders**: Many-to-many EC→reaction mappings inflate apparent coverage; RAST validation set may not be representative of all reaction types

## Revision History
- **v1** (2026-05-07): Initial plan

## Authors
- Sam Seaver, KBase / Argonne National Laboratory
