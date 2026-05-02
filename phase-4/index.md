---
layout: default
title: "Phase 4: Database Design & Storage"
nav_order: 5
has_children: true
permalink: /phase-4/
---

# Phase 4: Database Design & Storage
{: .no_toc }

**Duration:** Weeks 17–20
{: .label .label-yellow }

Databases are where most system design interviews go deep. Every performance problem at scale is ultimately a data problem — how it's stored, indexed, sharded, and replicated. This phase covers the internals that most engineers treat as a black box: why InnoDB chose B+ Trees, why Cassandra chose LSM trees, why DynamoDB uses a single-table model, and why OLAP systems use columnar storage.
{: .fs-5 }

---

## Topics

| # | Topic | Key Concepts |
|:--|:------|:-------------|
| 1 | [Relational Databases](relational-db/) | MVCC, Index Types, Sharding, Replication Lag, HikariCP |
| 2 | [NoSQL Databases](nosql/) | MongoDB, Cassandra, DynamoDB, Redis, Neo4j, InfluxDB |
| 3 | [Storage Engines](storage-engines/) | InnoDB, LSM Trees, RocksDB, Columnar Storage, S3 |
| 4 | [Data Warehousing & OLAP](data-warehousing/) | Star Schema, BigQuery, Redshift, Snowflake, Apache Iceberg |

---

## Why Phase 4 Matters

Every system design interview eventually hits the storage layer. The questions sound simple but require deep internals to answer well:

- Why does Cassandra write faster than MySQL at scale? (**LSM Trees vs B+ Trees**)
- What happens to your MySQL query plan when the table grows to 100M rows? (**Index internals, query optimization**)
- How does Instagram shard 2 billion user profiles? (**Sharding strategies and trade-offs**)
- Why doesn't Cassandra support joins? (**Wide-column data model constraints**)
- When should you use DynamoDB vs PostgreSQL? (**Access pattern driven OLTP trade-offs**)
- How does BigQuery scan a petabyte in seconds? (**Columnar storage + massively parallel processing**)

Understanding storage engines — not just their APIs — is what separates Principal Engineers from Senior Engineers.

---

## Phase 4 Checklist

- [ ] Explain the four transaction isolation levels and which anomalies each prevents
- [ ] Describe how MVCC works in InnoDB vs PostgreSQL — where are old row versions stored?
- [ ] Explain the difference between a B+ Tree index and a Hash index — when is each optimal?
- [ ] Define a covering index and explain why it eliminates a table lookup
- [ ] Describe three sharding strategies — trade-offs of range, hash, and directory-based
- [ ] Explain replication lag and how read-your-writes consistency is achieved
- [ ] Configure HikariCP connection pool size for a CPU-bound vs I/O-bound workload
- [ ] Describe MongoDB's embedding vs referencing trade-offs for schema design
- [ ] Explain Cassandra's partition key and why partition key design matters more than secondary indexes
- [ ] Design a DynamoDB single-table schema with GSI for a multi-entity access pattern
- [ ] Explain why LSM trees write faster than B+ Trees — and where the read cost goes
- [ ] Describe RocksDB's write path: memtable → WAL → SSTable → compaction
- [ ] Explain how columnar storage (Parquet) accelerates analytical queries vs row storage
- [ ] Explain OLTP vs OLAP trade-offs and why the same database rarely serves both well
- [ ] Design a star schema for an e-commerce data warehouse with fact and dimension tables
