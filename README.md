# Promoted UPC Household Targeting

Given a list of promoted UPCs, this notebook finds households that are likely to buy similar products, and sorts them into Best / Better / Good groups.

Promoted lists usually span several unrelated categories (salad, protein, snacks, etc.), so a single "average" product doesn't work well. Instead, the notebook first groups the promoted UPCs into clusters and then scores households separately for each cluster.

## How it works

1. **Embed the promoted UPCs.** Each UPC is turned into a vector using the kroBERT embedding model (based on product descriptions).
2. **Cluster them.** Hierarchical clustering groups similar promoted UPCs together. The distance threshold is picked automatically by `select_threshold`: it tries a range of thresholds, throws out ones that leave too many single-UPC clusters or too few/many clusters overall, then keeps the one with the best silhouette score.
3. **Build a centroid per cluster.** The centroid is the normalized average embedding of the UPCs in that cluster.
4. **Find similar UPCs.** Every UPC in the catalog is compared to each centroid using cosine similarity. Promoted UPCs are removed, and only UPCs with a similarity of 0.60 or higher are kept.
5. **Bring in YNB scores.** For each household, we look up its YNB propensity score for the similar UPCs.
6. **Score households within each cluster.** The final score is:

   | Piece | Weight |
   |---|---|
   | Cosine similarity of the UPC to the cluster | 40% |
   | Household's YNB score for that UPC | 40% |
   | Number of qualifying UPCs the household has (scaled 0 to 1 within the cluster) | 20% |

   A household's cosine/YNB piece comes from its best-scoring UPC in the cluster.
7. **Rank and bucket.** Households are ranked inside each cluster and split into deciles:
   - **Best**: top 10%
   - **Better**: next 40%
   - **Good**: bottom 50%

## Inputs

- A list of promoted UPCs (paste it into `promoted_upc_list`; you can download it from Prism for the campaign).
- The kroBERT embeddings parquet for all UPCs (the latest run date is picked up automatically).
- The YNB table `personalization_prd.relevancy_sciences.pzn_ynb_prd_final_rank` (the latest fiscal week is picked up automatically).
- An API key for the krAIG proxy, stored in the Databricks secret scope.

## Output

`hh_deciles` has one row per household per cluster, with:

| Column | Meaning |
|---|---|
| `centroid` | Cluster id |
| `EHHN` | Household id |
| `ynb_cossim_blended_score` | Cosine + YNB score of the household's best UPC |
| `number_hh_upcs` | Scaled count of qualifying UPCs |
| `ynb_cossim_upc_final_blended_score` | Final blended score |
| `hh_rank` | Rank within the cluster (1 is best) |
| `HH_Decile`, `HH_Bucket` | Decile (10 is best) and Best/Better/Good label |

`best_hhs` is the same table filtered to the "Best" households.

## Things to know

- Scores are only comparable **within** a cluster, not across clusters. A household can show up in more than one cluster.
- The thresholds in `select_threshold` (max 10% singletons, 8 to 25 clusters) are fixed numbers. If the promoted list gets much bigger or smaller, adjust them.
- The weights (40/40/20) and the 0.60 cosine cutoff are judgment calls. Change them at the top of the relevant cells if needed.
- Don't call `.display()` or `.toPandas()` on the full household tables. They are very large and will hit the driver result limit. Filter to one cluster or use `.limit()`.

## Requirements

Databricks with Spark, plus `effo_embeddings`, `openai`, `scikit-learn`, `scipy`, `numpy`, `pandas`, `matplotlib`.
Requirements

Databricks with Spark, plus hdbscan, effo_embeddings, openai, scikit-learn, scipy, numpy, pandas, matplotlib.
