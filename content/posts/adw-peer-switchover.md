---
title: "Is Peer Database Role Switchover (ADW Serverless → ADW Dedicated) supported?"
date: 2025-11-02
draft: false
description: "Clarifying whether a Data Guard–based role switchover from Autonomous Data Warehouse Serverless to Dedicated is supported, and the recommended migration path."
tags: ["Oracle", "Autonomous Database", "ADW", "ZDM", "Data Guard", "Migration"]
categories: ["Database", "Oracle Cloud"]
---

When planning a migration from **Autonomous Data Warehouse (ADW) Serverless** to **ADW Dedicated**, one question that often comes up is whether you can use a **peer or standby database** setup — similar to a traditional **Data Guard switchover** — to transition environments.

Let’s break down what that would involve and what Oracle actually supports.

---

### The Proposed Scenario

The idea typically goes like this:

1. Set up a **standby (peer) database** on **Exadata Dedicated (19c)**.  
2. Perform a **role switchover**, making the Dedicated database the new primary.  
3. Later, upgrade to **23ai** as part of the post-migration plan.

This approach essentially models a **Data Guard–based migration**, using role switchovers to minimize downtime.

---

### Is This Supported?

**No**, it is not supported.

You **cannot** set up a peer or standby database for an **Autonomous Data Warehouse  (ADW-S)** instance on an **ADW Dedicated (ADW-D)** instance.  
**Cross-type Data Guard configurations** — between Serverless and Dedicated — are **not supported**.

This restriction exists because the **infrastructure and management models** of the two environments are fundamentally different. Autonomous Serverless and Dedicated operate on separate control planes, with distinct automation and resource management layers.  As a result, Oracle does not allow **mixing the two** for replication, standby, or peer configurations.

---

### Recommended Migration Path

For migrations between ADW Serverless and Dedicated environments, Oracle recommends using **Zero Downtime Migration (ZDM)**.

**ZDM** provides a controlled, Oracle-supported workflow for:
- Moving data and metadata
- Validating the migration
- Managing the cutover with minimal downtime
- Maintaining compatibility with future upgrades (e.g., **19c → 23ai**)

This is the approach used in production-grade migrations today and aligns with Oracle’s official guidance.

---

### Summary

| Approach | Supported | Notes |
|-----------|------------|-------|
| Data Guard Role Switchover (ADW-S → ADW-D) | ❌ No | Cross-type configurations unsupported |
| Zero Downtime Migration (ZDM) | ✅ Yes | Oracle-recommended for Serverless → Dedicated migrations |

---

### Final Thoughts

While a Data Guard–style switchover might seem appealing conceptually, it’s simply not feasible across Autonomous Serverless and Dedicated boundaries.  
**ZDM** remains the correct and supported way to move from **ADW Serverless** to **ADW Dedicated**, ensuring both data integrity and compliance with Oracle’s operational model.

---


