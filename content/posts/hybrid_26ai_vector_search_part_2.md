---
title: "Perform Hybrid Semantic and Relational Search on Oracle ATP-S 26ai Using Python"
date: 2025-11-28
draft: false
description: "Run hybrid semantic + relational search on Oracle ATP-S 26ai using Python and vector embeddings."
tags: ["Oracle", "ATP-S", "26ai", "Vector Search", "Python"]
categories: ["Database", "Machine Learning"]
---

## Overview

In [**Part 1**](https://markstechnotes.github.io/MyBlog/posts/storing-generating-vector-embeddings-atps-26ai_part_1/), we prepared an Oracle ATP-S 26ai environment by:

- Loading Amazon product data
- Creating relational and vector tables
- Generating embeddings using Hugging Face
- Storing them in `PRODUCT_VECTORS`

Now in Part 2, we’ll run a customer-style search query by combining:

- a vector similarity search, and
- relational filters from `PRODUCT_METADATA`

We’ll embed a customer query in Python, submit it to Oracle, and return the closest-matching products. 

## 1. Tables Recap

We will be querying two tables:

- `PRODUCT_METADATA` — contains relational product + review information.
- `PRODUCT_VECTORS` — stores the 768-dimensional embeddings:

| Column | Description |
|--------|-------------|
| VECTOR_ID | Unique ID (e.g., REVIEW_ID or PRODUCT_ID or SYS_GUID()) |
| PRODUCT_ID | Foreign key to metadata |
| REVIEW_ID | Optional, if embedding per review |
| VECTOR_SOURCE | Source of text (e.g., concatenated columns) |
| EMBEDDING_VECTOR | 768-dimensional vector |
| CREATED_AT | Timestamp |

**Goal:**
- Embed a customer query
- Compare it to `EMBEDDING_VECTOR`
- Return top matches joined with metadata

## 2. Generate a Customer Query Embedding in Python

Use the same Hugging Face model as in Part 1.

```python
from sentence_transformers import SentenceTransformer
model = SentenceTransformer("all-mpnet-base-v2")
query = "lightweight laptop for programming with long battery life"
query_embedding = model.encode(query).tolist()
```

Consistency between document embeddings and query embeddings is essential.

## 3. Connect to Oracle from Python

```python
import oracledb
connection = oracledb.connect(
    user="<ADB_USER>",
    password="<ADB_PASSWORD>",
    dsn="<ADB_DSN"
)
cursor = connection.cursor()
```

## 4. Perform Hybrid Vector + Relational Search 

The `query_embeddings` script converts the query embedding List object into a JSON-like string, which is sent to Oracle, converted using `TO_VECTOR()`, and compared against stored embeddings using `VECTOR_DISTANCE()`:


```python
vector_str = json.dumps(query_embedding)  # produces [0.01,0.2,...]
```

```sql
WITH query AS (
  SELECT TO_VECTOR('{vector_str}') AS q FROM dual
)
SELECT v.PRODUCT_ID,
       m.PRODUCT_NAME,
       v.REVIEW_ID,
       VECTOR_DISTANCE(v.EMBEDDING_VECTOR, q, COSINE) AS similarity
FROM PRODUCT_VECTORS v
JOIN PRODUCT_METADATA m
  ON v.PRODUCT_ID = m.PRODUCT_ID, query
ORDER BY similarity ASC
FETCH FIRST 10 ROWS ONLY
```

Joining with `PRODUCT_METADATA` allows retrieval of the top 10 most relevant products, combining semantic relevance with relational information.

The source code for `query_embeddings.py` can be found at [**Vector Search Demo**](https://github.com/MarksTechnotes/markstechnotes-labs/blob/main/vector-search-demo/query_embedding.py)   

You can optionally filter by category, price, or rating:

```sql
WHERE m.CATEGORY = 'Laptops' AND m.RATING >= 4
```

Hybrid search leverages both:
- **Vectors** → semantic relevance
- **SQL** → structured filtering

## 6. Example End-to-End Flow

1. Customer enters a search query
2. Python generates an embedding
3. Python sends embedding to Oracle
4. Oracle performs vector similarity search
5. Oracle joins semantic scores with relational data
6. Results are returned to Python

This completes the full hybrid relational and semantic search workflow in 26ai. In [**Part 3**](https://markstechnotes.github.io/MyBlog/posts/hybrid_26ai_19c_vector_search_part_3/) , we’ll extend this setup to **hybrid deployments** where relational data remains in a 19c database

---

The complete source code can be found at [**Vector Search Demo**](https://github.com/MarksTechnotes/markstechnotes-labs/tree/main/vector-search-demo)   
