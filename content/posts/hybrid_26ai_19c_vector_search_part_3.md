---
title: "Combine Oracle 19c Relational Data with Oracle 26ai Vector Search Using Materialized Views"
date: 2025-11-28
draft: false
description: "Guide to using Oracle ATP-S 26ai for vector search while querying relational data in Oracle 19c via materialized views and DB links."
tags: ["Oracle", "26ai", "19c", "Vector Search", "Materialized Views", "Hybrid Search", "Python", "Autonomous Database"]
categories: ["Database", "Cloud", "AI"]
---

## Overview

In [**Part 2**](https://markstechnotes.github.io/MyBlog/posts/hybrid_26ai_vector_search_part_2/), we prepared **PRODUCT_METADATA** and **PRODUCT_VECTORS** tables in Oracle 26ai, generated embeddings using Hugging Face, and demonstrated semantic search using Python.

In this part, we’ll extend this setup to **hybrid deployments** where relational data remains in a 19c database, but vector search happens in 26ai. We achieve this using **DB links** and **materialized views**.

### A Practical Guide for Hybrid Workloads

Modernizing applications doesn't always require a full database
migration. If you're running Oracle Database 19c today, you can adopt
Oracle 26ai's powerful vector capabilities without moving all your
relational data.\
This post shows how to extend your 19c environment with vector search in
26ai using **database links (DB Links)** and **materialized views
(MVs)** in a "hybrid architecture" that adds AI features with minimal
disruption.

------------------------------------------------------------------------

## Why Extend 19c Instead of Migrating?

Many enterprises have:

-   Large and stable 19c deployments
-   Existing PL/SQL logic, integrations, and application dependencies
-   Strict change‑control processes
-   Legacy applications that cannot be easily rewritten

A full migration to 26ai may take time but vector search can be
adopted immediately by standing up a complementary ATP‑S 26ai instance
and connecting it to your 19c database.

This hybrid approach allows teams to:

-   Keep relational data in 19c
-   Store embeddings and vectors in 26ai
-   Use 26ai for AI workloads
-   Query both systems together using DB links
-   Build eventually consistent caches using materialized views

------------------------------------------------------------------------

## Architecture Overview

    +------------------------+             +-----------------------------+
    | Oracle Database 19c    |   DB Link   | Oracle Autonomous DB 26ai   |
    | (Existing Relational)  +-----------> | (Vector Tables / AI Search) |
    | - Customers            |             | - Embeddings                |
    | - Orders               |             | - Vector Search Index       |
    +------------------------+             +-----------------------------+
              ^                                           |
              |------------- Materialized Views ----------|

------------------------------------------------------------------------


------------------------------------------------------------------------

## Step 1. Hybrid Search Setup

We want to query relational data in **19c** while performing vector search in **26ai**:

- **PRODUCT_METADATA** → resides in 19c ATP-S instance  
- **PRODUCT_VECTORS** → resides in 26ai ATP-S instance

This allows customers to leverage vector search in 26ai **without migrating existing relational data** from 19c.

---

## Step 2. Prepare 19c Relational Data

1. Create a **19c ATP-S instance** (`reviews19c`).  
2. Use **Load Data** from Oracle Database Actions to import `amazon.csv` into the `PRODUCT_METADATA` table.

---

## Step 3: Create a Database Link in 26ai Pointing to 19c

In 26ai:

```sql
BEGIN
  DBMS_CLOUD.CREATE_CREDENTIAL(
        credential_name => 'DB_LINK_CRED',
        username => 'your_19c_user',
        password => 'your_19c_password'
  );
END;
/

BEGIN
    DBMS_CLOUD_ADMIN.CREATE_DATABASE_LINK(
        db_link_name    => 'REVIEWS19C_LINK', 
        hostname        => 'your-19c-host',
        port            => '1521',
        service_name    => 'your_19c_service',
        credential_name => 'DB_LINK_CRED',
        directory_name  => NULL
    );
END;
/
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

```sql
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
```
The source code for `query_embeddings.py` can be found at [**Vector Search Demo**](https://github.com/MarksTechnotes/markstechnotes-labs/blob/main/vector-search-demo/query_embedding.py) 

This executes a **hybrid semantic + relational search**:

- `PRODUCT_VECTORS` → provides vector similarity  
- `PRODUCT_METADATA_MV` → provides relational filters from 19c data  

------------------------------------------------------------------------


## Benefits of the Hybrid Approach

* ✅ No large migration project: relational data stays in 19c     
* ✅ Incremental adoption of AI: only vector workloads move to 26ai   
* ✅ Safe and reversible: drop the DB link or MV if needed    
* ✅ Works with any 19c application: PL/SQL, APEX, ORDS, Java, OCI apps   
* ✅ Modernize at your own pace: migrate tables later if desired  

## Summary

You can add powerful vector search to your existing Oracle 19c
applications **without** a disruptive migration by:

1.  Creating a 26ai ATP‑S instance
2.  Building vector tables there
3.  Connecting 19c to 26ai using DB links
4.  Creating materialized views for local caching
5.  Updating only the parts of your application that need AI features

This hybrid pattern gives you the best of both worlds: stability in 19c
and modern AI capabilities in 26ai.

------------------------------------------------------------------------
The complete source code can be found at [**Vector Search Demo**](https://github.com/MarksTechnotes/markstechnotes-labs/tree/main/vector-search-demo) 
