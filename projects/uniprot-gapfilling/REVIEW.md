---
reviewer: BERIL Automated Review (Claude, claude-sonnet-4-6)
date: 2026-05-20
project: uniprot-gapfilling
---

# Review: UniProt-Guided Gap-Filling via Enzyme and Reaction Embeddings

## Summary

This is a well-executed prototype that demonstrates genuine methodological maturity and scientific honesty. The research question is clearly stated, the hypothesis is falsifiable, and the seven-notebook pipeline flows logically from landscape characterization through calibration, scoring, and statistical correction. The most impressive element is NB05b: the recognition and correction of a max-of-N order statistic artifact that would otherwise have produced a false-positive conclusion (H1 supported). That self-correction — including pool-size correction, a 1,000-permutation null model, and cross-reaction specificity z-scores — elevates this from a routine prototype to a methodologically careful study whose negative result is genuinely informative. The REPORT clearly and accurately states H0 is not rejected. Coverage is good: all 12 expected figures are present, all notebooks have saved outputs, and a `requirements.txt` and `Reproduction` section exist. The main gaps are a stale cell output in NB01, an unacknowledged selection bias in the background calibration pool, and a narrative inconsistency between the NB05b verdict print and the final README/REPORT conclusion.

---

## Methodology

**Research question and hypothesis.** Both are clear and testable. H0/H1 are stated precisely in the RESEARCH_PLAN, the analysis tests them with quantitative criteria, and the final verdict ("H0 not rejected") is correctly rendered in the README and REPORT.

**Approach.** The data flow is well-designed: Rosetta parquets → Swiss-Prot candidate assembly → ESM-2 embeddings via `llm_homology_api` → FAISS cosine search → percentile ranking → pool-size-corrected evaluation. The proxy fallback using `reaction_similarity > 0.7` for uncovered reactions is reasonable. The cross-genome consistency check (cosine std = 0.0006 across 48 genomes) is a good sanity check that justifies collapsing to one representative genome for permutation testing.

**Scope.** The prototype is appropriately scoped: 42 reactions, 48 *E. coli* genomes, Swiss-Prot only. Limitations (single organism, Swiss-Prot only, mean-pooled embeddings) are clearly listed in both the RESEARCH_PLAN and REPORT.

**Background calibration pool — unacknowledged selection bias.** The percentile rank background in NB04 is built from 200 randomly sampled proteins drawn from the *candidate pool* (3,168 Swiss-Prot enzymes), not from unrelated random protein sequences. Because Swiss-Prot proteins are reviewed, high-quality enzyme sequences, they cluster in a compressed region of ESM-2 space alongside the gap-fill candidates. The resulting background distribution may be *too generous* (already biased toward high similarity), which means the percentile rank scores are potentially less conservative than a truly null background would produce. The REPORT acknowledges that "200 random proteins per genome for background distribution may undersample the tail" but does not note the pool-selection bias. This is worth documenting explicitly, especially because the entire corrected-verdict framework rests on this background distribution.

**Reproducibility.** The README's `## Reproduction` section describes all steps, notes which notebooks require Spark, lists expected runtimes, and documents cached data alternatives. This is well done.

---

## Code Quality

**SQL correctness.** The `reaction_similarity` query correctly applies `CAST(similarity AS FLOAT) > 0.7` and uses the `seed.reaction:{rxn}` ID format. No SQL reserved-word or column-name pitfalls from `docs/pitfalls.md` are triggered.

**Statistical methods.** The scoring progression — from raw cosine → percentile rank → pool-size-corrected rank → permutation p-value → specificity z-score — is sound. The permutation p-value uses the standard `(count + 1) / (N + 1)` continuity correction. The pool-size correction formula `(p/100)^N * 100` is the correct CDF of the max-of-N uniform order statistic. The cross-reaction specificity z-score is straightforward and correctly computed.

**Stale print output in NB01.** The setup cell sets `DATA_OUT = Path('..') / 'user_data'`, but the output in the summary-table cell reads `Saved to ../data/gapfill_landscape.csv`. Since `Path('..') / 'user_data'` would print as `../user_data/...`, this output must have been generated when `DATA_OUT` pointed to `'data'` — indicating stale outputs from an earlier code version. The actual file lands in `user_data/` (NB02 reads it successfully and the REPORT correctly cites `user_data/gapfill_landscape.csv`), so the behavior is correct, but the stale printout is misleading for readers tracing data provenance.

**Unused import.** NB04 imports `from scipy.stats import percentileofscore` but uses `np.searchsorted` for scoring (which is faster and equivalent). The unused import should be removed.

**Proxy loop memory.** NB02 cell d1 accumulates `proxy_records` as a list before deduplication. If a protein appears in multiple similar reactions, it can be appended multiple times before the final `drop_duplicates`. This is not a bug — the deduplication is applied correctly — but the intermediate list can be larger than necessary. No impact on correctness.

**Pitfall compliance.** The project correctly uses `from berdl_notebook_utils.setup_spark_session import get_spark_session` (the on-cluster CLI import form, per `docs/pitfalls.md`). The `CAST` on the `similarity` column avoids the string-typed numeric column pitfall. No `SELECT DISTINCT` + aggregate patterns are used.

**Notebook organization.** All seven notebooks follow the standard setup → query → analysis → visualization pattern. Markdown headers, per-notebook input/output declarations, and sanity-check cells (NB03, section 6) are well placed.

---

## Findings Assessment

**Conclusions are supported by the data.** The five findings in the REPORT are directly traceable to notebook outputs:
- Finding 1 (coverage expansion from 12 to 32 reactions) is demonstrated in NB05 cell e1 with explicit counts.
- Finding 2 (0.013 median cosine separation) is computed in NB03b cell h1.
- Finding 3 (max-of-N order statistic trap) is quantified in NB05b cell b1 with direct excess values (-16.3 for direct, -2.5 for proxy).
- Finding 4 (2/32 reactions significant at p < 0.05) is the NB05b permutation result.
- Finding 5 (H0 not rejected) correctly synthesizes the above.

**Narrative inconsistency in NB05b.** The revised hypothesis evaluation cell (g1) prints "H1 WEAKLY SUPPORTED: After correction, only 1/32 reactions retain actionable evidence." Describing 3% actionability as "supported" is at odds with the README and REPORT, which correctly say H0 is not rejected. The print statement appears to be a threshold artifact from the verdict classification logic (`>= 0.3` triggers the H1-supported branch, and 3% < 30%, so the else branch fires — but the else-branch message was not updated to reflect H0). The REPORT has the right conclusion, but anyone reading notebook outputs rather than the REPORT could be confused.

**Limitations are well-acknowledged.** Single organism, Swiss-Prot only, mean-pooled embeddings, and 10 uncovered reactions are all clearly stated. The future-directions section is substantive and grounded (fine-tuned models, alignment-based scoring, per-residue attention weighting, multi-evidence integration).

**Orphan reactions.** 10 of 42 reactions have no candidates. The REPORT notes this but does not characterize *why* — e.g., are these novel KBase reactions with no SEED equivalents, unusual chemistries, or errors in the gap-fill input? A brief characterization of the orphan set would help readers understand whether they are a tractable future target or a structural gap in the data.

---

## Suggestions

1. **Fix the stale NB01 print output.** Re-run NB01 with the current code so the `DATA_OUT` path in the summary cell output matches the declared variable (`../user_data/gapfill_landscape.csv`). This is a one-cell re-run and ensures notebook outputs accurately reflect code state. **(Critical for reproducibility)**

2. **Acknowledge background pool selection bias in the REPORT.** Add a sentence to the Limitations section noting that the percentile rank background is drawn from Swiss-Prot candidates (already enzyme-biased in ESM-2 space), not from truly random protein sequences. State that this likely makes the background less conservative — meaning percentile rank scores may be over-estimated relative to a random-protein null. **(Important for methodological transparency)**

3. **Fix the NB05b verdict print string.** In cell g1 the else-branch prints "H1 WEAKLY SUPPORTED" when only 1/32 reactions (3%) are actionable. Change this to a statement consistent with "H0 not rejected" so the notebook output matches the README and REPORT conclusion. **(Moderate — avoids reader confusion)**

4. **Remove the unused `percentileofscore` import in NB04.** `from scipy.stats import percentileofscore` is imported but never called; `np.searchsorted` is used instead. Remove the unused import to avoid ambiguity about the scoring implementation. **(Minor)**

5. **Characterize the 10 orphan reactions.** Add a brief note in NB02 (or the REPORT) describing why each of the 10 uncovered reactions has no candidates — e.g., absent from `reaction_similarity` above threshold 0.7, novel KBase-only reactions, unusual stoichiometry. This helps prioritize future candidate-assembly work and tells readers whether the orphans are tractable or structurally problematic. **(Useful for future expansion)**

6. **Document the off-cluster Spark setup more explicitly.** The README says NB02 requires "a live Spark session via `berdl_notebook_utils` (on-cluster only, or off-cluster with `.venv-berdl`)" but does not explain what `.venv-berdl` is. Link to `.claude/skills/berdl-query/references/off-cluster-mechanics.md` or add a one-line pointer so a new collaborator can set up the environment without asking. **(Nice to have)**

7. **Note single-genome permutation test scope in the REPORT.** The cross-genome consistency justification (std = 0.0006) is solid for *E. coli*, but the REPORT's Future Directions section proposes expanding to diverse organisms — where within-species genome scores may diverge more. Add a caveat that per-genome permutation tests would be needed for cross-species studies. **(Nice to have, pre-empts a common reviewer question)**

---

## Review Metadata

- **Reviewer**: BERIL Automated Review (Claude, claude-sonnet-4-6)
- **Date**: 2026-05-20
- **Scope**: README.md, RESEARCH_PLAN.md, REPORT.md, references.md, requirements.txt, 7 notebooks (NB01–NB05b with full cell outputs), 12 figures, docs/pitfalls.md
- **Note**: This review was generated by an AI system. It should be treated as advisory input, not a definitive assessment.
