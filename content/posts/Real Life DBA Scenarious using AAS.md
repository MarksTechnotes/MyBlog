---
title: "Real-Life DBA Scenarios using AAS"
date: 2026-04-09
draft: false
description: "Real-Life DBA Scenarios using AAS"
tags: ["AAS", "Oracle", "Performance", "Database"]
categories: ["Database", "Performance"]
---

# Real-Life DBA Scenarios Using AAS

Here are three real-life scenarios where DBAs use **Average Active Sessions (AAS)** to solve performance issues.

---

## 1. The "Demand Surge" (Scaling Check)

Imagine a retail site during a flash sale. The DBA sees **AAS jump from 2.0 to 15.0** on a server with **8 CPUs**.

**Diagnosis:**
- Because **AAS > CPU count**, the system is saturated.

**Fix:**
- By drilling down into the AAS *Wait Class*, the DBA sees it is mostly **CPU activity**.
- This proves the hardware is too small for peak load.
- The solution is an immediate vertical scale-up (add more CPU cores).

---

## 2. The "Locking Disaster" (Application Bug)

A payroll application suddenly freezes. The total CPU usage on the server looks low (5%), but the AAS is 12.0 on a 24-CPU machine.

**Diagnosis:**
- The DBA checks the AAS chart and sees a massive block of **Concurrency waits** (often shown in dark red).

**Fix:**
- Filter AAS by **Top Sessions** and **Blocking Session**
- Identify one user session that forgot to COMMIT
- This single session is blocking 11 others, creating a queue

---

## 3. The "Bad Execution Plan" (Query Tuning)

After a software update, a report slows from **10 seconds to 10 minutes**. The overall AAS increases from **0.5 to 1.5**.

**Diagnosis:**
- Using **ASH Analytics**, the DBA filters AAS by **SQL ID**
- One query accounts for 1.0 of the 1.5 AAS
- The wait class is **User I/O**

**Fix:**
- The query is doing a ull table scan
- Add the missing index
- AAS contribution drops back near zero

---

## Summary of Rules of Thumb

| Observation | Likely Conclusion |
|------------|------------------|
| Spike in AAS (Mostly CPU) | Too much work for processors; tune SQL or add CPU cores |
| Spike in AAS (Mostly Concurrency) | Application locking issue; find the blocking session |
| Spike in AAS (Mostly I/O) | Storage bottleneck or missing indexes causing heavy disk reads |
