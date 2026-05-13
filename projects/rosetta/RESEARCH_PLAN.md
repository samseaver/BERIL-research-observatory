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
| Database | Table(s) | Purpose | Rows |
|----------|----------|---------|------|
| `kbase_msd_biochemistry` | `reaction`, `molecule`, `reagent` | Target reaction set; 34,343 mass-balanced (status=OK) | 56K / 46K / 263K |
| `refdata_uniprot` | `identifier` (partitioned) | UniProt cross-references | 4.35B |
| `refdata_uniprot` | `entity`, `protein`, `name` | UniProt protein metadata | 335M / 215M / 734M |
| `u_seaver__msd_biochemistry` | Same as standard + extra `reagents` table | User's augmented biochemistry (same reaction schema, no extra columns) | ~same |

### Bridge Tables (user-provided)
| File | Purpose | Rows |
|------|---------|------|
| `user_data/Unique_ModelSEED_Reaction_ECs.txt` | EC→ModelSEED reaction lookup (25,757 reactions, 7,351 ECs) | 30,789 |
| `user_data/uniprot_rast_filtered_mapping.tsv.gz` | UniProt protein→RAST annotation (32.2M contain ECs, 2,336 unique ECs) | 84.5M |

### Evidence Channels (NB01-confirmed)

Prioritized by data provenance: UniProt-native annotations first (highest curation), then pangenome/external annotations, then indirect chains.

#### Tier 1 — UniProt-native (highest priority)
| Channel | Source | Evidence Type | Scale | Bridge |
|---------|--------|---------------|-------|--------|
| 1. UniProt EC | `identifier WHERE db='EC'` | Direct EC xrefs | 38.6M rows, 34.9M proteins | EC→reaction lookup |
| 2. UniProt BRENDA | `identifier WHERE db='BRENDA'` | BRENDA EC numbers | 38.6K rows | EC→reaction lookup |
| 3. Rhea catalytic activity | `comment_xml` WHERE content LIKE '%Rhea%' | Curated Rhea IDs + co-annotated EC from catalytic activity XML | 330K rows, 236K proteins, 13,589 distinct Rhea IDs | EC extracted from same XML block; 78% of rows (257K/330K) have EC co-annotated; 33.8K proteins have Rhea but no EC (mostly transporters) |

#### Tier 2 — Pangenome annotations
| Channel | Source | Evidence Type | Scale | Bridge |
|---------|--------|---------------|-------|--------|
| 4. eggNOG EC | `eggnog_mapper_annotations.EC` | EC annotations | 26.0M (27.8% of 93.6M) | EC→reaction lookup |
| 5. eggNOG KEGG_Reaction | `eggnog_mapper_annotations.KEGG_Reaction` | Direct KEGG R-numbers | 20.3M (21.7%) | R-number→reaction.abbreviation |
| 6. bakta EC | `bakta_annotations.ec` + `bakta_db_xrefs WHERE db='EC'` | EC annotations | 19.0M / 15.2M | EC→reaction lookup |
| 7. bakta KEGG | `bakta_db_xrefs WHERE db='KEGG'` | KEGG gene IDs | 16.7M | Need KEGG gene→reaction bridge |

#### Tier 3 — User-provided / curated external
| Channel | Source | Evidence Type | Scale | Bridge |
|---------|--------|---------------|-------|--------|
| 8. RAST annotations | `user_data/uniprot_rast_filtered_mapping.tsv.gz` | EC extracted from RAST text | 32.2M with ECs | EC→reaction lookup |
| 9. PaperBLAST | `curatedgene` (255K) | EC from desc field + curated function | 255K | EC→reaction lookup |
| 10. FitnessBrowser seedclass | `seedclass` (EC in `num` col, `type` col) | EC numbers | 61.9K rows | EC→reaction lookup |
| 11. FitnessBrowser besthitmetacyc | `besthitmetacyc` | Direct MetaCyc reaction IDs (`rxnId`) + EC (`ecnum`) | 59.7K rows; `rxnId` contains MetaCyc IDs (e.g., `RXN-11834`, `PREPHENATEDEHYDRAT-RXN`); `ecnum` has corresponding EC; `protId` links to MetaCyc protein | MetaCyc rxnId→reaction.abbreviation; ecnum→EC lookup |

#### Tier 4 — Indirect / low confidence
| Channel | Source | Evidence Type | Scale | Bridge |
|---------|--------|---------------|-------|--------|
| 12. InterPro→GO→EC | `protein2ipr` → `go_mapping` → EC | Indirect domain chain | 1.18B → 30K GO mappings | GO→EC→reaction |
| 13. Name matching | `reaction.name` vs UniProt descriptions | Fuzzy text matching | Always available | Direct |

### Key Discoveries
- KEGG xrefs in `identifier` are gene IDs (e.g., `pvk:EPZ47_00465`), not R-numbers — use eggNOG's `KEGG_Reaction` instead
- BioCyc xrefs in `identifier` are protein-monomer IDs (e.g., `MetaCyc:MONOMER-21486`), not reaction IDs — bridge via `besthitmetacyc.rxnId`
- Rhea is NOT in the `identifier` table — it lives in `comment_xml` as catalytic activity XML containing `<dbReference type="Rhea" id="RHEA:nnnnn"/>` alongside co-annotated EC (`<dbReference type="EC" id="x.x.x.x"/>`) and ChEBI substrate/product IDs
- Of 236K Rhea-annotated proteins, 229.7K (97.3%) also have EC in the `identifier` table; 6,434 have Rhea only — small but high-quality additions (curated SwissProt)
- InterPro `entry` table has NO EC column — must chain via GO mappings
- `reaction.abbreviation` contains KEGG R-numbers (9,339), MetaCyc patterns (7,312), and EC-like patterns (2,596)
- `besthitmetacyc` provides direct MetaCyc reaction IDs (e.g., `RXN-11834`, `CARBOXYCYCLOHEXADIENYL-DEHYDRATASE-RXN`) that can match `reaction.abbreviation` MetaCyc patterns — de-prioritized vs UniProt-native data but useful as confirmatory evidence

## Query Strategy

### Performance Plan
- **Tier**: JupyterHub Spark SQL (on-cluster)
- **Estimated complexity**: Moderate-High (multiple large table joins)
- **Known pitfalls**:
  - Must `SET spark.sql.autoBroadcastJoinThreshold = -1` before any join on `identifier` (4.35B rows)
  - `identifier_partitioned` exists but has same row count — use `identifier` with `db` filter
  - `reaction.abbreviation` holds KEGG R-numbers for 9,339 reactions (not 80% as initially estimated)
  - No EC column on `reaction` — use user-provided EC→reaction lookup
  - ModelSEED IDs use `seed.reaction:rxnNNNNN` format, not bare `rxnNNNNN` — but user lookup uses bare `rxnNNNNN`
  - Process one `identifier.db` type at a time to manage memory
  - eggNOG EC/KEGG_Reaction fields can contain multiple values (comma-separated) — must explode
  - **Mass-balance filtering is critical**: all bridge tables must join against `reaction WHERE status = 'OK'` (34,343 reactions) — the user EC→reaction lookup contains bare rxnIDs that include unbalanced reactions; filter on join
  - Many Rhea reactions are unbalanced; Rhea IDs that bridge through EC to an unbalanced ModelSEED reaction should be excluded from the primary mapping (report separately as coverage ceiling)

## Analysis Plan

### Notebook 1: Schema Discovery — DONE
- **Goal**: Characterize all relevant table schemas, enumerate cross-reference types, identify mass-balanced filter, find user DB differences
- **Expected output**: Schema inventory, cross-reference type counts, cached reaction/molecule/reagent TSVs
- **Status**: Complete — all evidence channels characterized, Rhea discovered in `comment_xml`

### Notebook 2: Bridge Tables & RAST Processing
- **Goal**: Build EC→reaction, KEGG R-number→reaction, MetaCyc→reaction bridge tables from user data and `reaction.abbreviation`; all bridges filtered to mass-balanced reactions only (`status = 'OK'`); also report unbalanced coverage ceiling; process 84.5M-row RAST annotation file to extract ECs
- **Expected output**: `ec_to_reaction.parquet`, `kegg_to_reaction.parquet`, `metacyc_to_reaction.parquet`, `rast_protein_ec.parquet` (each filtered to balanced reactions, with unbalanced counts reported)

### Notebook 3: UniProt-Native Evidence (Tier 1)
- **Goal**: Query `identifier` for EC (38.6M) and BRENDA (38.6K); extract Rhea IDs + co-annotated EC from `comment_xml` (330K rows); map all to reactions via bridges
- **Expected output**: Per-protein UniProt-native evidence parquet

### Notebook 4: Pangenome Annotation Evidence (Tier 2)
- **Goal**: Collect EC and KEGG_Reaction from eggNOG (93.6M) and bakta (132.5M); map gene_cluster annotations to reactions
- **Expected output**: Per-gene-cluster annotation evidence parquet

### Notebook 5: Curated & Specialized Evidence (Tiers 3–4)
- **Goal**: Extract evidence from PaperBLAST curatedgene, FitnessBrowser seedclass + besthitmetacyc (MetaCyc rxnId bridge), InterPro→GO→EC chain
- **Expected output**: Per-source evidence parquets

### Notebook 6: Evidence Integration & Scoring
- **Goal**: Combine all evidence channels, assign confidence tiers, compute coverage statistics
- **Expected output**: Integrated mapping table, coverage statistics, evidence contribution analysis

### Notebook 7: RAST Validation
- **Goal**: Compute F1 per evidence channel and combined against RAST ground truth (32.2M protein-EC pairs)
- **Expected output**: Validation results, precision-recall analysis per channel

### Notebook 8: Summary Visualizations
- **Goal**: Generate final data product and publication-quality figures
- **Expected output**: Final mapping table, UpSet plots, coverage heatmaps, confidence tier distribution

### Notebook 9: Evidence-Stratified Validation
- **Goal**: Stratify Tier 1 validation by UniProt data source (Swiss-Prot vs TrEMBL) and protein existence level
- **Expected output**: Per-stratum F1 scores, evidence quality analysis

### Notebook 10: Transport & Unmapped Reaction Analysis
- **Goal**: Quantify transport reaction contribution to unmapped gap; explore text-based matching of UniProt transport proteins; characterize remaining unmapped non-transport reactions by EC gap, database origin, and thermodynamic profile
- **Expected output**: Transport×mapped cross-tabulation, UniProt transport proteins parquet, EC orphan classification, database origin analysis

### Notebook 11: Transport Evidence Integration
- **Goal**: Build transport-specific evidence pipeline using 4 annotation layers (protein names, GO terms, comment_xml, InterPro domains) matched against reagent-molecule substrate vocabularies; assign confidence tiers
- **Expected output**: Per-reaction transport evidence mapping parquet, transport evidence figure, coverage impact analysis

## Expected Outcomes
- **If H1 supported**: Multi-evidence approach achieves >70% reaction coverage with F1 >0.8 — demonstrates the value of systematic evidence integration for metabolic model reconstruction
- **If H0 not rejected**: Coverage remains below 50% or F1 below 0.7 — identifies which evidence channels are most limiting and what additional data sources are needed
- **Potential confounders**: Many-to-many EC→reaction mappings inflate apparent coverage; RAST validation set may not be representative of all reaction types

## Revision History
- **v1** (2026-05-07): Initial plan
- **v2** (2026-05-08): Post-NB01 revision. Restructured evidence channels into 4 priority tiers (UniProt-native first). Added Rhea catalytic activity from `comment_xml` (330K rows, 236K proteins, 13,589 Rhea IDs). Discovered KEGG xrefs are gene IDs not R-numbers, BioCyc xrefs are protein monomers not reactions, RHEA lives in XML not identifier table. User EC→reaction lookup covers 25,757 reactions. RAST validation set is 84.5M rows (32.2M with ECs), not ~2K. Revised notebook plan from 9 to 8 notebooks. De-prioritized FitnessBrowser besthitmetacyc (Tier 3) in favor of UniProt-native data.
- **v3** (2026-05-11): Added NB09 evidence-stratified validation. Tier 1 protein-EC pairs stratified by UniProt data source (Swiss-Prot vs TrEMBL) and protein existence level (experimental, transcript, computational). Swiss-Prot F1=0.890 vs TrEMBL F1=0.837. PE-experimental proteins show lowest F1 (0.769) due to multi-functional annotation complexity. Swiss-Prot alone covers 44.2% of balanced reactions; TrEMBL adds 2,043 reactions via 523 exclusive ECs.
- **v4** (2026-05-12): Added NB10 transport & unmapped reaction analysis. Transport reactions are 17.5% of balanced reactions but 29.2% of unmapped (82.5% unmapped vs 42.5% for non-transport). Text-based matching of UniProt transport proteins to unmapped transport reactions returned 0 candidates — descriptions are too generic. Of 12,038 unmapped non-transport reactions, 90% (10,832) have no EC assigned (true orphans unreachable by any EC-based pipeline); only 10% (1,206) have an EC but no protein annotated. Unmapped reactions dominated by ModelSEED-native entries (45.6%) and reactions with no abbreviation (36.6%), vs mapped reactions which are primarily KEGG-origin (52.0%) or MetaCyc-origin (26.9%).
- **v5** (2026-05-13): Added NB11 transport evidence integration. Multi-layer substrate matching (protein names, GO terms, comment_xml, InterPro domains) against reagent-molecule vocabulary (2,753 substrates across 5,797 transport reactions) recovered 3,903 previously unmapped transport reactions — transport coverage 17.5%→82.5%. 2,597 reactions (56%) supported by all 4 layers; 3,140 high confidence. Total evidence coverage extends from 50.5% to 61.9% (21,254 of 34,343). Remaining unmapped: 13,089 (38.1%), dominated by 10,832 EC orphan non-transport reactions.

## Authors
- Sam Seaver, KBase / Argonne National Laboratory
