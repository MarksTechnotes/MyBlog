---
title: "Monitoring Oracle database performance using AAS"
date: 2026-04-15
draft: false
description: "Monitoring Oracle database performance using AAS"
tags: ["AAS", "Oracle", "Performance", "Database"]
categories: ["Database", "Performance"]
---

# A Guide to Average Active Sessions (AAS)

Every DBA has lived through the vague 'database is slow' complaint. The challenge isn't a lack of data—we are flooded with metrics like CPU, Memory, and Storage usage or Total Sessions—but finding the signal in the noise. 

Before you look at the metrics: **Define the Scope**

A common pitfall when receiving a "database is slow" complaint is diving into the metrics too early. The first question should always be: Is everything slow, or is it just one specific process?

    If it's a "Global" slowdown: Use Average Active Sessions (AAS) to check for system-wide bottlenecks (CPU, I/O, or Locks).

    If it's a "Local" slowdown: Drill down into the specific SQL ID or session using ASH.

**AAS** is your tool for the "Global" view—let's look at how to interpret it.

---

## What is AAS?

While *"Total Sessions"* tells you who is logged in, **AAS tells you who is actually doing work.**

Think of it like a highway:

- **Active Sessions** = a snapshot of how many cars are on the road at one exact second  
- **AAS** = the *average* number of cars on that road over time (like an hour or a day)

In Oracle, it’s calculated as:

```
AAS = Total DB Time / Elapsed Wall Clock Time
```

---

## How to Read the Numbers

To understand if your database is healthy, compare your **AAS** to your **CPU Count**.

Use this query to determine the CPU capacity you should compare against:

```sql
SELECT value AS cpu_count
FROM v$parameter
WHERE name = 'cpu_count';
```

| AAS Value     | Status     | Meaning |
|--------------|-----------|--------|
| Close to 0   | Idle      | The database is nearly empty. Most activity is likely background housekeeping. |
| < CPU Count  | Healthy   | The database is processing work efficiently. There are free processors available. |
| > CPU Count  | Bottleneck| The system is overloaded. Sessions are queuing up, causing performance delays. |

For example, if your database has an **AAS of 0.08** and **2 CPUs**, it is only using **4% of its capacity**.

👉 Your database is extremely quiet, and any slowness users feel is likely occurring in the **application or network**, not the database.

A caveat worth mentioning is that while high AAS often correlates with CPU saturation under real workloads, lightweight continuously active sessions—especially internal Oracle tasks—can produce elevated AAS values even when actual OS CPU utilization remains low.

## CPU vs. Wait Sessions

AAS is usually broken down into two parts:

- **CPU Sessions**: Time spent actually processing data  
- **Wait Sessions**: Time spent waiting for resources (disk I/O, network, locks, etc.)

✅ In a healthy system:
- CPU Sessions should be the **largest portion**

🚨 Warning sign:
- High Wait Sessions = database is waiting instead of working

---

## Real-Life Tuning with AAS

DBAs use AAS like **CCTV footage** to diagnose problems:

- **Demand Surges**  
  If AAS spikes above CPU count during high volume → add more CPU cores

- **Locking Issues**  
  A spike in *Concurrency waits* → indicates blocking sessions

- **Bad SQL**  
  If one query dominates AAS → tune that SQL ID

---

*Bottom line: AAS is the pulse of your database. Keep it below your CPU count, and you're in the clear!*

Knowing your database is busy is only half the battle. In Part 2, we’ll dive into the specific **Wait Classes** that drive high AAS and how to identify exactly what resource is holding your data hostage.