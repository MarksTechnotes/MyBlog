---
title: "Tips to Speed Up Data Pump Imports into Autonomous Database (ADB)"
date: 2025-11-14
draft: false
description: "Practical guidelines to improve performance and reduce runtime when importing large databases into Oracle Autonomous Database using Data Pump."
tags: ["Oracle", "Autonomous Database", "Data Pump", "Migration", "Performance"]
categories: ["Database", "Cloud"]
---

Many DBAs prefer using **Oracle Data Pump** when migrating data into **Autonomous Database (ADB)**, largely because the tool is familiar and reliable. However, Data Pump imports can become slow and resource-intensive, especially when migrating databases larger than a few 100 GB. In one customer case I was involved in, a **650 GB import** into ADB took **18 hours** to complete - clearly a pain point.

A few simple adjustments can dramatically reduce import times and make the overall migration process smoother.

---
## 1. Temporarily Increase OCPUs for the duration of the import

The ATP-S instance being used by the customer was configured with just 1 OCPU. While enough for the daily workload requirements it was slowing down the import. For large imports, consider temporarily scaling up OCPUs on your ADB instance:

1. Manually increase OCPUs before starting the import even if Auto-Scaling is enabled  
2. Run the Data Pump job  
3. Scale back down when finished  

Since ADB scales compute independently and charges by the hour, this approach is cost-effective and significantly boosts performance.

---
## 2. Use the `parallel` parameter wisely

### ATP-S Data Pump Import Parallel Cheat Sheet

| OCPUs (Base)       | Recommended PARALLEL       | Notes / Guidelines |
|:-------------------|:---------------------------|:------------------|
| 1                 | 1                         | Single worker; no parallelism. |
| 2                 | 2                         | Safe for small imports. |
| 4                 | 4                         | Fully utilize CPU; start point for moderate imports. |
| 8                 | 8                         | Standard for large imports; matches CPU count. |
| 16                | 16                        | Good for very large datasets; monitor PGA and I/O. |
| 32                | 32                        | Only if the import is truly CPU-bound; ensure TEMP/I/O can handle it. |
| Optional Slight Increase | 1.25×–1.5× OCPUs   | Only if CPU utilization is low and I/O is light; monitor closely. |
| Avoid              | 2× or more OCPUs         | Likely to oversubscribe CPU and PGA, causing slower throughput or failures. |

**Monitor during import:**
   
```sql
   SELECT session_id, program, service_name, degree
   FROM v$session
   WHERE program LIKE '%impdp%';
```

Setting `parallel` too high often leads to **resource contention**, which can slow the import down instead of speeding it up. Moderate parallelism strikes the right balance between performance and stability.

---

## 3. Use the Autonomous Database `HIGH` Service

For large import operations, always connect using the `HIGH` service.

Benefits include:

- Higher CPU allocation  
- More memory per session  
- Reduced resource contention  
- Better throughput for bulk data operations  

This makes it ideal for Data Pump imports where speed matters.

---


## 4. Import Only User-Managed Schemas

Instead of importing the entire database, restrict imports to **application schemas** and other user-managed objects.

This avoids:

- Pulling in Oracle-maintained schemas  
- Longer import durations  
- Unnecessary metadata processing  

A targeted schema-level import is cleaner and much faster.

---

## Final Thoughts

While this list is not exhaustive, customers who follow these guidelines consistently experience:
faster Data Pump imports with shorter migration windows.
Since Data Pump Import is often the slowest phase of an ADB migration, small performance improvements here can make a big impact.


