---
title: "Real-Time Replication: ATP to ADW with OCI GoldenGate"
date: 2026-02-17
draft: false
description: "Real Time replcation from ATP to ADW with GoldenGate Cloud Service"
tags: ["Oracle", "ATP", "ADW", "Data Pump", "Golden Gate", "Autonomous Database"]
categories: ["Database", "Autonomous Database", "Data Pump", "Golden Gate"]
---
**Target Audience:** Senior DBAs, Cloud Architects, and Oracle User Groups  
**Technical Context:** Replicating SOE schema between 4-ECPU Autonomous instances

---

## 1. Environment & Performance Profile
To provide a baseline for these benchmarks, both the source and target environments were configured as follows:

* **Source:** ATP (`dbai26`) — **4 ECPUs**
* **Target:** ADW (`PerfDB23ai`) — **4 ECPUs**
* **Data Volume:** ~700 MB (SOE Schema)

### Initial Load via `DBMS_DATAPUMP`
I used the `DBMS_DATAPUMP` PL/SQL package to integrate directly with OCI Object Storage via `DBMS_CLOUD` credentials.

* **Export Time:** 266 seconds
* **Import Time:** 218 seconds

> ### 🛑 The Quota Requirement
> During the setup, **NOT granting the target schema `UNLIMITED QUOTA ON DATA` caused the Data Pump job to fail.** Even with `DWROLE` assigned, the import engine requires explicit space permissions to materialize objects.
>
> **The Fix:**
> ```sql
> ALTER USER SOE QUOTA UNLIMITED ON DATA;
> ```

---

## 2. Solving the "Invisible" Key Problem (OGG-06439)
Post-import, the GoldenGate Replicat threw:  
`WARNING OGG-06439: No unique key is defined for table ORDERS.`

### The Diagnosis
In Autonomous environments, Data Pump imports constraints in an `ENABLE NOVALIDATE` state. While the DB enforces these for *new* data, GoldenGate ignores them because it has not verified that the *existing* data follows the rule.

### The Fix
Validate all Primary and Unique keys in the `SOE` schema using this PL/SQL loop:

```sql
BEGIN
  FOR r IN (SELECT owner, table_name, constraint_name 
            FROM all_constraints 
            WHERE owner = 'SOE' 
              AND status = 'ENABLED' 
              AND validated = 'NOT VALIDATED'
              AND constraint_type IN ('P', 'U')) 
  LOOP
    EXECUTE IMMEDIATE 'ALTER TABLE ' || r.owner || '.' || r.table_name || 
                      ' MODIFY CONSTRAINT ' || r.constraint_name || ' VALIDATE';
  END LOOP;
END;
```

## 3. Surgical Error Handling (ORA-1403 & ORA-02291)

To bridge the gap between the Data Pump snapshot and real-time trails, I utilized the `REPERROR` parameter for precision handling.

* **ORA-1403 (No Data Found):**  Occurs if Replicat updates a row not in the initial snapshot.
* **ORA-02291 (Parent Key Not Found):** Occurs if child records are processed before parents during hand-off.

**Replicat Parameter Configuration:**

```bash
REPLICAT rep_soe
USERIDALIAS adw_target_alias
-- Precision Error Handling (Avoids HANDLECOLLISIONS)
REPERROR (1403, DISCARD)
REPERROR (2291, DISCARD)
MAP SOE.*, TARGET SOE.*;
```
## 4. Trail File Navigation & Final State

I resolved a backlog lag by identifying the specific GoldenGate Trail Sequence 6 and Offset (RBA) - 1778 and manually positioning the Replicat to start from there via the Golden Gate Admin Console. 

# OCI GoldenGate Troubleshooting Reference

| Issue | Symptom | Technical Resolution |
| :--- | :--- | :--- |
| **Quota Deficiency** | `DBMS_DATAPUMP` fails | Grant `UNLIMITED QUOTA ON DATA` to schema. |
| **Missing Keys** | `OGG-06439` Warning | Move constraints from `NOVALIDATE` to `VALIDATE`. |
| **Sync Handoff** | `ORA-1403` / `ORA-02291` | Add `REPERROR (code, DISCARD)` to params. |
| **Connectivity** | Admin Console timeout | Whitelist OCI-GG Deployment IP in ADB ACL. |
| **Security** | Auth Failure | Match `GGADMIN` password during unlock to Vault Secret exactly. |