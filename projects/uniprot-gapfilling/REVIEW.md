---
reviewer: BERIL Automated Review (Claude, claude-sonnet-4-6)
date: 2026-05-20
project: uniprot-gapfilling
---

# Review: UniProt-Guided Gap-Filling via Enzyme Embeddings

## Summary

This is a well-executed prototype that tests whether ESM-2 protein embeddings can discriminate biologically plausible metabolic gap-filling candidates from implausible ones in *E. coli* models. The project runs a complete seven-notebook pipeline — from gap-fill reaction characterization through embedding retrieval, similarity scoring, and statistical evaluation — and arrives at a clear, honest null result: pretrained ESM-2 cosine similarity does not provide discriminative gap-filling evidence after appropriate statistical correction. The strongest methodological contribution is the self-correcting NB05→NB05b arc: the initial analysis discovered an apparent paradox (proxy reactions outscoring direct reactions), which NB05b rigorously diagnosed as a max-of-N order statistic artifact and corrected with pool-size adjustment, permutation null modeling, and cross-reaction specificity z-scoring. That diagnostic and correction framework is genuinely reusable and is the paper-worthy output of this project even more than the null result itself. Gaps worth addressing: one notebook cell (NB02's proxy loop) has no visible text output, the gene ID interpretability limitation for the single actionable reaction (rxn04657) is documented but unresolved, and the background calibration uses enzyme-biased Swiss-Prot proteins rather than truly random sequences — which the REPORT acknowledges but understates in terms of impact on corrected verdict interpretation.

## Methodology

**Research question**: Clearly stated and testable as H0/H1. The question — whether ESM-2 cosine similarity can discriminate same-reaction enzyme candidates from cross-reaction proteins — is well-scoped for a prototype, and the null result is as informative as a positive.

**Approach soundness**: The pipeline logic is correct. Rosetta mappings establish curated reaction-to-UniProt links; FAISS enables efficient cosine search across 48 genomes; rank-based percentile scoring was the right pivot once NB03b revealed a 0.013 median separation in absolute cosine space. The progression from raw scoring (NB04–NB05) to corrected scoring (NB05b) reflects good scientific reflexes rather than a flaw in study design.

**Data sources**: Clearly identified in README, RESEARCH_PLAN, and REPORT. The Rosetta project dependency is documented with explicit prerequisite instructions. The NB02 Spark query against `kbase_msd_biochemistry.reaction_similarity` is the only BERDL-dependent step beyond the Rosetta parquets, and the README correctly calls out which notebooks can run locally from cached data (`user_data/proxy_candidates.parquet`, `user_data/candidate_embeddings.json`).

**Scope appropriateness**: Limiting to Swiss-Prot (3,168 unique proteins after proxy expansion) for the prototype is well-justified in the RESEARCH_PLAN. The decision is documented with an F1 rationale (Swiss-Prot 0.89 vs TrEMBL 0.84), though that number is asserted without a citation to where it comes from in the Rosetta project outputs.

**Minor inconsistency**: NB01's partition summary reports "No Rosetta evidence: 14 reactions" but NB02 section 1 prints "No Rosetta evidence (need proxy): 16." The code in NB02 correctly adds `evidence_no_sp` and `missing` reactions to the proxy search set, so the behavior is right — but the label breaks continuity from NB01 and would confuse a reader expecting consistent counts.

## Code Quality

**SQL correctness**: The proxy reaction query in NB02 correctly uses `CAST(similarity AS FLOAT)` when filtering `reaction_similarity`, consistent with the pitfalls doc on string-typed numeric columns. The reaction ID format `seed.reaction:{rxn_id}` matches the BERDL schema. No `SELECT DISTINCT` + aggregate issues are present. BERDL pitfall coverage is clean overall; no pangenome or fitness-browser pitfalls are triggered because the project doesn't touch those schemas.

**Statistical methods**: The pool-size correction (`corrected = (p/100)^N * 100`) is the right transformation for the maximum of N uniform draws. The permutation test uses a well-designed exclusion set (real proteins excluded from null draws), seeds for reproducibility (`np.random.default_rng(42)`), and a conservative +1 numerator. The cross-reaction specificity z-score is a sensible third signal, though with 32 reactions and nearly identical real-best-cosine values (most ≥ 0.999), the z-score denominator (cross-rxn std ≈ 0.011) makes the measure nearly meaningless at the top of the distribution — all high-similarity reactions cluster within ±1 z of each other. This is correctly reported (zero reactions achieve z > 1), but the `cross_reaction_specificity.png` panel 1 axis could mislead a reader without an annotation.

**Notebook organization**: Excellent. Each notebook has a clear header markdown cell stating inputs, outputs, and purpose. The flow setup → query → analysis → visualization → save is consistent throughout. Cell text outputs are comprehensive and well-formatted across all notebooks.

**NB02 proxy loop (cell d1)**: The proxy search loop iterates over `proxy_rxns` and is expected to print per-reaction candidate counts, but this cell has no saved text output in the notebook. Every other output-producing cell in the pipeline has printed results captured. This is the only gap in output completeness.

**Background distribution design (NB04)**: The 200-protein genome background per genome is drawn from the Swiss-Prot candidate pool itself — enzyme sequences, not random proteins. This is acknowledged in the REPORT Limitations section as "likely too generous," but the downstream effect matters: the corrected verdicts already show 23/32 reactions as "insufficient." If the background were tighter (true random *E. coli* proteins), the picture would be even more null. The REPORT's phrasing could be stronger: the corrected medians (3.4 for high-evidence, 0.7 for no-evidence) may be *upper bounds* on the true signal.

## Findings Assessment

**Finding 1 (Rosetta coverage expansion)**: Well-supported. Numbers from NB01 and NB02 are internally consistent: 25 reactions with direct Swiss-Prot candidates, 7 additional via proxy, 10 uncovered. The comparison against the original flat-lookup (12 reactions → 32) matches NB05's output exactly.

**Finding 2 (ESM-2 compression)**: Well-supported and quantified. The 0.013 median separation (same-reaction vs cross-reaction) is computed over 1.25M pairs — a large enough sample. The candidate-vs-genome top-1 median of 0.975 correctly identifies the failure mechanism: any Swiss-Prot enzyme finds a near-perfect match in an *E. coli* genome regardless of functional specificity.

**Finding 3 (pool-size artifact)**: The strongest finding in the project. The expected-max formula and the forensic comparison (direct excess = −16.3 percentile points, proxy excess = −2.5) cleanly diagnose the problem. The insight that "direct reactions score *below* null while proxy reactions sit near null" is the key diagnostic that makes NB05b's correction principled rather than post-hoc.

**Finding 4 (permutation null)**: The result that only 2/32 reactions achieve p < 0.05 is credible. rxn04660 is significant at p = 0.001 but has specificity_z < 1, so it does not clear the moderate-evidence bar. rxn04657 (N=1 candidate, p = 0.028) retains moderate status precisely because there is no pool-size inflation with a single candidate — that is the correct interpretation and the REPORT makes it clearly.

**Finding 5 (H0 not rejected)**: Clearly stated and supported by the data. The verdict comparison table (28 demotions after correction) is the key output. The REPORT correctly frames this as an informative null result with specific failure mode characterized, not as a pipeline failure.

**NB05 intermediate verdict**: NB05 section 6 concludes "H1 SUPPORTED: 16/32 reactions have moderate-to-strong embedding evidence" based on raw, uncorrected scores. A markdown caveat cell at the top of that section flags NB05b as superseding this verdict, but the cell *output* prints "H1 SUPPORTED" with no qualification. A reader running only NB05 would see the wrong conclusion in the output. Appending a corrective print statement to that cell would resolve this.

**Gene ID interpretability for rxn04657**: NB03 sanity check shows FAISS hits return ordinal indices (e.g., `1986`, `1739`) rather than locus names. The RESEARCH_PLAN documents this as a known limitation. For rxn04657 — the single actionable reaction — the "best-hit gene" is an ordinal index that cannot be biologically interpreted without parsing the corresponding `.faa` FASTA to map ordinal position → locus tag. Given that rxn04657 is the headline result, the REPORT should include at least a brief resolved identifier.

**Limitations section**: Thorough and honest. Single-organism scope, Swiss-Prot-only candidates, mean-pooled embeddings, and biased background calibration are all identified. The Future Directions section is specific and actionable (fine-tuned models, alignment-based scoring, per-residue attention, reaction fingerprints).

## Suggestions

1. **[Critical] Capture NB02 proxy loop output** — Re-run cell d1 and save the notebook so the per-reaction proxy candidate counts are visible in the output. Without output, a reader cannot verify which reactions got proxy candidates or how many. This is the only cell in the pipeline with missing text output.

2. **[High] Resolve the gene ID for rxn04657** — Add a short code block (either appended to NB05b or in the REPORT) that parses `user_data/ecoli-files/219790_10_1-protein.faa` to map ordinal FAISS index → locus tag for the best-hit gene. A one-liner is sufficient: `locus = [r.id for r in SeqIO.parse(faa_path, 'fasta')][ordinal_idx]`. This turns the single actionable result from "ordinal index in FAISS" into a named gene, making the output biologically interpretable.

3. **[High] Quantify the background calibration bias** — The REPORT acknowledges the enzyme-biased background may over-estimate scores. Add a quick validation in NB03b or NB05b: draw a second 200-protein background from a random sample of the *E. coli* genome proteome (available in `ecoli-files/`) and compare the resulting percentile-rank distribution. Even a two-panel figure showing Swiss-Prot background vs random-protein background would establish whether the bias materially changes the corrected verdicts or merely adjusts the magnitudes.

4. **[Medium] Fix the NB01→NB02 count label** — NB02 section 1 prints "No Rosetta evidence (need proxy): 16" but should read something like "Reactions needing proxy search: 16 (14 no-evidence + 1 evidence-no-SP + 1 missing)" to match NB01's partition output and prevent confusion about why the count changed.

5. **[Medium] Add a corrective print to NB05 section 6** — The hypothesis evaluation cell prints "H1 SUPPORTED" based on raw scores. Append a `print()` line: `"NOTE: NB05b supersedes this verdict — after pool-size correction, H0 is not rejected."` This ensures the correct final conclusion appears in the cell output, not only in the markdown caveat above it.

6. **[Medium] Source the Swiss-Prot F1 claim in RESEARCH_PLAN** — The statement "Swiss-Prot proteins showed higher validation accuracy in Rosetta (F1 0.89 vs TrEMBL 0.84)" should cite the specific Rosetta notebook or data file that supports it. Without a cross-reference, this number is unverifiable by a reader.

7. **[Low] Annotate the cross-reaction specificity figure** — `figures/cross_reaction_specificity.png` panel 1 shows all reactions clustered between z = −4 and z = +1, with reference lines at z = 1 and z = 2 at the top edge of the visible range. A note that the top cluster (z ≈ 0.99) reflects floating-point saturation (cosine → 1.000) rather than genuine specificity would prevent over-interpretation.

8. **[Low] Confirm `gapfill_landscape.csv` is not gitignored** — NB01 saves this file to `user_data/` and NB02 reads it as its first input. If `.gitignore` excludes `user_data/*.csv`, a fresh runner would need to run NB01 before NB02, which is already documented in the README but worth double-checking against `.gitignore`.

## Review Metadata
- **Reviewer**: BERIL Automated Review (Claude, claude-sonnet-4-6)
- **Date**: 2026-05-20
- **Scope**: README.md, RESEARCH_PLAN.md, REPORT.md, 7 notebooks (NB01–NB05b), 12 figures, requirements.txt, docs/pitfalls.md
- **Note**: This review was generated by an AI system. It should be treated as advisory input, not a definitive assessment.
