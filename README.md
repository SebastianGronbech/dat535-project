# Yelp Dataset ETL & Sentiment Analysis Pipeline

This project implements a **Medallion Architecture** (Bronze $\rightarrow$ Silver $\rightarrow$ Gold) using **PySpark** to process the Yelp Academic Dataset. It handles data ingestion, schema validation, transformation, and applies a supervised Machine Learning model to perform sentiment analysis on reviews.

## 📁 Project Structure

```text
.
├── silver_transform_business.ipynb      # ETL: Raw Business JSON -> Silver Parquet
├── silver_transform_reviews.ipynb       # ETL: Raw Review JSON -> Silver Parquet
└── gold_supervised_sentiment_model.ipynb # ML: Train Sentiment Model & Create Gold Aggregates
```

## 🏗 Architecture Overview

The pipeline processes data through three layers:

1.  **Bronze (Raw):** Raw JSON files ingested from the Yelp dataset (`business.json`, `review.json`).
2.  **Silver (Cleaned):**
    *   Parsed, validated, and deduplicated data.
    *   Data types cast to correct formats (Integers, Dates).
    *   Text cleaning applied to reviews (HTML removal, regex cleaning).
    *   Stored in **Parquet** format, partitioned by `ingest_date`.
3.  **Gold (ML & Aggregated):**
    *   **Sentiment Analysis:** A Logistic Regression model trained on user stars to predict sentiment (Positive/Negative) based on review text.
    *   **Business Summary:** Aggregated metrics per business (Avg Star Rating vs. Avg ML Sentiment Score).

## 🛠 Prerequisites

*   **Apache Spark** (Cluster or Local mode)
*   **Python 3.11+**
*   **Libraries:** `pyspark`, `findspark`
*   **Data:** Yelp Academic Dataset (JSON format) placed in the Bronze directory.

> **Note:** The notebooks are currently configured to connect to a Spark Standalone Master at `spark://192.168.5.121:7077`. You may need to adjust the `.master()` configuration in the `SparkSession` builder for your environment.

---

## 🚀 Pipeline Details

### 1. Business Data Transformation (Bronze $\rightarrow$ Silver)
*   **Notebook:** `silver_transform_business.ipynb`
*   **Input:** `/data/bronze/yelp/raw/.../yelp_academic_dataset_business.json`
*   **Key Operations:**
    *   **Resilient Parsing:** Uses Spark RDDs to safely parse JSON and segregate malformed records (`_status` = `parse_error`) from valid ones.
    *   **Validation:** Checks for existence of `business_id`, `name`, and `categories`.
    *   **Deduplication:** Removes duplicates based on `business_id`.
    *   **Transformation:** Splits category strings into arrays and handles nulls.
*   **Output:** Parquet files at `/data/silver/yelp/business/`.

### 2. Review Data Transformation (Bronze $\rightarrow$ Silver)
*   **Notebook:** `silver_transform_reviews.ipynb`
*   **Input:** `/data/bronze/yelp/raw/.../yelp_academic_dataset_review.json`
*   **Key Operations:**
    *   **Schema Validation:** Ensures star ratings are 1-5 and dates are valid ISO formats.
    *   **Text Cleaning:** Removes HTML tags and special characters to prepare text for NLP.
    *   **Deduplication:** Removes duplicates based on `review_id`.
*   **Output:** Parquet files at `/data/silver/yelp/review/`.

### 3. Machine Learning & Gold Layer
*   **Notebook:** `gold_supervised_sentiment_model.ipynb`
*   **Input:** Silver Business and Review Parquet data.
*   **Workflow:**
    1.  **Label Engineering:** 
        *   Stars $\le$ 2 $\rightarrow$ Negative (Label 0)
        *   Stars $\ge$ 4 $\rightarrow$ Positive (Label 1)
        *   Stars = 3 $\rightarrow$ Dropped from training.
    2.  **Feature Engineering Pipeline:**
        *   `Tokenizer` $\rightarrow$ `StopWordsRemover` $\rightarrow$ `HashingTF` (Term Frequency) $\rightarrow$ `IDF` (Inverse Document Frequency).
    3.  **Model Training:** Logistic Regression (80/20 Train/Test split).
    4.  **Inference:** Applies the model to *all* reviews to generate a `sentiment_ml` prediction.
    5.  **Enrichment:** Joins predictions with Business metadata.
*   **Output:**
    *   **Reviews Gold:** `/data/gold/yelp/review_sentiment_ml/` (Reviews with ML sentiment).
    *   **Business Summary Gold:** `/data/gold/yelp/business_sentiment_summary/` (Aggregated stats).

---

## 📊 Data Schema (Silver Layer)

### Business Table
| Column | Type | Description |
| :--- | :--- | :--- |
| `business_id` | String | Unique Identifier |
| `name` | String | Business Name |
| `categories` | Array[String] | List of categories |
| `ingest_date` | String | Partition column (YYYY-MM-DD) |

### Review Table
| Column | Type | Description |
| :--- | :--- | :--- |
| `review_id` | String | Unique Identifier |
| `text_clean` | String | Pre-processed review text |
| `stars` | Integer | User rating (1-5) |
| `date` | String | Review date |
| `useful/funny/cool`| Integer | Vote counts |

---

## 📈 ML Model Performance
The pipeline includes an evaluation step using `MulticlassClassificationEvaluator`.
*   **Algorithm:** Logistic Regression
*   **Metric:** Accuracy
*   **Features:** TF-IDF Vectors (Top 10,000 features)