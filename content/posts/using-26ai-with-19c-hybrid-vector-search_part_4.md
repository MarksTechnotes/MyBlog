---
title: "How to Combine Oracle 19c Relational Data with Oracle 26ai Vector Search Using Materialized Views - Part 2"
date: 2025-11-28
draft: false
description: "Guide to using Oracle ATP-S 26ai for vector search while querying relational data in Oracle 19c via materialized views and DB links."
tags: ["Oracle", "26ai", "19c", "Vector Search", "Materialized Views", "Hybrid Search", "Python"]
categories: ["Database", "Cloud", "AI"]
---

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

## Step 1: Create a Database Link in 26ai Pointing to 19c

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

------------------------------------------------------------------------

## Step 2: Create the Vector Table in 26ai

``` sql
CREATE TABLE product_vectors (
    product_id NUMBER PRIMARY KEY,
    description_text CLOB,
    embedding VECTOR(1536)
);
```

Insert vector embeddings using external embedding models or OCI GenAI.

------------------------------------------------------------------------

## Step 3: Query 19c Data Using Materialized View in 26ai

Create a materialized view in 26ai to cache relational data from 19c:

```sql
CREATE MATERIALIZED VIEW product_metadata_mv
BUILD IMMEDIATE
REFRESH COMPLETE
ON DEMAND
AS
SELECT *
FROM product_metadata@REVIEWS19C_LINK;
```

Now queries in 26ai can combine vector search with relational data efficiently.

------------------------------------------------------------------------

## Step 4: Hybrid Semantic + Relational Search in 26ai

```sql
WITH query AS (
  SELECT TO_VECTOR(:query_embedding) AS q FROM dual
)
SELECT v.product_id,
       m.product_name,
       v.review_id,
       VECTOR_DISTANCE(v.embedding, q, COSINE) AS similarity
FROM product_vectors v
JOIN product_metadata_mv m
  ON v.product_id = m.product_id, query
ORDER BY similarity ASC
FETCH FIRST 10 ROWS ONLY;
```
* Vector search uses VECTOR_DISTANCE  
* Relational filters come from product_metadata_mv

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

