# Unsupervised Learning — Clustering & Dimensionality Reduction

Brief primer — asked less often than supervised ML in senior interviews, but customer-segmentation and anomaly-detection framings do come up.

## Clustering

- **k-means**: partitions data into k spherical clusters by minimizing within-cluster variance. Choosing k: the **elbow method** (plot inertia vs k, look for the bend) and the **silhouette score** (higher = better-separated clusters). A retail team segmenting customers by recency/frequency/monetary (RFM) value typically lands on 4-6 clusters this way, then names each one ("dormant high-value," "new frequent buyer," etc.) for a marketing team to act on.
- **Hierarchical clustering**: builds a dendrogram of nested clusters — useful when you want to inspect structure at multiple granularities rather than commit to one k up front.
- **DBSCAN**: groups points by density and explicitly labels sparse points as noise, rather than forcing every point into a cluster. It finds arbitrarily shaped clusters where k-means (which assumes roughly spherical, similar-sized clusters) fails — e.g. detecting a ring-shaped fraud pattern in geographic coordinates that k-means would slice through incorrectly.

## Dimensionality reduction

- **PCA**: a linear, deterministic projection onto the directions of maximum variance. Used as a preprocessing step before another model, or to fight the curse of dimensionality before clustering. A **scree plot** (variance explained per component) tells you how many components to keep — commonly enough to explain 90-95% of variance.
- **t-SNE / UMAP**: non-linear, built for 2D/3D *visualization* of local structure, not general-purpose preprocessing — they distort global distances, so don't feed their output into a downstream model expecting Euclidean distance to mean something.

## Anomaly detection

- With little or no labeled anomaly data: **Isolation Forest** (anomalies get isolated in fewer random splits because they're "different"), **one-class SVM** (learns a boundary around normal data), or an **autoencoder** (train to reconstruct normal data; high reconstruction error on new data flags an anomaly). Credit-card fraud teams often run one of these as a first-pass filter before a supervised model even sees labeled fraud cases, precisely because labeled fraud is rare and delayed.

## A key pitfall

High-dimensional data breaks Euclidean-distance-based methods (k-means, k-NN) because in high dimensions, points become nearly equidistant from each other — the "curse of dimensionality." Reduce dimensionality (PCA) or pick a distance measure suited to the data before clustering wide feature sets.

## Quick self-check

Can you explain why DBSCAN would out-perform k-means on a dataset of GPS pings forming a ring-shaped delivery route, and why you'd reach for PCA rather than t-SNE if the reduced features are going to feed into a downstream regression model? If yes, you have enough here — move on to the supervised and deep-learning topics where interviews spend most of their time.
