---
title: "How to Combine Oracle 19c Relational Data with Oracle 26ai Vector Search Using Materialized Views"
date: 2025-11-28
draft: false
description: "Guide to using Oracle ATP-S 26ai for vector search while querying relational data in Oracle 19c via materialized views and DB links."
tags: ["Oracle", "26ai", "19c", "Vector Search", "Materialized Views", "Hybrid Search", "Python"]
categories: ["Database", "Cloud", "AI"]
---

In the previous post, we prepared **PRODUCT_METADATA** and **PRODUCT_VECTORS** tables in Oracle 26ai, generated embeddings using Hugging Face, and demonstrated semantic search using Python.

In this post, we’ll extend this setup to **hybrid deployments** where relational data remains in a 19c database, but vector search happens in 26ai. We achieve this using **DB links** and **materialized views**.

---

## 1. Hybrid Search Setup

We want to query relational data in **19c** while performing vector search in **26ai**:

- **PRODUCT_METADATA** → resides in 19c ATP-S instance  
- **PRODUCT_VECTORS** → resides in 26ai ATP-S instance

This allows customers to leverage vector search in 26ai **without migrating existing relational data** from 19c.

---

## 2. Prepare 19c Relational Data

1. Create a **19c ATP-S instance** (`reviews19c`).  
2. Use **Load Data** from Oracle Database Actions to import `amazon.csv` into the `PRODUCT_METADATA` table.


---

## 3. Create a DB Link in 26ai to 19c

### Step 1: Create Credential

```sql
BEGIN
  DBMS_CLOUD.CREATE_CREDENTIAL(
        credential_name => 'DB_LINK_CRED',
        username => '<ADB_USER>',
        password => '<ADB_PASSWORD>'
  );
END;
/
```

### Step 2: Create Database Link

```sql
BEGIN
    DBMS_CLOUD_ADMIN.CREATE_DATABASE_LINK(
        db_link_name => 'REVIEWS19C_LINK', 
        hostname => '<ADB_HOST>', 
        port => '1521',
        service_name => '<ADB_SERVICE>',
        credential_name => 'DB_LINK_CRED',
        directory_name => NULL
    );
END;
/
```

### Step 3: Test the Link

```sql
SELECT * FROM PRODUCT_METADATA@REVIEWS19C_LINK;
```

---

## 4. Create a Materialized View in 26ai

Materialized views allow **26ai** to access relational data from 19c efficiently:

```sql
CREATE MATERIALIZED VIEW PRODUCT_METADATA_MV
BUILD IMMEDIATE
REFRESH COMPLETE
ON DEMAND
AS
SELECT *
FROM PRODUCT_METADATA@REVIEWS19C_LINK;
```

---

## 5. Query Using Python

Once the materialized view is ready, update your `query_embeddings.py` code to reference `PRODUCT_METADATA_MV` instead of `PRODUCT_METADATA`.

```python
sql = f"""
WITH query AS (
  SELECT TO_VECTOR('{vector_str}') AS q FROM dual
)
SELECT v.PRODUCT_ID,
       m.PRODUCT_NAME,
       v.REVIEW_ID,
       VECTOR_DISTANCE(v.EMBEDDING_VECTOR, q, COSINE) AS similarity
FROM PRODUCT_VECTORS v
JOIN PRODUCT_METADATA_MV m
  ON v.PRODUCT_ID = m.PRODUCT_ID, query
ORDER BY similarity ASC
FETCH FIRST 10 ROWS ONLY
"""
```
The source code for `query_embeddings.py` can be found at https://github.com/MarksTechnotes/markstechnotes-labs/blob/main/vector-search-demo/query_embedding.py

This executes a **hybrid semantic + relational search**:

- `PRODUCT_VECTORS` → provides vector similarity  
- `PRODUCT_METADATA_MV` → provides relational filters from 19c data  

---

## 6. Summary

By combining **DB links** and **materialized views**, you can:

- Keep your **core relational data in 19c**  
- Use **Oracle 26ai vector search** for semantic queries  
- Avoid full migrations and risky cutovers  
- Enable **hybrid search** combining vector similarity with relational filters  

This pattern is ideal for customers looking to adopt vector search capabilities **incrementally** without disrupting existing 19c deployments.

---

The complete source code can be found at https://github.com/MarksTechnotes/markstechnotes-labs/tree/main/vector-search-demo