---
title: "Size Matters: Why ZDM is the way to migrate 75TB from ADB-S to ADB-D without the Drama"
date: 2026-01-24
draft: false
description: "Why use ZDM to migrate large databases"
tags: ["Oracle", "ADB", "ZDM","Autonomous Database", "GoldenGate"]
categories: ["Database", "Autonomous Database", "ZDM", "GoldenGate"]
---

A customer I am working is in the process of migrating a massive **75TB database** from Autonomous Database Serverless (ADB-S) to Dedicated (ADB-D), which is a high-stakes operation where downtime isn't an option. While Oracle offers a suite of tools like Data Pump and GoldenGate, the sheer volume of data makes the "how" just as important as the "what." At this scale, the primary challenge isn't just moving the bytes; it’s maintaining absolute data integrity while the source database continues to process thousands of transactions per second.

## Zero Downtime Migration (ZDM)

Oracle’s recommended approach is **Zero Downtime Migration (ZDM)**, which acts as a sophisticated orchestrator. Rather than being a standalone "mover," ZDM automates the heavy lifting by combining the strengths of Data Pump and GoldenGate. It uses Data Pump for the initial bulk transfer of the 75TB and GoldenGate to sync the "delta" changes. This eliminates the manual complexity of tracking **System Change Numbers (SCNs)**, ensuring there are no gaps or overlaps in data during the transition.

ZDM provides a "pause and validate" workflow that allows teams to test the 75TB target environment before the final cutover. 

```bash
# Pre-migration evaluation for a risk-free start
zdmcli migrate database -sourcesid SOURCE_ADB -rsp config.rsp -eval
```

## Why not GoldenGate?

Some customers consider using GoldenGate alone, but for a full 75TB migration, this is often a "garden hose vs. swimming pool" scenario. GoldenGate is designed for real-time replication via SQL, which is significantly slower for initial loads than the direct-path methods used by Data Pump. ZDM optimizes this by using Data Pump with high parallelism for the heavy lifting:

```bash
# Typical starting points in environments of this size might look like
DATAPUMP_PARALLELISM=64
DATAPUMP_CHUNK_SIZE_IN_MB=2048
```

Furthermore, GoldenGate doesn't move metadata (like table structures and indexes) automatically. It is far more effective as a "catch-up" mechanism used after the bulk data has been "pre-seeded" into the target.

The "hardest" part of this manual journey is **SCN Management**. In a 75TB environment, the initial export can take days. If the SCN isn't perfectly coordinated between the export and the replication start point, you risk data corruption or "Unique Constraint" errors. ZDM solves this by performing a "handshake" between tools, ensuring that the second GoldenGate starts, it picks up exactly where the Data Pump snapshot left off, preserving transaction atomicity across the entire 75TB landscape. 

## The Pilot vs the Engine vs the Fuel Line

ZDM is not a **magic pill** that will solve data architectural issues. We recently saw a migration where ZDM requested 62 threads of parallelism, but the execution remained single-threaded. The culprit wasn't the pilot (ZDM) or the Engine (Data Pump); it was the Fuel Line (the data structure) itself. Large LOBs or unpartitioned tables act as a bottleneck that stalls even the most powerful migration engines. For a 75TB leap, your data architecture must be 'migration-ready' before the first byte moves.ZDM can orchestrate the move, but it cannot override the physics of unpartitioned data

At this scale, ZDM is less about convenience and more about risk containment. It does not eliminate complexity — it centralizes it.  Teams that assume ZDM “handles everything” still run into issues around redo volume spikes, unexpected Data Pump runtimes, and GoldenGate lag that only becomes visible after the bulk load has already committed. By automating the technical minutiae of log sequence tracking and parallelism tuning, ZDM allows database architects to focus on what matters most: a successful, risk-free transition to Dedicated infrastructure.