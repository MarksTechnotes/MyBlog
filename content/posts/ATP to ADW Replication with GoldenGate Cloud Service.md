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

### Enable Supplemental Logging 
To prepare the environment for Change Data Capture (CDC), I configured the source at two specific layers of the redo hierarchy:

* **Minimal Supplemental Logging:** This was enabled at the PDB level `(ALTER PLUGGABLE DATABASE ADD SUPPLEMENTAL LOG DATA)` to turn on the "master switch" for the redo stream. This ensures the Oracle logs contain the necessary metadata for GoldenGate to interpret transactions.

* **Object-level Supplemental Logging:** I applied this at the SOE schema level using ADD TRANDATA on the GoldenGate Administration Console. This command ensures that 'Before Images' of primary keys are captured for every operation. By logging the unique identifier of the row—even when the primary key itself isn't being updated—we provide the Replicat with the exact coordinates it needs to build a valid WHERE clause for the target ADW.

### Create Checkpoint Table
I established a **Checkpoint Table** within the GGADMIN schema on the target database directly through the GoldenGate Administration Console to ensure once only replication and enable recovery from network interruptions or process restarts.

### Initial Load via Data Pump `DBMS_DATAPUMP`
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

### Non-Integrated Replicat Mode Selection

For this implementation, I intentionally used a **Non-Integrated** Replicat. The objective was to validate correctness, constraint semantics, and initial-load handoff behavior before introducing apply parallelism. A single-threaded Replicat provides deterministic ordering, clearer diagnostics, and simpler recovery when positioning by trail sequence and RBA. Integrated or Parallel Replicat options become relevant once correctness is established and apply throughput becomes the dominant constraint.

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

## 💡  Dev vs. Prod Strategy

While REPERROR (..., DISCARD) is excellent for clearing initial sync noise in a Development environment, it is risky for Production. Discarding a record in Prod creates "silent data drift"—a scenario where your source and target no longer match without triggering an alert.
For Production: Transition from DISCARD to an Exceptions Table strategy. This allows the Replicat to continue processing while logging failed transactions for audit.

```bash
-- Production-grade mapping example
MAP SOE.*, TARGET SOE.*, 
EXCEPTIONSONLY,
INSERTALLRECORDS, 
MAPEXCEPTION (TARGET SOE.EXCEPTIONS_TABLE);
```

## 4. Trail File Navigation & Final State

I resolved a backlog lag by identifying the specific GoldenGate Trail Sequence 6 and Offset (RBA) - 1778 and manually positioning the Replicat to start from there via the Golden Gate Admin Console. 


## OCI GoldenGate Troubleshooting & Production Reference

| Issue | Symptom | Immediate Resolution (Dev) | Architectural Standard (Prod) |
| :--- | :--- | :--- | :--- |
| **Quota Deficiency** | `DBMS_DATAPUMP` fails | `ALTER USER SOE QUOTA UNLIMITED ON DATA` | Include as part of Pre-Migration checklist |
| **Missing Keys** | `OGG-06439` Warning | `MODIFY CONSTRAINT ... VALIDATE` | Ensure Validation Script is part of the Schema Baseline before sync. |
| **Sync Handoff** | `ORA-1403` / `ORA-02291` | `REPERROR (code, DISCARD)` | **Exceptions Table** with `INSERTALLRECORDS` for data audit. |
| **Connectivity** | Console Timeout | Whitelist GG IP in Database ACL | **Private Endpoints + NSGs** to eliminate public exposure. |
| **Security** | Auth Failure | Sync `GGADMIN` password with Vault Secret | Use **OCI Vault** for dynamic secret rotation and least-privilege users. |


## Note on Environment Scale:

The fixes and parameters documented in this post were performed within a controlled, small-scale development environment and primarily intended to help someone new to the OCI GoldenGate Cloud Service. For production-grade architectures involving multi-terabyte datasets, additional considerations regarding Parallel Replicat tuning, network bandwidth, and high-availability (HA) configuration should be applied.