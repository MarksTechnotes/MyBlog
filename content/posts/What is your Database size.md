---
title: "Why “Database Size” Can Mean Different Things in ADW (Segments vs. Datafiles)"
date: 2026-03-28
draft: false
description: "Why “Database Size” Can Mean Different Things in ADW (Segments vs. Datafiles)"
tags: ["Oracle", "ADB", "Autonomous Database", "Datafiles", "Segments"]
categories: ["ADB", "Autonomous Database"]
---

## Why “Database Size” Can Mean Different Things in ADW (Segments vs. Datafiles)

When people ask “How big is the database?”, the answer depends on *which* size you’re measuring. In Autonomous Data Warehouse (ADW), two common SQL-based views can appear to disagree.



### Segments: object footprint (logical allocation)
A **segment** is the storage allocated to a database object—tables, indexes, LOB segments, partitions, etc. Querying `DBA_SEGMENTS` answers:

> “How much space has been allocated to my objects?”

```sql
SELECT tablespace_name,
       file_name,
       ROUND(bytes/1024/1024/1024, 2) AS gb
FROM   DBA_DATA_FILES
ORDER  BY gb DESC;
```

This is a great metric for schema/object analysis and identifying which objects are consuming space.

### Datafiles: allocated capacity (tablespace footprint)
A **datafile** is the physical storage allocated to a tablespace. Querying `DBA_DATA_FILES` answers:

> “How much storage capacity is allocated to this tablespace/database?”

 
```sql
SELECT tablespace_name,
       ROUND(SUM(bytes)/1024/1024/1024, 2) AS gb
FROM   dba_segments
GROUP  BY tablespace_name
ORDER  BY gb DESC;
```

This includes both used and unused space and is often the better “storage footprint” metric for capacity discussions.

### Example: the `SAMPLESCHEMA` tablespace
In one ADW database, `DBA_SEGMENTS` showed most space was in `SAMPLESCHEMA`:

- `SAMPLESCHEMA` segment allocation: **~162 GB**
- `SAMPLESCHEMA` datafile allocation: **~200 GB**

This is normal: segments represent *allocated-to-objects* space, while datafiles represent *allocated capacity*.

### Why `DBA_FREE_SPACE` can show 0 even when datafiles > segments
It’s common to see `DBA_FREE_SPACE` show no free space (or no rows) for a tablespace even when the datafile total exceeds segment bytes. That’s because `DBA_FREE_SPACE` reports **free extents**, not “all unused bytes,” and it does not account for:
- free space *inside* allocated segments (below a high-water mark),
- fragmentation or extent allocation patterns,
- and in managed services, it may not always behave as expected for every tablespace.

### Practical tip: avoid undercounting tablespaces in reports
If you build a “tablespace usage” report using an **INNER JOIN** between `DBA_DATA_FILES` and `DBA_FREE_SPACE`, you can accidentally drop tablespaces that have no `DBA_FREE_SPACE` rows. Prefer a `LEFT JOIN` (with `NVL`) or use `DBA_TABLESPACE_USAGE_METRICS` where available.

### Takeaway
When discussing “database size,” be explicit:
- Use **datafiles** for *allocated capacity / footprint*
- Use **segments** for *object-level allocation*
And don’t be surprised if `DBA_FREE_SPACE` doesn’t tell the full story about what’s truly “available.”