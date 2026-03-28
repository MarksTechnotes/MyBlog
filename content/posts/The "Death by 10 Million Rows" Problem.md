---
title: "The 'Death by 10 Million Rows' Problem: Why Big Transactions Fail"
date: 2026-03-28
draft: false
description: "The 'Death by 10 Million Rows' Problem: Why Big Transactions Fail"
tags: ["Oracle", "ADB", "Autonomous Database", "Big Transactions"]
categories: ["ADB", "Autonomous Database"]
---

# The "Death by 10 Million Rows" Problem: Why Big Transactions Fail

We’ve all been there: You have a "simple" update to run on a massive table. You hit execute on a single `UPDATE` statement, and then... you wait. Suddenly, the DBA is at your desk because you’ve locked the table for everyone else and the **Undo Tablespace** is screaming for mercy.

Here’s why "one big transaction" is usually a bad idea—and how to break it down.

---

### The Single-Statement Trap
When you run a single `UPDATE` on 10 million rows, Oracle treats it as a single, indivisible unit of work.

```sql
UPDATE orders 
SET status = 'PROCESSED' 
WHERE status = 'PENDING';

COMMIT;
```

 This creates three massive bottlenecks:

*   **Undo Exhaustion**: Oracle must store the "before" image of all 10 million rows to support **read consistency**. If your Undo space isn't huge, the query crashes.
*   **Locking Gridlock**: Every row you touch is locked until you `COMMIT`. This stops other users from modifying that same data.
*   **The All-or-Nothing Risk**: If the database hiccups at row 9,999,999, the whole thing rolls back. You’ve wasted hours of processing time with zero results.

---

### The Solution: Logical Batching
Instead of one giant leap, take 200 small steps. By breaking the work into batches (e.g., 50,000 rows at a time), you keep the database "breathing."

#### The Pro Code Pattern: PL/SQL Bulk Processing
Using `BULK COLLECT` with a `LIMIT` clause is the gold standard. It minimizes the "context switching" between the SQL and PL/SQL engines.

```sql
DECLARE
  CURSOR c_huge_data IS 
    SELECT rowid FROM orders WHERE status = 'PENDING';
    
  TYPE t_rowid IS TABLE OF UROWID;
  v_rowids t_rowid;
BEGIN
  OPEN c_huge_data;
  LOOP
    -- 1. Grab a manageable "chunk" (e.g., 50k rows)
    FETCH c_huge_data BULK COLLECT INTO v_rowids LIMIT 50000;
    EXIT WHEN v_rowids.COUNT = 0;

    -- 2. Update just that chunk
    FORALL i IN 1..v_rowids.COUNT
      UPDATE orders SET status = 'PROCESSED' WHERE rowid = v_rowids(i);

    -- 3. Commit to free up Undo space and release locks
    COMMIT; 
  END LOOP;
  CLOSE c_huge_data;
END;
```

### Summary: The Golden Rules

*   **Don't commit too often**: Committing after every single row creates massive overhead. Aim for a "sweet spot" batch size, typically between **5,000 to 50,000 rows**.
*   **Watch the logs**: High-volume batching generates significant **Redo Log** activity; ensure your storage can handle the IOPS.
*   **Consider CTAS**: If you are updating a vast majority of the table, it is often faster to use **Create Table As Select (CTAS)** to build a new version of the table rather than updating the existing one.
*   **Monitor Undo**: Even with batching, keep an eye on your **Undo Tablespace** to prevent "Snapshot Too Old" errors.
