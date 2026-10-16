## Retail Customer Segmentation & Recommendation System

## Overview
This repository hosts a project on online retail, using a dataset from a UK-based retailer available at the [UCI Machine Learning Repository](https://archive.ics.uci.edu/dataset/352/online+retail). The dataset contains all transactions between 2010 and 2011 (541,909 rows, 8 columns). It is used to build a customer segmentation model and a segment-based product recommender that support targeted marketing.

## Problem
Transactional data is turned into a customer-level dataset with RFM-style behaviour features. Customers are then grouped into distinct segments with the K-means algorithm, so that each group can be understood and marketed to differently. Finally, a recommendation system suggests best-selling products from a customer's own segment that the customer has not bought yet, while avoiding products that are often returned.

## Objectives
* **Data Cleaning & Transformation**: Handle missing values, duplicates, cancelled orders, non-product stock codes and invalid prices.
* **Feature Engineering**: Build 8 customer-level features covering recency, frequency, monetary value, product variety and return behaviour.
* **Data Preprocessing**: Reduce skew with a log transform, standardise the features, and apply PCA.
* **Customer Segmentation using K-Means**: Choose k using inertia, silhouette score and Davies-Bouldin index, then segment customers.
* **Cluster Analysis & Evaluation**: Profile each segment, assess cluster quality, and suggest a marketing strategy per segment.
* **Recommendation System**: Recommend 3 best-selling, low-return products per customer from their segment.

## Key Results

| Item | Result |
|:---|:---|
| Transactions after cleaning | 399,573 (from 541,909) |
| Customers segmented | 4,324 |
| Features | 8 (log-transformed and standardised) |
| PCA | 3 components, 86.4% of variance |
| Clusters (k) | 4 |
| Silhouette score | 0.339 |
| Davies-Bouldin index | 0.964 |
| Calinski-Harabasz index | 2,503.6 |
| Top segment (Champions) | 18.8% of customers, 67.3% of revenue |
| Recommendations | 3 products per customer, return-filtered |

## Dataset Description

| __Variable__ | __Description__ |
|     :---      |       :---      |
| __InvoiceNo__ | Code representing each unique transaction. If this code starts with the letter 'C', it indicates a cancellation. |
| __StockCode__ | Code uniquely assigned to each distinct product. |
| __Description__ | Description of each product. |
| __Quantity__ | The number of units of a product in a transaction. Negative values are returns or cancellations. |
| __InvoiceDate__ | The date and time of the transaction. |
| __UnitPrice__ | The unit price of the product in sterling. |
| __CustomerID__ | Identifier uniquely assigned to each customer. |
| __Country__ | The country of the customer. |

## Data Cleaning
The dataset went from 541,909 rows to **399,573 rows**.

| Step | Decision |
|:---|:---|
| Missing values | Rows without `CustomerID` (about 25%) or `Description` were dropped. Customer-level features and recommendations need the ID, and imputing it would invent customers. |
| Duplicates | 5,225 exact duplicate rows were removed as likely data-entry errors. |
| Cancelled orders | **Kept and flagged** with a `Transaction_Status` column. All negative quantities match the "C" invoice prefix, and return behaviour is a useful customer signal. After cleaning, cancelled invoices are 15.53% of all invoices (2.13% of line items). |
| Non-product stock codes | Codes such as `POST`, `D`, `C2`, `M`, `BANK CHARGES`, `PADS`, `DOT`, `CRUK` (0.48% of rows) were removed. |
| Service descriptions | "Next Day Carriage" and "High Resolution Image" rows were removed, and descriptions were standardised to uppercase. |
| Zero unit price | 33 rows were removed as likely errors or freebies. |

Outlier treatment was deliberately postponed until after feature engineering, because removing transaction-level outliers could discard genuine large buyers.

## Feature Engineering
Transactions were aggregated into one row per customer, giving **8 features**:

| Group | Feature | Definition |
|:---|:---|:---|
| Recency | `Days_Since_Last_Purchase` | Days since the last completed order |
| Frequency | `Total_Transactions` | Number of completed orders |
| Frequency | `Total_Products_Purchased` | Total units bought, net of returns |
| Monetary | `Total_Spend` | Sum of price × quantity, net of returns |
| Monetary | `Average_Transaction_Value` | Total spend divided by all invoices |
| Diversity | `Unique_Products_Purchased` | Number of distinct stock codes |
| Returns | `Cancellation_Frequency` | Number of cancelled invoices |
| Returns | `Cancellation_Rate` | Cancelled invoices divided by all invoices |

38 customers with zero or negative total spend (pure cancellations or freebies) were dropped, leaving **4,324 customers**.

## Preprocessing
* **Log transform:** the count and amount features were heavily right-skewed, so `log1p` was applied instead of deleting extreme customers, who are genuine high-value buyers. For example, skew fell from 21.55 to 0.21 for `Total_Spend` and from 22.93 to about 0 for `Total_Products_Purchased`. `Cancellation_Rate` is already a proportion and was left unchanged.
* **Scaling:** all 8 features were standardised with `StandardScaler`, since K-means is distance-based.
* **PCA:** 3 components were kept, enough to reach at least 85% of the variance (53.8%, 74.6% and 86.4% cumulative). The loadings were inspected to interpret the components.

## Customer Segmentation with K-Means

| k | Inertia | Silhouette | Davies-Bouldin |
|:--:|--:|--:|--:|
| 2 | 17,832.7 | 0.374 | 1.075 |
| 3 | 13,991.4 | 0.299 | 1.099 |
| **4** | **10,907.8** | **0.338** | **0.965** |
| 5 | 9,306.9 | 0.308 | 1.011 |
| 6 | 8,199.0 | 0.310 | 1.061 |

**k = 4** was chosen: it has the best Davies-Bouldin score and the best silhouette among sensible options (k = 3 to 6). k = 2 scores higher on silhouette, but it only separates active from inactive customers, which is too coarse for marketing. The final model uses `n_init=50` and `random_state=42`, and clusters are numbered by median spend (0 = highest).

**Pairwise check:** clusters 0 and 3 are the best separated (silhouette 0.64), and clusters 0 and 2 overlap the most (0.38). Merging clusters 1 and 3 would leave the overall silhouette essentially unchanged (0.339 to 0.342), but they were kept separate because their recency, order count and revenue contribution differ.

## Customer Segments
Values are medians unless stated otherwise. Shares cover identified customers only.

| | **0 · Champions** | **1 · Reliable Regulars** | **2 · Return-Prone Occasionals** | **3 · Low-Value One-Time Buyers** |
|:---|---:|---:|---:|---:|
| Customers | 811 (18.8%) | 1,368 (31.6%) | 713 (16.5%) | 1,432 (33.1%) |
| Share of revenue | 67.3% | 21.9% | 6.5% | 4.3% |
| Days since last purchase | 11 | 40 | 67 | 128 |
| Number of orders | 9 | 3 | 2 | 1 |
| Total spend (£) | 3,168 | 913 | 645 | 225 |
| Average order value (£) | 300 | 356 | 158 | 169 |
| Unique products bought | 115 | 50 | 31 | 14 |
| Cancellation rate (mean) | 0.20 | ~0.00 | 0.37 | ~0.00 |

**Suggested strategies**
* **Champions:** protect and reward them with account management, early access, volume discounts and loyalty perks. Their ~20% cancellation rate deserves investigation because it directly affects revenue.
* **Reliable Regulars:** grow them into Champions with reorder reminders, bundles and cross-selling.
* **Return-Prone Occasionals:** find out why orders are cancelled (product information, quality, delivery, wrong items) before offering heavy discounts.
* **Low-Value One-Time Buyers:** use low-cost win-back campaigns only.

## Recommendation System
1. **Data used:** completed purchases only.
2. **Best sellers per cluster:** products ranked by the number of *different customers* in the cluster who bought them, so one bulk buyer can't push an item to the top.
3. **Return filter:** products with more than 10% of sold units returned are excluded.
4. **Rule:** each customer gets the first 3 products from the top 50 best sellers of their cluster that they haven't bought.
5. **Output:** one row per customer with the cluster, three stock codes and their descriptions.

**Results:** 30 distinct products are recommended across all 4,324 customers, and only 2 customers (both Champions who had already bought almost everything in their pool) received fewer than 3 recommendations.

**Limitations**
* Recommendations are not strongly personalised: customers in the same cluster receive nearly the same list.
* The clusters' best-seller lists overlap (clusters 0 and 2 share 4 of their top 10, clusters 2 and 3 share 7), so the added value over a single shop-wide list should be tested.
* The recommender has not been evaluated on future purchases.
* New customers have no cluster and would need a fallback, such as overall best sellers.
* Silhouette 0.339 means moderately separated clusters: customer behaviour is closer to a continuum than isolated groups.
* Revenue and customer shares cover only identified customers, because about 25% of rows had no `CustomerID`.

**Possible improvements:** item-based collaborative filtering, evaluation on held-out purchases (precision@k, hit rate), and comparison against a global best-seller baseline.

## How to Run
1. Download the dataset from [UCI](https://archive.ics.uci.edu/dataset/352/online+retail) or Kaggle and save it as `data.csv`.
2. Install the requirements: `pandas`, `numpy`, `matplotlib`, `seaborn`, `scikit-learn`.
3. Open the notebook and update the data path. It reads from a Kaggle path by default (`/kaggle/input/datasets/carrie1/ecommerce-data/data.csv`), so change it to `data.csv` when running locally. Use `encoding='ISO-8859-1'`.
4. Run all cells in order.

## File Descriptions
-  **`Code.ipynb`**: Jupyter notebook with data cleaning, feature engineering, preprocessing, K-means segmentation, cluster profiling and the recommender.
-  **`data.csv`**: CSV file containing the online retail dataset.
-  **`README.md`**: This file, providing an overview of the project.
