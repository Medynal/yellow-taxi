# NYC Yellow Taxi

Analysis of New York City yellow taxi trip records for July to September 2025, covering roughly 11.7 million trips. The project runs end to end in PySpark: data preparation, exploratory analysis, distributed clustering with MLlib, and a supervised demand classifier. Each stage records the reasoning behind the technical choices, with attention to how decisions affect performance.

## Dataset

The data comes from the New York City Taxi and Limousine Commission (TLC), which publishes monthly trip records. Three months (July, August and September 2025) are combined for this analysis.

Source: https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page

The three Parquet files are hosted in a github repository so the notebook downloads them once over HTTP, caches them locally, and skips the download on later runs. This keeps the notebook portable across machines without requiring Git.

## Why Spark

At 11 to 12 million rows across three months, the dataset sits above the point where single node tooling such as pandas becomes slow and memory bound. Spark provides distributed processing and a query optimiser, which makes the workload practical on a single workstation and demonstrates patterns that scale to a cluster.

The Spark session is configured rather than left at defaults. `spark.sql.shuffle.partitions` is reduced from the default of 200, which suits clusters with hundreds of cores but produces excessive scheduler overhead on a laptop. Adaptive query execution, skew join handling and Parquet filter pushdown are enabled, with each choice explained in the notebook.

## What the notebook does

**Data preparation.** Null and duplicate profiling are run as single pass aggregations to avoid repeated scans. Around 26% of rows carry systematic nulls in `passenger_count` and `ratecode_id`, traced to vendor software rather than random loss, so these are imputed with modal values rather than dropped. Negative fares are removed, trips outside the date window are filtered, and extreme distances are winsorised. The cleaning step removes about 10% of the raw data.

**Exploratory analysis.** Five questions are answered across four dimensions:

- Temporal: how pickup demand varies by hour of day and day of week.
- Spatial: which 20 pickup zones generate the most trips and revenue.
- Trip: how average fare scales with distance.
- Payment: tipping behaviour by pickup zone, applicable to only card payments since cash tips are not recorded by the meter.
- Temporal and fare: whether a rush hour premium exists in fare per mile, measured with the median to resist the heavy tail at short distances.

Each question is paired with a note on the relevant Spark behaviour, such as data skew during a `groupBy` on zone and how adaptive query execution mitigates it.

**Clustering.** MLlib KMeans is applied to two feature spaces. The first builds a behavioural fingerprint per pickup zone (trip volume, fares, distances, durations, cyclical hour encoding, weekend and airport shares). The second clusters individual trips into archetypes such as short urban, standard, outer borough and long airport. The number of clusters is chosen by a silhouette sweep, and the interpretation flags how singleton clusters can inflate the silhouette score.

**Supervised learning.** Each trip is assigned a demand class (low, medium, high) from quantile thresholds on hourly pickup volume. The small demand lookup table is broadcast to avoid shuffling the 11 million row dataset. A decision tree reaches 74% accuracy, with feature importance dominated by `pickup_hour`. The write up examines why: because the label is derived from an hourly aggregate, the hour feature leaks the target, so the accuracy reflects label construction rather than predictive insight.

## Key findings

- Weekday demand peaks in the evening commute window; weekend demand shifts later into Friday and Saturday nights. The 04:00 to 05:00 window is the daily minimum.
- A small set of Midtown Manhattan zones and the airports account for a disproportionate share of trips and revenue.
- Average fare is close to linear in distance.
- Card tip percentages cluster around 18 to 22% across most zones, with airports trending higher. Recorded cash tipping is near zero, which is a recording artefact rather than behaviour.

## Tech stack

PySpark (Spark SQL and DataFrame API), Spark MLlib, pandas, matplotlib, seaborn.

## Repository structure

```
nyc-yellow-taxi-spark/
├── README.md
├── yellowcab.ipynb        full analysis: preparation, EDA, clustering, classification
└── Dataset/               cached Parquet files (created on first run)
```

## References

The notebook cites published work on Spark performance and on TLC data quality, listed in its references section. Methods follow standard guidance for partition sizing, broadcast joins, and the cost difference between fitting and evaluating clustering models at scale.
