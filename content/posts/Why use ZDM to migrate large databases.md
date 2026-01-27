---
title: "Why use ZDM or DMS to migrate large databases from ADB-S to ADB-D without the drama"
date: 2026-01-27
draft: false
description: "Why use ZDM or DMS to migrate large databases"
tags: ["Oracle", "ADB", "ZDM","Autonomous Database", "DMS", "Data Pump", "GoldenGate"]
categories: ["Database", "Autonomous Database", "ZDM", "DMS", "Data Pump", "GoldenGate"]
---

A customer I am working with is in the process of migrating a massive **75TB database** from Autonomous Database Serverless (ADB-S) to Dedicated (ADB-D), which is a high-stakes operation where downtime isn't an option. In addition, ADB-S has bi-weekly/monthly maintenance windows which can interfere with the migration process if it takes too long, that makes it critical to complete the migration in a timely manner.  While Oracle offers a suite of tools like Data Pump and GoldenGate, the sheer volume of data makes the "how" just as important as the "what.". At this scale, the primary challenge isn't just moving the bytes; it’s maintaining absolute data integrity while the source database continues to process thousands of transactions per second.

## Zero Downtime Migration (ZDM) and Database Migration Service (DMS)

Oracle’s recommended approach is **Zero Downtime Migration (ZDM)** or its cloud-native evolution, OCI **Database Migration Service (DMS)**. Both act as sophisticated orchestrators. Rather than being standalone "movers," they automate the heavy lifting by combining Data Pump (for the 75TB initial load) and GoldenGate (to sync "delta" changes).

While ZDM is a powerful CLI tool you run on your own VM, DMS is a fully managed service in the OCI Console. DMS essentially provides a "managed cockpit" for the ZDM engine, which is a major advantage for 75TB moves because it offloads the resource requirements of the migration host to Oracle's managed infrastructure.

Whether you use the CLI or the Console, these tools automate the heavy lifting by combining the strengths of Data Pump and GoldenGate. Since this is an online logical migration, the source database will be online and accessible to users during the migration process.

ZDM provides a "pause and validate" workflow that allows teams to test the 75TB target environment before the final cutover. 

```bash
# Pre-migration evaluation for a risk-free start
zdmcli migrate database -sourcesid SOURCE_ADB -rsp config.rsp -eval
```

## Why not GoldenGate alone?

Some customers consider using GoldenGate alone, but for a full 75TB migration, this is often a "garden hose vs. swimming pool" scenario. GoldenGate is designed for real-time replication via SQL, which is significantly slower for initial loads than the direct-path methods used by Data Pump. ZDM optimizes this by using Data Pump with high parallelism for the heavy lifting:

```bash
# Typical starting points in environments of this size might look like
DATAPUMP_PARALLELISM=64
DATAPUMP_CHUNK_SIZE_IN_MB=2048
```

 GoldenGate is far more effective as a "catch-up" mechanism used after the bulk data has been "pre-seeded" into the target.

The "hardest" part of this manual journey is **SCN Management**. In a 75TB environment, the initial export can take days. If the SCN isn't perfectly coordinated between the export and the replication start point, you risk data corruption or "Unique Constraint" errors. ZDM solves this by performing a "handshake" between tools, ensuring that the second GoldenGate starts, it picks up exactly where the Data Pump snapshot left off, preserving transaction atomicity across the entire 75TB landscape. 

## The Pilot vs the Engine vs the Fuel Line

Orchestration via DMS/ZDM is not a **magic pill** that will solve data architectural issues. In the migration I was referring to earlier, the customer configured ZDM for 62 threads of parallelism, but the execution remained single-threaded. The culprit wasn't the pilot (ZDM) or the Engine (Data Pump); it was the Fuel Line (the data structure) itself. Large LOBs or unpartitioned tables act as a bottleneck that stalls even the most powerful migration engines. For a 75TB leap, your data architecture must be 'migration-ready' before the first byte moves. ZDM can orchestrate the move, but it cannot override the physics of unpartitioned data.

```sql
-- Check if your 'Fuel Line' is blocked: 
-- Are workers actually working, or just idling?
SELECT worker_method, state, count(*) 
FROM v$datapump_worker 
GROUP BY worker_method, state;
```

## Bandwidth Requirement 

Don’t settle for a 1 Gbps connection. At 75TB, that leads to a theoretical 11-day transfer—and that's before you account for index rebuilding or metadata. Even with optimized throughput, a migration of this scale can take a week or longer.

To hit that window, you need:   
    **10 Gbps Minimum:** Anything less turns a week-long project into a month-long ordeal.  
    **Target Scaling:** Scale your Target ADB-D to at least 32-64 ECPUs during the migration to handle the write volume.    
    **Managed Infrastructure:** If using ZDM CLI, ensure your host has at least 8-16 OCPUs. If you want to bypass the risk of a "small VM" bottleneck entirely, DMS is the better fit as Oracle manages the migration resources for you.
    
## Conclusion

At 75TB, choosing between ZDM and DMS is about risk containment. While ZDM offers CLI-level granularity, DMS provides a streamlined, managed experience that is often a better fit for cloud-native architectures. Neither tool eliminates complexity—they centralize it. Teams must still account for redo volume spikes and ensure their data structure allows for parallelism. By automating SCN tracking and parallelism tuning, these tools allow architects to focus on the goal: a successful, risk-free transition to Dedicated infrastructure.