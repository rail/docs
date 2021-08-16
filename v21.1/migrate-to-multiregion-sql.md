---
title: Migrate to Multi-region SQL
summary: Learn how to migrate to CockroachDB's improved multi-region SQL user experience.
toc: true
---

## Overview

In CockroachDB v21.1, we added support for improved [multi-region capabilities](multiregion-overview.html). Using these capabilities, you can use high-level SQL statements to control where your data is stored and how it is accessed to provide the best, lowest-latency experience for your application's users.

Prior to v21.1, there was support for accomplishing many of the same goals using low-level mechanisms known as [replication zones](configure-replication-zones.html).

This page has:

- Descriptions of the mapping between the legacy replication-zone-based workflows and the multi-region SQL abstractions that should replace them for most users.
- Instructions for migrating from replication zones to multi-region SQL abstractions.

NOTE: Most users operating multi-region databases will be better served by the multi-region SQL statements.  Most users do not need to use the replication-zone-based workflows mentioned here.

## Mapping from legacy to new

Old | New
--- | ---
Duplicate Indexes | GLOBAL tables
Geo-partitioned replicas | REGIONAL tables with ZONE survival goal
Geo-partitioned leaseholders | REGIONAL tables with REGION survival goal

For more information about how to choose between GLOBAL and REGIONAL tables, see [When to use REGIONAL vs GLOBAL tables]().

For more information about how to choose between ZONE and REGION survival goals, see [When to use ZONE vs REGION survival goals]().

## How to migrate a database from replication zones to multi-region SQL

The following instructions assume that you will re-use your existing cluster regions (that is, regions added during `cockroach start --locality`) with the new SQL abstractions.  If you DO want to use new regions altogether, you must shutdown and start each node fresh with the new regions (again, via `cockroach start --locality`)

### Step 1. Remove old replication zone configurations

### Step 2. Add a primary region to your database

ALTER DATABASE ... SET PRIMARY REGION = XXX

### Step 3. Maybe add more nodes to the cluster, in more regions

If you are happy with your existing nodes and regions, this step is optional.  If you are going to add more nodes, the best time to do it is now.

### Step 4. Add more regions to the database

ALTER DATABASE ... ADD REGION XXX

### Step 5. Just wait longer

After running the commands described above, the cluster will probably take a performance hit. With all of the data shuffling and network traffic resulting from the fact that you are asking the clsuter to move through several successive large state changes in succession, the performance impact could be substantial.  Therefore, we recommend running this procedure during a time when you would normally schedule maintenance or downtime, e.g. 3am on a Tuesday.

But you may ask: _why_ is there such a performance impact? And why do these steps need to be done in this order?

At a high level, the answer is that the underlying replication zone mechanisms that determine where to place CockroachDB's data are in fact a constraint propagation system.  And every time you add a new constraint to the already existing set, CRDB has to check the new constraint against the existing set, and possibly do work to make sure that it is now meeting N+1 constraints (whereas before it was meeting N constraints).

Therefore if you (1) add a bunch of new nodes at time T without changing your multi-region config (really, constraints), the cluster will do a bunch of work rebalancing among those nodes based on CRDB's "default" set of constraints C in place at time T, and then when you (2) add a primary region at a later time T+1, yielding a new "narrower" set of constraints C', the cluster AGAIN has to do a bunch of work rebalancing etc. to meet the new constraint set C' which is now in place at time T+1. The fact that the constraints started out "wide" and got "narrow" results in a lot of data movement etc.

By contrast, if you add a primary region first thing at time T when you have one node, the initial set of constraints is already "narrowed" significantly, so any new constraints you add at time T+1, ..., etc when you add a bunch of new nodes will be validated against the already existing "narrower" constraints and the data will go where you want it more quickly without so much rebalancing (since you avoided the wide constraint -> narrow constraint change described above)

Basically it's like a constraint propagation system that will be faster if you narrow the set of constraints as soon as possible during cluster setup and before adding more nodes, so it has a narrower range of things to consider (thus resulting in less churn / data movement / network traffic / etc)

## See also

- [Multi-region overview](multiregion-overview.html)
- [When to use `REGIONAL` vs. `GLOBAL` tables](when-to-use-regional-vs-global-tables.html)
- [When to use `ZONE` vs. `REGION` survival goals](when-to-use-zone-vs-region-survival-goals.html)
- [Topology Patterns](topology-patterns.html)
- [Disaster Recovery](disaster-recovery.html)
- [Multi-region SQL performance](demo-low-latency-multi-region-deployment.html)
