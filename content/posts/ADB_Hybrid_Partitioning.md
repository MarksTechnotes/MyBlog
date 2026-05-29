---
title: "Implementing Data Archival in Autonomous Database Using Hybrid Partitioned Tables"
date: 2026-05-28
draft: false
description: "Use Hybrid Partitioned Tables and OCI Object Storage to build a cost-effective archival strategy for long-term data retention in Autonomous Database."
tags: ["Autonomous Database", "ADB-Dedicated", "Hybrid Partitioning", "Data Archival", "OCI", "DBMS_CLOUD"]
categories: ["Autonomous Database", "Architecture"]
---

# Implementing Data Archival in Autonomous Database using Hybrid Partitioned Tables

## Introduction

Organizations often need to retain historical data for compliance, reporting, and audit purposes while controlling database storage costs. In Autonomous Database (ADB), Hybrid Partitioned Tables provide an effective approach for keeping recent data in database storage while offloading older, infrequently accessed data to lower-cost object storage.

This architecture allows applications and reporting tools to continue querying a single logical table while reducing the amount of expensive database storage consumed by historical data.

---

## 1. Using Hybrid Partitioning to Reduce Storage Costs

A Hybrid Partitioned Table contains both:

- **Internal partitions** stored inside Autonomous Database storage.
- **External partitions** stored outside the database, typically in OCI Object Storage.

A common lifecycle pattern is:

| Data Age | Storage Location |
|-----------|------------------|
| Recent operational data | Internal partitions |
| Historical read-mostly data | External partitions in Object Storage |

Benefits include:

- Reduced Autonomous Database storage consumption
- Transparent SQL access across all partitions
- Partition pruning for efficient query performance
- Lower total cost of ownership for long-term data retention
- Simplified archival strategy

The database optimizer automatically accesses only the relevant partitions during query execution.

---

## 2. Creating Hybrid Partitioned Tables in Autonomous Database

Autonomous Database provides the `DBMS_CLOUD.CREATE_HYBRID_PART_TABLE` procedure to create Hybrid Partitioned Tables backed by cloud object storage.

### Example

```sql
BEGIN
  DBMS_CLOUD.CREATE_HYBRID_PART_TABLE(
    table_name => 'SALES_HPT',
    credential_name => 'OBJ_STORE_CRED',
    format => json_object(
      'delimiter' value ',',
      'recorddelimiter' value 'newline'
    ),
    column_list => '
      sale_id      NUMBER,
      sale_date    DATE,
      amount       NUMBER
    ',
    partitioning_clause => '
      PARTITION BY RANGE (sale_date)
      (
        PARTITION p2023 VALUES LESS THAN (DATE ''2024-01-01'')
          EXTERNAL LOCATION (
            ''https://objectstorage.region.oraclecloud.com/archive/sales_2023.csv''
          ),
        PARTITION p2024 VALUES LESS THAN (DATE ''2025-01-01''),
        PARTITION pmax VALUES LESS THAN (MAXVALUE)
      )'
  );
END;
/
```
### Example External CSV File

The external partition in the example could reference a CSV file stored in OCI Object Storage with the following contents:

```csv
1001,2023-01-15,1250.50
1002,2023-02-01,987.25
1003,2023-03-12,1500.00
1004,2023-04-30,725.75
1005,2023-06-18,2200.00
1006,2023-08-05,1100.25
1007,2023-10-22,875.00
1008,2023-12-31,1999.99
```

The file maps to the following columns:

| Column | Data Type |
|----------|----------|
| sale_id | NUMBER |
| sale_date | DATE |
| amount | NUMBER |

Because the format specifies a comma delimiter and newline record delimiter, each row in the file represents a single record and no header row is required.

In this example:

- `p2023` is an external partition stored in OCI Object Storage.
- `p2024` and `pmax` remain internal database partitions.
- Queries against the table automatically access the appropriate partition.

---

## 3. Backup and Standby Strategy

Hybrid Partitioned Tables require two protection strategies because the data resides in two different locations.

### Internal Partitions

Internal partitions are protected by standard Autonomous Database capabilities:

- Automatic Autonomous Database backups
- Point-in-time recovery (PITR)
- Autonomous Data Guard (when configured)
- Database restore operations

### External Partitions

External partitions are not stored inside database data files.

Protection is provided by the storage platform hosting the external files, typically OCI Object Storage:

- Object Storage durability
- Object Storage versioning
- Cross-region replication (optional)
- Retention and lifecycle policies

### Important Consideration

Autonomous Database backups and standby databases protect:

- Table metadata
- Internal partitions
- Partition definitions

They do **not** back up or replicate the external data files themselves.

If external files are lost, they must be recovered through the Object Storage protection strategy rather than through database recovery.

---

## 4. Restrictions on External Partitions

External partitions are intended for historical and read-only data.

Key restrictions include:

### Read-Only Access

External partitions do not support:

- INSERT
- UPDATE
- DELETE
- MERGE

Data must be modified outside the database and republished as external files.

### No Transactional Processing

External partitions do not participate in:

- Row locking
- Undo generation
- Redo generation
- ACID transactional updates

### Archive-Oriented Use Cases

External partitions are best suited for:

- Historical reporting
- Compliance retention
- Audit access
- Long-term archival storage

They are generally not suitable for:

- Active OLTP workloads
- Frequently updated historical records
- Operational transaction processing

---

## Conclusion

Hybrid Partitioned Tables provide an elegant archival architecture for Autonomous Database. By keeping active data in internal partitions and moving historical data to OCI Object Storage, organizations can significantly reduce storage costs while preserving transparent SQL access to retained data.

For long-term retention use cases such as utility billing, metering, compliance, and regulatory reporting, Hybrid Partitioned Tables offer a practical balance between performance, accessibility, and cost efficiency.

## References

1. Oracle Autonomous Database – Query Hybrid Partitioned Data
   https://docs.oracle.com/en/cloud/paas/autonomous-database/serverless/adbsb/query-hybrid-partition.html

2. Oracle Autonomous Database Dedicated – DBMS_CLOUD Documentation
   https://docs.oracle.com/en/cloud/paas/autonomous-database/dedicated/adbdm/

3. Oracle Database 19c – Managing Hybrid Partitioned Tables
   https://docs.oracle.com/en/database/oracle/oracle-database/19/vldbg/manage_hypt.html

4. Database Heartbeat – Save Storage Cost with Hybrid Partitioned Tables in Oracle Autonomous Database
   https://database-heartbeat.com/2021/03/23/save-storage-cost-with-hybrid-partitioned-tables-in-oracle-autonomous-database/
