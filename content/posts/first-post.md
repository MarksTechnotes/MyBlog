+++
date = '2025-11-02T17:11:40-06:00'
draft = false
title = 'Smart Scan Demo on Exadata Cloud Service (ExaCS)'
+++


One of the key advantages of Oracle Exadata is Smart Scan — the ability to offload query processing from the database server to the storage cells, reducing data movement across the interconnect and improving performance dramatically.

Here’s a simple demonstration that shows how Smart Scan behaves in practice.

## Setup

Connected to SQL*Plus on an Exadata Cloud Service VM as SYSDBA:
```sql
sqlplus sys/<password>@<exadata-scan-host>:1521/<db_service> as sysdba
```

Replace <password>, <exadata-scan-host>, and <db_service> with your own Exadata Cloud Service connection details as shown in your OCI console or TNS names file.

Check that initial baseline statistics (from v$sysstat and v$mystat) show almost no I/O activity:
```sql
physical read total bytes                       0.0019 MB
cell physical IO interconnect bytes             0.0019 MB
```

## Test 1 — Smart Scan Disabled

Smart Scan can be turned off explicitly using the optimizer parameter CELL_OFFLOAD_PROCESSING = FALSE:
```sql
SELECT /*+ OPT_PARAM('cell_offload_processing', 'false') */ COUNT(*) 
FROM   soe.order_items WHERE  unit_price < 1500;

Elapsed time: 0.99 seconds
Physical reads: ~208 MB
Cell interconnect bytes: ~208 MB
Offload I/O: none
```

This confirms that the query ran without offload; all data was returned from storage to the database node for filtering.

## Test 2 — Smart Scan Enabled

After reconnecting to clear statistics:
```sql
SELECT COUNT(*) FROM soe.order_items WHERE unit_price < 1500;

Elapsed time: 0.13 seconds
Physical reads: ~208 MB
Cell interconnect bytes: 0.30 MB
Offload bytes: ~208 MB eligible for Smart I/O
```

The results speak for themselves — with Smart Scan, the interconnect traffic dropped from 208 MB to just 0.3 MB, and query elapsed time improved nearly 8×.

## Why It’s Faster

With Smart Scan enabled:
- Predicate evaluation (unit_price < 1500) happens in the storage cells, not the database node.
- Only the qualifying rows are sent over the interconnect.
- The buffer cache is bypassed, using PGA memory directly, which is still faster overall.
  
The EXPLAIN PLAN confirms the offload operation:

```sql
|*  2 |  TABLE ACCESS STORAGE FULL | ORDER_ITEMS |
```

## Takeaway

Smart Scan isn’t just a theoretical feature — it’s a tangible performance multiplier. By pushing predicate filtering and column projection down to the storage layer, Exadata reduces data transfer dramatically and speeds up analytics at scale.

Once you see the delta in I/O bytes and timing, you realize why Smart Scan is one of the defining capabilities of Exadata.

