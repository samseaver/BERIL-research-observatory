---
reviewer: BERIL Automated Review (Claude, claude-sonnet-4-6)
date: 2026-05-12
project: rosetta
---

# Review: Rosetta — UniProt-to-ModelSEED Reaction Mapping

## Summary

The Rosetta project delivers a rigorous, well-executed investigation into multi-evidence protein-to-reaction mapping for metabolic model reconstruction. The research successfully demonstrates that systematic integration of diverse annotation sources (UniProt, eggNOG, bakta, PaperBLAST, RAST) can achieve 50.5% coverage of mass-balanced ModelSEED reactions with strong validation accuracy (F1 = 0.837 against RAST). The project is methodologically sound, computationally thorough, and properly scoped. Documentation is comprehensive, notebooks are well-organized with saved outputs, and all key findings are supported by evidence. The work makes a concrete contribution to metabolic modeling by quantifying the coverage ceiling of EC-based annotation and providing reusable bridge tables for future reconstruction pipelines.

**Strengths**: Excellent data provenance tracking, proper handling of large-scale joins (4.35B-row identifier table), careful mass-balance filtering throughout, transparent validation design, and honest assessment of limitations. The stratified validation by evidence quality (NB09) is particularly strong.

**Areas for improvement**: Some notebooks could benefit from additional inline documentation of non-obvious SQL logic; the InterPro→GO→EC chain was deferred but could strengthen Tier 4; and the 16,992 unmapped reactions deserve more characterization.

## Methodology

### Research Design

The research question is clear, testable, and scientifically meaningful: "What fraction of UniProt can be reliably mapped to mass-balanced reactions in ModelSEED, and what evidence supports each mapping?" The four-tier evidence hierarchy (UniProt-native → pangenome → curated → indirect) provides a principled framework for prioritizing annotation quality. The use of RAST annotations as validation ground truth is pragmatic given that RAST is the annotation pipeline used by ModelSEED for model reconstruction, though the author correctly acknowledges this measures agreement between automated systems rather than absolute accuracy.

The hypothesis structure is well-designed. H0 (coverage ≤50% or F1 ≤0.7) and H1 (coverage >70% and F1 >0.8) set concrete targets. The results (50.4% coverage, F1=0.837) reject H0 but fail to support H1, which the author interprets correctly as "meaningful coverage with good accuracy, but not reaching ambitious H1 thresholds." This is honest scientific communication.

### Data Sources and Provenance

Data provenance is exemplary. Every evidence channel is traced to a specific BERDL table with row counts, and user-provided files are documented in RESEARCH_PLAN.md with row counts and purposes. The schema discovery notebook (NB01) systematically characterizes 13 UniProt tables, 2 pangenome tables, and 3 FitnessBrowser tables before any analysis begins. The discovery that KEGG xrefs are gene IDs (not R-numbers), BioCyc xrefs are protein monomers (not reactions), and Rhea lives in comment_xml (not identifier) demonstrates careful exploration rather than assumptions from documentation.

The mass-balance filter (`status = 'OK'`, 34,343 of 56,012 reactions) is consistently applied throughout all bridge tables. The author reports unbalanced coverage separately as a ceiling (7,966 EC mappings, 2,402 KEGG, 2,273 MetaCyc excluded) rather than inflating apparent coverage. This is scientifically responsible.

### Reproducibility

Reproduction instructions are complete and specific: access to BERDL, placement of user data files in user_data/, sequential execution of notebooks 01-09. Each notebook is self-contained with saved outputs visible in the repository. All figures referenced in REPORT.md exist in figures/ with timestamps matching the report revision date. The requirements.txt includes all visualization dependencies (upsetplot, matplotlib, seaborn).

One minor gap: the user-provided files (Unique_ModelSEED_Reaction_ECs.txt, uniprot_rast_filtered_mapping.tsv.gz) are not in the repository, which is appropriate given their size (84.5M rows for RAST), but their generation process is not documented. A script or README in user_data/ explaining how to regenerate these would strengthen reproducibility for external users.

### Computational Rigor

Large-scale data handling is competent. The notebooks correctly set `spark.sql.autoBroadcastJoinThreshold = -1` before joining on the 4.35B-row identifier table, avoiding memory issues documented in docs/pitfalls.md. The RAST processing (84.5M rows) uses streaming with gzip.open() and periodic progress prints rather than loading the full dataset into memory. EC and KEGG_Reaction fields are properly exploded when comma-separated (NB04). Memory management is explicit with `del` statements and `gc.collect()` calls after large DataFrames.

SQL queries are generally correct. The Rhea extraction (NB03) uses regex patterns to parse XML without assuming structure. The validation merges (NB07) use outer joins with indicator columns to separate TP/FP/FN, which is the standard approach. One observation: some queries use `COUNT(*)` followed by `.collect()[0]['n']` when `.count()` would be more direct, but this does not affect correctness.

## Code Quality

### SQL and Data Wrangling

SQL queries are functional and mostly readable. Schema discovery queries (NB01) systematically enumerate db values, count distinct entities, and sample rows for each evidence type. Bridge table construction (NB02) correctly filters to balanced reactions and reports excluded rows. The EC extraction from RAST text uses `re.compile(r'(\d+\.\d+\.\d+\.\d+|\d+\.\d+\.\d+\.-|\d+\.\d+\.-\.-|\d+\.-\.-\.-)')` which captures partial EC numbers as intended.

Areas for improvement:

1. **NB01, cell 14**: The identifier db enumeration query groups by db and counts rows/entities, but the output is cached to avoid re-running the 4.35B-row aggregation. The cached CSV is loaded if it exists, but there's no explanation of why the cache was generated or when it should be refreshed. A comment explaining this would help future users.

2. **NB03, cell 3 (Rhea parsing)**: The nested loop over rhea_raw iterates 330K rows and applies regex patterns per row. This is acceptable for this dataset size, but for larger sources a vectorized approach (e.g., pandas str.findall) would be more idiomatic. The current approach works and is transparent, so this is a minor style point.

3. **NB06, cell 5 (confidence assignment)**: The `assign_confidence()` function uses nested if/elif with conditions like `n_tiers >= 3` for high confidence and `has_tier1` for medium. The logic is correct but the tier1-only medium confidence rule could be more explicitly documented inline (e.g., "Tier 1 alone is treated as medium confidence even if only 1 tier due to curation quality").

### Statistical Methods

Precision, recall, and F1 calculations are implemented correctly throughout. The validation framework (NB07) computes per-channel metrics using TP = intersection, FP = left_only, FN = right_only from outer merge indicators, which is standard. The stratified validation (NB09) extends this to data source (Swiss-Prot vs TrEMBL) and protein existence levels, providing meaningful breakdowns.

The EC-level validation (NB07) shows low recall for BRENDA (0.1%) and Rhea (0.8%) despite high precision. The author does not explicitly explain this in the notebook, but the numbers are correct: BRENDA and Rhea annotate far fewer proteins than the full RAST set (27K and 180K vs 30M), so high EC precision does not translate to recall against RAST's 32M protein-EC pairs. This is a legitimate finding but could use a one-sentence interpretation in the notebook.

### Pitfall Awareness

The project demonstrates awareness of BERDL-specific pitfalls:

- **Memory management**: Disabling autoBroadcastJoin before large identifier queries (pitfall documented in docs/pitfalls.md)
- **Mass-balance filtering**: Consistently filtering to `status = 'OK'` throughout (this pitfall is not in docs/pitfalls.md but should be, as unbalanced reactions are a common trap in metabolic modeling)
- **Multivalue fields**: Exploding comma-separated EC and KEGG_Reaction fields in eggNOG/bakta (pitfall documented)
- **Entity ID formats**: Correctly handling `uniprot:ACCESSION` prefixes in identifier table and stripping them for downstream joins

One observation: NB05 defers the InterPro→GO→EC chain with the rationale "requires external ec2go mapping not in BERDL." This is a reasonable decision given that it's Tier 4 (lowest confidence), but the GO consortium's ec2go mapping is publicly available and could have been ingested as user_data. The decision to defer is acceptable but the door is left open for future work.

### Notebook Organization

The 9-notebook structure is logical and well-sequenced:

1. **NB01** (schema discovery): Characterize all tables before making assumptions — excellent practice
2. **NB02** (bridges): Build filtered EC/KEGG/MetaCyc→reaction mappings — properly scoped as reusable artifacts
3. **NB03-05** (evidence tiers): Extract evidence systematically by tier — clear separation of concerns
4. **NB06** (integration): Combine all channels and assign confidence — proper synthesis step
5. **NB07** (validation): Validate against RAST — well-designed with EC/protein/reaction levels
6. **NB08** (visualizations): Generate publication-quality figures — appropriate as final summary
7. **NB09** (stratified validation): Stratify by evidence quality — strong extension addressing reviewer questions

Each notebook has a markdown header cell explaining purpose, inputs, and outputs. Outputs are saved as parquet files in data/ with consistent naming. All notebooks have saved cell outputs, making it possible to review results without re-running queries.

Minor improvement opportunity: NB04 (pangenome evidence) computes eggNOG KEGG_Reaction mappings (35.5M pairs) but does not save them to parquet, only reporting counts. The comment "KEGG gene_cluster pairs not saved locally" explains this, but saving even a summary (e.g., unique gene_cluster→KEGG counts) would be useful for future Tier 2 protein-level integration.

## Findings Assessment

### Are Conclusions Supported?

**Finding 1** (50.5% coverage, high confidence for 42.1%): Supported. Table in NB06 shows 17,351 of 34,343 reactions with evidence, and confidence distribution shows 14,470 high confidence. The UpSet plot (tier_upset.png) visually confirms multi-tier support.

**Finding 2** (Tier 1 dominates, additional tiers confirm): Supported. NB06 shows Tier 1 covers 17,215 reactions (50.1%), Tier 2 adds only 18, and Tier 3 adds 117. The incremental coverage table explicitly reports this. The interpretation that additional tiers provide confirmation rather than expansion is accurate and scientifically valuable.

**Finding 3** (Protein-level F1 = 0.84): Supported. NB07 computes F1 = 0.8372 on 15.95M shared proteins with TP = 14.1M, FP = 2.6M, FN = 3.0M. The precision/recall breakdown is provided. This is a strong result for an automated mapping.

**Finding 4** (Coverage is the bottleneck): Supported. The F1 threshold (0.8 > 0.7) was met but coverage (50.4%) fell short of H1's 70% target. The 16,992 unmapped reactions are explicitly identified. The interpretation that this reflects the current state of enzyme annotation rather than mapping methodology failure is reasonable, though it could be strengthened by characterizing the unmapped set (see suggestions below).

**Finding 5** (Error patterns reveal specificity issues): Supported. NB07 error analysis shows top FP ECs include partial numbers (7.1.1.2, 5.4.99.-, 2.6.1.-) and top FN ECs include large families (1.6.5.3 NADH dehydrogenase, 3.6.3.14 ABC transporters). The interpretation of specificity mismatches (UniProt precise vs RAST broad, or vice versa) is plausible.

**Finding 6** (Swiss-Prot more accurate, computational proteins dominate): Supported. NB09 shows Swiss-Prot F1 = 0.890 vs TrEMBL F1 = 0.837 (+5.3 pp), and Swiss-Prot covers 44.2% of reactions with only 218K proteins while TrEMBL's 26.3M proteins add 2,043 reactions via 523 exclusive ECs. The PE-experimental F1 = 0.769 being lowest is counterintuitive but explained as multifunctional annotation complexity, which is a thoughtful interpretation.

All six major findings have direct supporting evidence from notebook outputs, and the tables/figures cited in REPORT.md exist and match the claims.

### Limitations Acknowledged?

The author explicitly lists five limitations in REPORT.md:

1. **Protein-level validation limited to Tier 1**: Acknowledged. Tier 2 uses gene_cluster_id and Tier 3 uses locus IDs, so cross-entity protein-level F1 is not computable. This is an inherent limitation of the data structure, not a methodological flaw.

2. **EC number granularity**: Acknowledged. Many-to-many EC→reaction mappings and partial EC numbers inflate coverage and create FP/FN mismatches. This is a known issue in metabolic modeling and is properly flagged.

3. **RAST as ground truth**: Acknowledged. RAST is itself automated, not experimentally verified. The author justifies RAST as "pragmatically valid" since it's the ModelSEED annotation pipeline, which is reasonable.

4. **No direct Rhea→reaction bridge**: Acknowledged. The 33.8K Rhea-only proteins (without co-annotated EC) are identified as a missed opportunity. This is a concrete future direction.

5. **Coverage is EC-centric**: Acknowledged. The author notes that sequence similarity, structural homology, and substrate docking could potentially reach the unmapped 49.5%, and lists this as future work.

These are honest, scientifically appropriate acknowledgments. One additional limitation not explicitly stated: the bridge tables (EC/KEGG/MetaCyc→reaction) are user-provided rather than derived directly from BERDL. The provenance of Unique_ModelSEED_Reaction_ECs.txt (25,757 reactions, 7,351 ECs) is not documented, though it's likely from the ModelSEED database itself. Documenting this source would strengthen transparency.

### Incomplete Analysis?

One gap: the 16,992 unmapped reactions (49.5%) are identified but not characterized. The author suggests they may include (1) orphan reactions, (2) reactions with known enzymes lacking EC, and (3) reactions in poorly studied pathways. This is plausible but speculative. Future work should include:

- Pathway membership analysis: Are unmapped reactions clustered in specific metabolic pathways (e.g., secondary metabolism, xenobiotic degradation)?
- Organism distribution: Are they more common in specific taxonomic groups?
- Overlap with known orphan enzyme lists (e.g., from OrphanEnzyme.org or MetaCyc)
- Structural features: Are they more likely to be transport reactions, spontaneous reactions, or multi-step transformations?

This analysis would help prioritize experimental characterization efforts and is a natural extension of the current work. The current report lists "Characterize unmapped reactions" as future direction (item 3), which acknowledges the gap.

## Suggestions

### 1. Document user data provenance and regeneration

**Priority: High**  
**Rationale**: The two user-provided files (Unique_ModelSEED_Reaction_ECs.txt and uniprot_rast_filtered_mapping.tsv.gz) are critical to the analysis but their generation is not documented. External users cannot regenerate the mapping without these files.

**Action**: Add a README.md or PROVENANCE.md to user_data/ explaining:
- Source of Unique_ModelSEED_Reaction_ECs.txt (likely ModelSEED database dump; provide URL or SQL query)
- Generation of uniprot_rast_filtered_mapping.tsv.gz (RAST annotation export; provide script or API call)
- Versions/timestamps of these datasets

Alternatively, if these files are proprietary or too large to share, document the process so collaborators with database access can regenerate them.

### 2. Characterize the 16,992 unmapped reactions

**Priority: Medium**  
**Rationale**: Identifying what's missing is as scientifically valuable as quantifying what's mapped. The unmapped reactions represent the frontier for enzyme discovery and annotation.

**Action**: Add NB10 (or extend NB06) to analyze unmapped reactions:
- Query `reaction.name` patterns (e.g., grep for "transporter", "spontaneous", "uncharacterized")
- Check overlap with known orphan enzyme lists if available
- Compute substrate/product stoichiometry complexity (e.g., number of reagents, presence of cofactors)
- Sample 50-100 unmapped reactions and manually categorize as orphan/transport/other

This would strengthen the discussion of why coverage plateaus at 50%.

### 3. Save eggNOG KEGG_Reaction mappings

**Priority: Low**  
**Rationale**: NB04 computes 35.6M gene_cluster→KEGG_Reaction pairs but does not save them, only reporting counts. While this is acceptable for reaction-level coverage, saving them would enable future Tier 2 protein-level analysis.

**Action**: Save the eggNOG KEGG mappings to `pangenome_gc_kegg.parquet` or at minimum save a summary table of unique gene_cluster→KEGG counts. If space is a concern, save only mappings where KEGG_Reaction is in bridge_keggs.

### 4. Add inline comments to complex SQL queries

**Priority: Low**  
**Rationale**: Some SQL queries (especially multi-table joins in NB01 and NB04) could be clearer with inline comments explaining the logic.

**Examples**:
- NB01, cell 8 (reaction abbreviation sampling): The UNION ALL query samples 4 patterns (kegg_r, ec_like, metacyc, other). Adding comments explaining why these patterns matter would help readers unfamiliar with ModelSEED.
- NB04, cell 3 (bakta EC union): The union of bakta_annotations.ec and bakta_db_xrefs combines two sources. A comment explaining why both are needed (e.g., "bakta stores EC in both annotation text and structured xrefs") would clarify.

### 5. Explore the InterPro→GO→EC chain

**Priority: Low**  
**Rationale**: NB05 defers this with "requires external ec2go mapping not in BERDL." The GO consortium's ec2go file is publicly available and small (typically <10K mappings). Including this would complete Tier 4 and quantify the low-confidence indirect evidence.

**Action**: Download ec2go from http://geneontology.org/external2go/ec2go, parse to extract GO→EC mappings, join interproscan_go (266M gene_cluster→GO pairs) to ec2go, then map to reactions. Report coverage but flag as Tier 4 (low confidence). Even if this adds zero new reactions (due to overlap with Tier 1-3), documenting the null result is scientifically valuable.

### 6. Validate Tier 2/3 at gene cluster level

**Priority: Low**  
**Rationale**: Currently only Tier 1 has protein-level validation because RAST uses UniProt IDs. However, RAST annotations could be mapped to gene clusters via genome assemblies, enabling Tier 2 validation.

**Action**: If genome assembly → gene_cluster mappings exist in BERDL (check kbase_ke_pangenome tables), map RAST annotations to gene clusters and compute Tier 2 F1. This would answer whether pangenome EC annotations transfer correctly across genomes. If mappings don't exist, document this as a limitation and suggest it as future work.

### 7. Add a "Quick Start" section to README.md

**Priority: Low**  
**Rationale**: The current README is concise but assumes familiarity with BERDL. A 3-4 sentence quick start ("clone repo, place user_data files, run notebooks 01-08 in order") would help new users.

**Action**: Add a "Quick Start" subsection to README.md before "Reproduction" with example commands:
```
git clone <repo>
cd projects/rosetta
# Place user_data files (see RESEARCH_PLAN.md)
jupyter notebook notebooks/01_schema_discovery.ipynb
# ... run 01-08 in sequence
```

## Review Metadata

- **Reviewer**: BERIL Automated Review (Claude, claude-sonnet-4-6)
- **Date**: 2026-05-12
- **Scope**: README.md, RESEARCH_PLAN.md, REPORT.md, references.md, requirements.txt, 9 notebooks (01-09), data/ (15 files), figures/ (6 files), docs/pitfalls.md
- **Note**: This review was generated by an AI system. It should be treated as advisory input, not a definitive assessment. The reviewer has examined all project documentation, notebook code and outputs, and generated data artifacts, but has not independently re-executed the analysis or verified external data sources.

---

## Final Assessment

The Rosetta project is scientifically sound, computationally rigorous, and well-documented. The research delivers on its promise to systematically quantify multi-evidence protein-reaction mapping and provides reusable artifacts (bridge tables, scored mappings) for the metabolic modeling community. The author demonstrates strong judgment in prioritizing evidence quality, properly scoping validation, and honestly reporting limitations. The work is publication-ready with minor revisions addressing the suggestions above.

**Recommended next steps**: (1) Document user data provenance, (2) characterize unmapped reactions, (3) submit findings to a metabolic modeling journal or conference (e.g., ISME, MSB, or Bioinformatics), (4) consider depositing bridge tables and evidence integration summary to Zenodo or FigShare for community reuse.
