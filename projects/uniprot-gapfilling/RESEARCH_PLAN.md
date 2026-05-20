# Research Plan: UniProt-Guided Gap-Filling via Enzyme and Reaction Embeddings

## Research Question
Can ESM-2 protein embeddings, combined with Rosetta's UniProt-to-ModelSEED mappings, provide biological evidence that an E. coli genome encodes enzymes capable of catalyzing gap-filled reactions?

## Hypothesis
- **H0**: Embedding similarity between known reaction catalysts (Swiss-Prot) and genome proteins does not distinguish biologically plausible gap-fills from implausible ones
- **H1**: Gap-filled reactions whose known catalysts have high embedding similarity to genome proteins are more likely to be biologically justified

## Scope
This is a prototype using 42 gap-fill reactions across 48 E. coli genomes. The goal is a working notebook that demonstrates the approach end-to-end. If the prototype shows promise, it can be expanded to more organisms, broader UniProt coverage (TrEMBL, multi-channel evidence), and integration with the KBase reconstruction pipeline.

## Literature Context

Standard gap-filling algorithms add reactions to metabolic models based on stoichiometric feasibility, with little biological evidence that the host organism encodes the necessary enzymes (Karp et al., 2018; Orth & Palsson, 2012). Several tools address this by incorporating genomic, phylogenetic, or topological evidence (Zimmermann et al., 2021 — gapseq; Prigent et al., 2017 — Meneco; Vayena et al., 2022), but none leverage pretrained protein language model embeddings as a similarity signal.

ESM-2 (Lin et al., 2023) produces 1280-dimensional embeddings that capture structural and functional properties of proteins at evolutionary scale. The question is whether cosine similarity in this embedding space can discriminate between plausible and implausible enzyme candidates for gap-filled reactions — a use case not yet tested in the literature.

This project combines ESM-2 embeddings with Rosetta's curated UniProt-to-ModelSEED mappings (built on ModelSEED biochemistry; Henry et al., 2021) to test whether embedding similarity provides gap-filling evidence beyond what sequence annotation alone offers.

See [references.md](references.md) for the full bibliography.

## Approach

### Data Flow
```
Gap-fill reactions (42 rxns × 48 genomes)
        │
        ▼
Rosetta mappings (reaction → EC → Swiss-Prot proteins)
        │
        ▼
Swiss-Prot protein sequences (from refdata_uniprot)
        │
        ▼
ESM-2 embeddings for candidates          ESM-2 embeddings for genome
(via llm_homology_api, ~1,140 proteins)  (pre-computed in ecoli-files/)
        │                                 │
        ▼                                 ▼
FAISS cosine similarity search
        │
        ▼
Scored gap-fill proposals: Rosetta tier × embedding similarity
```

### Key Design Decisions
1. **Swiss-Prot only for prototype**: 1,140 proteins vs 222,522 total candidates. Swiss-Prot proteins are reviewed/curated and showed higher validation accuracy in Rosetta (F1 0.89 vs TrEMBL 0.84). This keeps the embedding retrieval to ~3 API batches.
2. **Rosetta evidence tiers weight the score**: A Tier 1 (UniProt-native EC) candidate with moderate embedding similarity should rank above a proxy hit with high similarity, because the functional annotation is stronger.
3. **Reaction similarity as last resort**: Only query `reaction_similarity` for the ~14 gap-fill reactions with no Rosetta evidence. Rosetta already covers 26/42 with evidence.
4. **Genome embeddings are pre-computed**: 48 E. coli genomes already have FAISS indexes in `ecoli-files/`.

## Data Sources

### From Rosetta Project (`projects/rosetta/data/`)
| File | Purpose | Size |
|---|---|---|
| `evidence_integration_summary.parquet` | Per-reaction evidence channels and confidence tiers | 34,343 reactions |
| `ec_to_reaction.parquet` | EC number → ModelSEED reaction bridge | 22,823 pairs |
| `uniprot_native_protein_ec.parquet` | UniProt protein → EC mappings with channel counts | 27.6M pairs |
| `swissprot_proteins.parquet` | Swiss-Prot protein ID list (reviewed subset) | — |

### From BERDL (Spark SQL)
| Table | Purpose | Filter Strategy |
|---|---|---|
| `kbase_msd_biochemistry.reaction_similarity` | Similar reactions for proxy fallback | `similarity > 0.7`, only for no-evidence reactions |
| `refdata_uniprot.protein` | Protein sequences for embedding retrieval | Filter to Swiss-Prot candidates |

### User-Provided (`user_data/`)
| File | Purpose |
|---|---|
| `enzyme-similarity.ipynb` | Original prototype notebook |
| `fetch_genome_embeddings.ipynb` | Embedding retrieval pipeline |
| `gapfill-rxn-genome-media.txt` | 42 gap-fill reactions × 48 genomes with media conditions |
| `uniprot-msd-links.tsv` | Original flat UniProt→reaction links (262K, replaced by Rosetta) |
| `uniprot.embeddings.json` | Original 859 UniProt embeddings (to be expanded) |
| `ecoli-files/` | 48 genomes: FASTA + FAISS indexes + embedding results |

## Analysis Plan

### NB01 — Gap-Fill Landscape
- **Goal**: Characterize the 42 gap-fill reactions against Rosetta evidence
- **Method**: Cross-reference gap-fill reactions with `evidence_integration_summary.parquet`. For each reaction: confidence tier, evidence channels, number of Swiss-Prot proteins linked via EC
- **Expected output**: Table partitioning reactions into well-annotated (26), unannotated (14), and missing (2). Figure showing evidence coverage

### NB02 — Swiss-Prot Candidate Assembly
- **Goal**: Build the candidate enzyme list for each gap-fill reaction
- **Method**:
  - Join `ec_to_reaction` + `uniprot_native_protein_ec` + `swissprot_proteins` to get Swiss-Prot proteins per reaction
  - For 14 no-evidence reactions: query `reaction_similarity` via Spark SQL, find proxy reactions with Swiss-Prot candidates
  - Fetch protein sequences from `refdata_uniprot`
- **Expected output**: Candidate table (reaction, uniprot_id, sequence, evidence_tier) — estimated ~1,140 unique proteins

### NB03 — Embedding Retrieval
- **Goal**: Get ESM-2 embeddings for all Swiss-Prot candidates
- **Method**: Submit sequences to `llm_homology_api` in batches of 500 (same pattern as `fetch_genome_embeddings.ipynb`). Verify genome FAISS indexes load correctly
- **Expected output**: `swissprot_candidate_embeddings.json`, FAISS index verification for 48 genomes

### NB04 — Similarity Scoring
- **Goal**: Score each gap-fill reaction per genome
- **Method**: For each genome × reaction:
  - Load genome FAISS index
  - Search with each Swiss-Prot candidate embedding
  - Record top-k hits with cosine similarity
  - Composite score = f(Rosetta evidence tier, embedding cosine similarity)
- **Expected output**: `gapfill_scores.parquet` — (reaction, genome, best_gene, cosine_sim, evidence_tier, composite_score)

### NB05 — Evaluation
- **Goal**: Assess whether embedding similarity provides useful gap-filling evidence
- **Method**:
  - Coverage: how many of the 42 reactions get confident candidates?
  - Score distributions by evidence tier
  - Comparison: Rosetta-enhanced vs original flat-lookup
  - Which reactions remain orphans?
- **Expected output**: Summary figures, assessment of whether expansion is warranted

## Known Data Limitations

**Gene ID format in genome results**: The `llm_homology_api` `.results.json` files use ordinal FASTA sequence numbers (1, 2, 3, …) as `query_id`, not the gene locus identifiers from FASTA headers (e.g., `562.55864.con.0010`). The `gene_id` column in scoring outputs therefore contains these ordinal indices. To resolve back to locus IDs, parse the corresponding `.faa` file and map by sequence position. This does not affect the similarity scoring (which operates on embeddings, not IDs) but limits interpretability of individual gene hits.

## Expected Outcomes
- **If H1 supported**: Prototype demonstrates that embedding similarity adds discriminative evidence for gap-filling → expand to full UniProt, more organisms, pipeline integration
- **If H0 not rejected**: Embeddings don't help distinguish plausible from implausible gap-fills → revisit scoring function or conclude that sequence-level evidence isn't sufficient for this task
- **Potential confounders**: ESM-2 may cluster by fold rather than function; Swiss-Prot coverage may be biased toward well-studied enzymes

## Future Expansion (if prototype succeeds)
- Add multi-channel TrEMBL proteins (~594 additional)
- Expand beyond E. coli to diverse organisms
- Integrate with KBase model reconstruction pipeline
- Use reaction embeddings (not just enzyme embeddings) for additional signal

## Revision History
- **v1** (2026-05-15): Initial plan — awaiting notebook upload
- **v2** (2026-05-15): Revised with Swiss-Prot-first prototype scope after candidate count analysis (222K total → 1,140 Swiss-Prot)
- **v3** (2026-05-20): Added literature context, documented gene ID format limitation, post-review updates

## Authors
- Sam Seaver, KBase / Argonne National Laboratory
