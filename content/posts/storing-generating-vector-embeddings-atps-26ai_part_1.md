---
title: "Generate and Store Vector Embeddings in ATP-S 26ai"
date: 2025-11-28
draft: false
description: "How to prepare data, generate embeddings using Hugging Face, and store vectors in Oracle ATP-S 26ai for hybrid semantic + relational search."
tags: ["Oracle", "ATP-S", "26ai", "Vector Search", "Hugging Face", "Embeddings"]
categories: ["Database", "Machine Learning"]
---

## Overview

Oracle Autonomous Database 26ai introduces native vector storage and vector search capabilities, making it possible to build hybrid semantic–relational search solutions directly inside the database.

In **Part 1 of this series**, the focus is on preparing the data and storing embeddings inside ATP-S 26ai.

This includes:

- Loading a Kaggle dataset  
- Creating relational and vector tables  
- Concatenating review text into a single embedding input  
- Generating embeddings externally using Hugging Face  
- Loading embeddings back into Oracle for vector search  

In [**Part 2**](https://markstechnotes.github.io/MyBlog/posts/hybrid_26ai_vector_search_part_2/) we will perform a hybrid semantic and relational search in 26ai on this vector data and finally in [**Part 3**](https://markstechnotes.github.io/MyBlog/posts/hybrid_26ai-_19c_vector_search_part_3/) we will extend the search functionality to hybrid deployments where the relational data exists in 19c instead of 26ai.

---

## 1. Dataset and Initial Setup

Download the Amazon product review dataset (CSV) from Kaggle at [Amazon Sales Dataset](https://www.kaggle.com/code/mehakiftikhar/amazon-sales-dataset-eda). This dataset contains product information, reviews, metadata, and descriptions.

Spin up a **26ai ATP-S** instance.  
We’ll use the “Load Data” page from Oracle "Database Actions" available with ATP-S to import the CSV.

---

## 2. Load the Amazon CSV Into Oracle

Using **Load Data**, upload the CSV file. Oracle automatically:

- Creates the table  
- Generates DDL  
- Loads the data  

The resulting table (called `AMAZON`) looks like this:

```
Name                Null? Type             
------------------- ----- ---------------- 
PRODUCT_ID                VARCHAR2(64)     
PRODUCT_NAME              VARCHAR2(4000)   
CATEGORY                  VARCHAR2(256)    
DISCOUNTED_PRICE          VARCHAR2(64)     
ACTUAL_PRICE              VARCHAR2(64)     
DISCOUNT_PERCENTAGE       VARCHAR2(64)     
RATING                    NUMBER           
RATING_COUNT              VARCHAR2(64)     
ABOUT_PRODUCT             VARCHAR2(4000)   
USER_ID                   VARCHAR2(4000)   
USER_NAME                 VARCHAR2(4000)   
REVIEW_ID                 VARCHAR2(256)    
REVIEW_TITLE              VARCHAR2(4000)   
REVIEW_CONTENT            VARCHAR2(32767)  
IMG_LINK                  VARCHAR2(256)    
PRODUCT_LINK              VARCHAR2(4000)   
```

---

## 3. Create PRODUCT_METADATA

We create a clean copy of the base table for easier relational joins:

```sql
CREATE TABLE PRODUCT_METADATA AS
SELECT *
FROM AMAZON
WHERE 1 = 0;

INSERT INTO PRODUCT_METADATA
SELECT *
FROM AMAZON;
```

Later, after fixing a bad row in the source CSV, this table was regenerated to maintain consistency.

---

## 4. Create PRODUCT_VECTORS (Vector Storage Table)

Oracle 26ai introduces the `VECTOR` datatype.  
We create a table to hold embeddings generated externally:

```sql
CREATE TABLE PRODUCT_VECTORS (
    VECTOR_ID        VARCHAR2(256) PRIMARY KEY,
    PRODUCT_ID       VARCHAR2(64),
    REVIEW_ID        VARCHAR2(256),
    VECTOR_SOURCE    VARCHAR2(50),
    EMBEDDING_VECTOR VECTOR(768),
    CREATED_AT       TIMESTAMP DEFAULT SYSTIMESTAMP
);
```

The `VECTOR(768)` type matches the embedding size used by the model.

---

## 5. Text to Embed

We want a single combined text field per review:

```
PRODUCT_NAME || ' ' || ABOUT_PRODUCT || ' ' ||
REVIEW_TITLE || ' ' || REVIEW_CONTENT
```

This gives a rich semantic representation for embedding generation.

---

## 6. Generate Embeddings Using Hugging Face

To avoid OpenAI rate limits, embeddings were generated on a laptop using the HuggingFace model:

- `sentence-transformers/all-mpnet-base-v2`  
- 768-dimensional embeddings  

A Python script `generate_hf_embeddings.py` produced a JSON file `amazon_with_embeddings.json` with:

- PRODUCT_ID  
- REVIEW_ID 
- ..
- ..
- EMBEDDING_TEXT 
- EMBEDDING_VECTOR (768 floats)

Example entry in output JSON file:

```json
{
  "PRODUCT_ID": "12345",
  "REVIEW_ID": "abcde",
  ..
  ..
  "EMBEDDING_TEXT": "Concatenated text from Product Name, About Product, Review Name, Review Title",
  "EMBEDDING_VECTOR": [0.0123, -0.984, ...]
}
```
The source code for `generate_hf_embeddings.py` and the resulting json file `amazon_with_embeddings.json` can be found at [**Vector Search Demo**](https://github.com/MarksTechnotes/markstechnotes-labs/tree/main/vector-search-demo)

---

## 7. Load Embeddings into Oracle

Upload the JSON embedding file into ATP-S using **Load Data**, creating a table like:

`AMAZON_WITH_EMBEDDINGS`

Populate `PRODUCT_VECTORS`:

```sql
INSERT INTO PRODUCT_VECTORS (
    VECTOR_ID,
    PRODUCT_ID,
    REVIEW_ID,
    VECTOR_SOURCE,
    EMBEDDING_VECTOR
)
SELECT
    SYS_GUID(),
    PRODUCT_ID,
    REVIEW_ID,
    'CONCATENATED_CONTENT',
    EMBEDDING_VECTOR
FROM AMAZON_WITH_EMBEDDINGS
WHERE REVIEW_ID IS NOT NULL;

COMMIT;
```

After running the full pipeline end-to-end, embeddings were generated for approximately **300 rows**.

---

## 8. What’s Next?

Now that ATP-S contains both:

- **Relational data** (`PRODUCT_METADATA`)  
- **Vector embeddings** (`PRODUCT_VECTORS`)  

We can support hybrid semantic search.

In [**Part 2**](https://markstechnotes.github.io/MyBlog/posts/hybrid_26ai_vector_search_part_2/), we will:

- Create query embeddings in Python  
- Connect to Oracle with `oracledb`  
- Execute `VECTOR_DISTANCE` searches  
- Join results with relational metadata  
- Return the top semantic matches for customer queries  

---

The complete source code can be found at [**Vector Search Demo**](https://github.com/MarksTechnotes/markstechnotes-labs/tree/main/vector-search-demo)   
