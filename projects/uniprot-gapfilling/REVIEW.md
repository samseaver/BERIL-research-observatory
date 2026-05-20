---
reviewer: BERIL Automated Review (Claude, claude-sonnet-4-6)
date: 2026-05-20
project: uniprot-gapfilling
---

# Review: UniProt-Guided Gap-Filling via Enzyme and Reaction Embeddings

## Summary

This is a well-executed prototype study that uses ESM-2 protein embeddings and Rosetta UniProt-to-ModelSEED mappings to evaluate whether sequence-level similarity can guide metabolic gap-filling in *E. coli* models. The project's most notable feature is rigorous self-correction: NB05's initial "H1 SUPPORTED" verdict was identified as a pool-size artifact and corrected in NB05b using pool-size-adjusted percentile ranks, permutation null models, and cross-reaction specificity z-scores. The final REPORT.md accurately reflects the corrected finding (H0 not rejected). All seven notebooks have saved text and table outputs, 12 figures are present in `figures/`, and the data flow is clearly documented. However, the project has several reproducibility gaps: the `## Reproduction` section of README.md was never filled in despite the project being marked "Complete," `requirements.txt` is missing most of its actual dependencies, and NB05 retains a printed "H1 SUPPORTED" conclusion that directly contradicts the corrected finding in NB05b and the report.

---

## Methodology

**Research question**: Clearly stated and testable. The H0/H1 distinction is explicit in both README.md and RESEARCH_PLAN.md.

**Approach**: Sound for prototyping. The data flow diagram in RESEARCH_PLAN.md accurately describes the actual pipeline. The decision to use Swiss-Prot only for the prototype (1,140 proteins vs 222K TrEMBL) is justified by the F1 score comparison and keeps retrieval tractable.

**Scope definition**: Well-bounded — 42 reactions, 48 *E. coli* genomes, Swiss-Prot candidates only. Limitations of this scope (single organism, one embedding model, mean-pooled representations) are all acknowledged in the report.

**Data sources**: Clearly identified. Cross-database dependencies (Rosetta parquet files at `projects/rosetta/data/`, BERDL Spark tables) are documented in RESEARCH_PLAN.md. There is no guard in NB01/NB02 to verify that the Rosetta parquet files exist before running, so a user attempting to reproduce without first running the Rosetta project would see a silent `FileNotFoundError`.

**Literature context**: Present and appropriate in REPORT.md. However, the RESEARCH_PLAN.md literature section still reads "*TBD — to be expanded with literature review*" even though the project is marked complete. This creates a gap for readers trying to understand the original design rationale.

**Reproducibility of design decisions**: The choice of 200 background proteins for the genome background distribution (NB04) is noted as a limitation in the report. The cross-genome standard deviation of cosine similarity (0.0006) is empirically confirmed in NB04 and used to justify single-genome permutation testing in NB05b — a defensible and well-documented choice.

---

## Code Quality

**Notebook organization**: Logical and clean. The NB01→NB02→NB03→NB03b→NB04→NB05→NB05b progression flows naturally: characterization → candidate assembly → retrieval → calibration → scoring → evaluation → correction. Each notebook documents its inputs and outputs in its opening markdown cell.

**SQL correctness**: NB02, cell d1 queries `kbase_msd_biochemistry.reaction_similarity` with `WHERE similarity > 0.7`. The pitfalls document (`docs/pitfalls.md`, "String-Typed Numeric Columns") warns that numeric columns in BERDL are often stored as strings and require explicit `CAST`. No `CAST(similarity AS FLOAT)` is used. The query produces correct output — suggesting the column may be properly typed in this table — but the assumption is not verified and the documented pitfall was not explicitly addressed.

**Spark session import**: NB02 correctly uses `from berdl_notebook_utils.setup_spark_session import get_spark_session` (the on-cluster explicit import pattern). Remaining notebooks use only local computation.

**Statistical methods**: Appropriate and rigorous. The pool-size correction in NB05b (CDF of max of N uniform draws) is mathematically correct. The 1,000-permutation test is sufficient for 32 reactions. The cross-reaction specificity z-score is a valid complementary measure.

**Misleading terminal output in NB05**: NB05, cell g1 prints:

```
--- Verdict ---
H1 SUPPORTED: 16/32 scored reactions have moderate-to-strong
embedding evidence.
```

This is the pre-correction finding, subsequently shown in NB05b to be spurious (only 1/32 reactions are actionable after correction). Since NB05 is a committed notebook with saved outputs, a reader examining it in isolation will reach an incorrect conclusion.

**Gene ID resolution in NB03/NB04**: In NB03, cell g1, top-5 genome hit IDs print as integers (1986, 1739, 3199, etc.) rather than gene identifiers, because the `.results.json` files for the pre-computed genome indexes store entries that are not Python dicts with a `'query_id'` key. The fallback `str(top1_idx)` in NB04 means the `gene_id` column in `gapfill_scores.parquet` and `gapfill_best_hits.parquet` contains raw index integers rather than meaningful gene identifiers for these entries. This does not affect similarity scores or scientific conclusions, but it limits interpretability of "which *E. coli* gene matches best."

**FutureWarning**: NB02, cell e1 emits a pandas `FutureWarning` about DataFrame concatenation with empty or all-NA entries. Non-critical but should be resolved before expanding the pipeline.

**Pitfalls compliance**: The project works within BERDL correctly — correct `get_spark_session()` import, batch-sized `.toPandas()` calls on small result sets (7 batches of ~500 rows), no unnecessary full-table scans. The `reaction_similarity` CAST issue is the only potential gap relative to documented pitfalls.

---

## Findings Assessment

**Finding 1 (Rosetta coverage expansion, 29% → 76%)**: Supported by NB01/NB02 outputs. The exact numbers in the report (24 high-confidence reactions, 25 direct Swiss-Prot candidate sets, 7 proxy-only, 10 uncovered) match NB01 and NB02 cell outputs precisely.

**Finding 2 (ESM-2 compression, 0.013 separation)**: Supported by NB03b output. The three-distribution baseline table is saved to `user_data/similarity_baselines.csv` and reproduced verbatim in the report. The conclusion that absolute cosine thresholds are unreliable is well-supported.

**Finding 3 (pool-size artifact)**: Supported by NB05b cell b1. The expected null values match the `100 * N / (N + 1)` formula numerically, and the "excess" analysis (direct excess: −16.3, proxy excess: −2.5) correctly diagnoses that proxy reactions land near — not above — the null expectation.

**Finding 4 (permutation test, 2/32 significant)**: Supported by NB05b cell d2. Only rxn04660 (p = 0.001) and rxn04657 (p = 0.028) pass p < 0.05. The report's claim that "rxn04657 (1 candidate, p = 0.028)" is the single actionable reaction after correction is confirmed by NB05b cell f1 (corrected verdict: moderate for rxn04657 only, reflecting its N=1 pool which has no pool-size inflation).

**Finding 5 (H0 not rejected)**: Supported by NB05b's corrected verdict table. The report's before/after comparison table (strong: 10→0, moderate: 6→1, etc.) is accurate.

**Internal inconsistency**: The REPORT.md narrative correctly concludes H0 is not rejected. However, the saved NB05 output prints "H1 SUPPORTED" — see Code Quality above. This is a correctness issue in the notebook record, not in the report itself, but a reader auditing the notebooks directly would be misled.

**Limitations**: Acknowledged clearly and specifically (single organism, Swiss-Prot only, single embedding model, mean-pooled representations, 200-protein background calibration, no TrEMBL coverage). The limitation around mean pooling ("loses active-site-level information") is particularly well-framed given the known ESM-2 architecture.

**Future directions**: Specific, concrete, and well-grounded in findings. The recommendation to use alignment-based scoring (gapseq-style) as a comparison is appropriate given the calibration results showing 0.013 median separation in ESM-2 space.

---

## Suggestions

1. **Fill in the `## Reproduction` section of README.md** *(critical — blocking reproducibility)*. The section currently reads "*TBD — add prerequisites and step-by-step instructions after analysis is complete*," but the project is marked complete. At minimum document: (a) which notebooks require Spark (NB02 only) vs. run locally from cached parquets, (b) that NB01–NB02 depend on the Rosetta project's parquet outputs in `projects/rosetta/data/`, (c) that NB03 requires a live `llm_homology_api` connection or the cached `user_data/candidate_embeddings.json`, and (d) expected runtimes for the API call step in NB03 and the FAISS scoring loop in NB04.

2. **Expand `requirements.txt`** *(critical — blocking reproducibility)*. The current file lists only `pandas`, `numpy`, `matplotlib`. Notebooks also require: `faiss-cpu` (NB03–NB05b), `httpx` (NB03), `scipy` (NB04 imports `from scipy.stats import percentileofscore`), `pyarrow` (all parquet I/O), and `berdl_notebook_utils` (NB02). Even if `berdl_notebook_utils` requires a non-PyPI install path, it should be listed with a note.

3. **Add a supersession note to NB05's hypothesis cell** *(significant)*. NB05 cell g1 prints "H1 SUPPORTED: 16/32 scored reactions have moderate-to-strong embedding evidence." NB05b later disproves this. Add a markdown cell immediately before or after the hypothesis section in NB05 stating that NB05b supersedes this verdict and corrects the pool-size bias. Alternatively, prepend the `print` statement text with "NB05 interim verdict (superseded by NB05b):" to make the status clear in the saved output.

4. **Update the Literature Context in RESEARCH_PLAN.md** *(moderate)*. The section reads "*TBD — to be expanded with literature review*." Since the analysis is complete and REPORT.md contains a solid literature context (8 cited references), either copy the key references into RESEARCH_PLAN.md or add a cross-reference: "*See REPORT.md §Interpretation / Literature Context*." A plan with a blank literature section misrepresents the project's prior art basis.

5. **Add a Rosetta dependency guard to NB01** *(moderate)*. NB01 cell c1 loads `ROSETTA / 'evidence_integration_summary.parquet'` without checking whether the file exists. Add an assertion at the top of NB01:
   ```python
   assert (ROSETTA / 'evidence_integration_summary.parquet').exists(), \
       "Run the Rosetta project first (projects/rosetta/) to generate parquet outputs."
   ```
   This gives a clear error to a reproducer who hasn't run the upstream project.

6. **Explicitly CAST similarity in NB02's proxy search SQL** *(moderate)*. Replace:
   ```sql
   WHERE reaction_1 = '{db_rxn_id}' AND similarity > 0.7
   ```
   with:
   ```sql
   WHERE reaction_1 = '{db_rxn_id}' AND CAST(similarity AS FLOAT) > 0.7
   ```
   This addresses the documented BERDL pitfall for string-typed numeric columns. The query appears to work currently, but explicit casting is defensive and documents intent for future readers.

7. **Document or fix the `.results.json` gene ID format** *(minor)*. The pre-computed genome FAISS indexes in `user_data/ecoli-files/` have associated `.results.json` files that appear to store non-dict entries, causing NB03 and NB04 to fall back to raw integer indices for gene IDs. Either document the expected format of these files (in a comment or the RESEARCH_PLAN), or fix the ID resolution code in NB03 cell g1 / NB04 cell 7 to correctly extract gene identifiers. The `gene_id` column in the output parquets currently contains integers rather than gene names for most records.

8. **Fix the FutureWarning in NB02** *(minor)*. NB02 cell e1 warns about DataFrame concatenation with all-NA entries. Resolve by filtering empty DataFrames before concat:
   ```python
   frames = [df for df in [direct_candidates, proxy_df] if not df.empty]
   all_candidates = pd.concat(frames, ignore_index=True)
   ```

---

## Review Metadata

- **Reviewer**: BERIL Automated Review (Claude, claude-sonnet-4-6)
- **Date**: 2026-05-20
- **Scope**: README.md, RESEARCH_PLAN.md, REPORT.md, 7 notebooks (NB01–NB05b), requirements.txt, 12 figures, docs/pitfalls.md
- **Note**: This review was generated by an AI system. It should be treated as advisory input, not a definitive assessment.
