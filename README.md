# Promoted UPC Household Targeting

Given a list of promoted UPCs, this notebook finds households that are likely to buy similar products, and sorts them into Best / Better / Good groups (to mirror BAU Bullseye targeting).

Promoted lists sometimes span several unrelated categories (salad, protein, snacks, etc.), so a single "average" product doesn't work well. Instead, this notebook aims to first group the promoted UPCs into relevant clusters and then scores households separately for each cluster. Finally, for each relevant category of UPC in the promoted products list, we will get the "Best" HHs.

## How it works

1. **Embed the promoted UPCs.** Each UPC is turned into a vector using the kroBERT embedding model (based on product descriptions). Using krobert_cfic_1.0, but update to 3.0 when it is productionalized.
2. **Cluster them.** Hierarchical clustering groups similar promoted UPCs together. The distance threshold is picked automatically by `select_threshold`: it tries a range of thresholds, throws out ones that leave too many single-UPC clusters or too few/many clusters overall, then keeps the one with the best silhouette score.
3. **Build a centroid per cluster.** The centroid is the normalized average embedding of the UPCs in that cluster.
4. Set Cluster_on to "raw", short for raw embeddings: if promoted UPC list is mostly one category or very similar products, use this.  Set Cluster_on to "centered", short for mean centered embeddings: if promoted UPC list is a variety of subcommodities, OR if you want more granularity within a similar UPC type (example: coffee by origin, pods/not pods, ect)
5. **Find similar UPCs.** Every UPC in the catalog is compared to each centroid using cosine similarity. Promoted UPCs are removed, and only UPCs with a similarity of 0.60 or higher are kept.
6. **Bring in YNB scores.** For each household, we look up its YNB basket likelihood score for the similar UPCs.
7. **Score households within each cluster.** The final score is:

   | Piece | Weight |
   |---|---|
   | Cosine similarity of the UPC to the cluster | 40% |
   | Household's YNB score for that UPC | 40% |
   | Number of qualifying UPCs the household has (scaled 0 to 1 within the cluster) | 20% |

   A household's cosine/YNB piece comes from its best-scoring UPC in the cluster.
8. **Rank and bucket.** Households are ranked inside each cluster and split into deciles:
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
- The weights (40/40/20) and the 0.60 cosine cutoff are judgment calls. Change them at the top of the relevant cells if needed.

WIP: StreamLit app for interactive HH recommending.

## Requirements

Databricks with Spark, plus `effo_embeddings`, `openai`, `scikit-learn`, `scipy`, `numpy`, `pandas`, `matplotlib`.

Additional Embeddings Resources:
- https://effo-embeddings.pages.8451.com/references/example_notebooks/4_next_gen_rec_examples.html
- https://eaas.pages.8451.com/user/index.html
Requirements

Databricks with Spark, plus hdbscan, effo_embeddings, openai, scikit-learn, scipy, numpy, pandas, matplotlib.
