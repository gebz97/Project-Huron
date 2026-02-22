# Apache Iceberg: The Definitive Guide - Summary

## 📚 Book Overview

**Authors:** Tomer Shiran, Jason Hughes, Alex Merced (all from Dremio)
**Publisher:** O'Reilly Media
**Focus:** Data Lakehouse functionality, performance, and scalability with Apache Iceberg

This book provides a comprehensive guide to Apache Iceberg, a high-performance table format for data lakes that brings data warehouse-like capabilities (ACID transactions, time travel, schema evolution) to open data lake storage.

---

# 📊 Key Takeaways

## Iceberg vs. Hive Table Format

| Feature | Hive | Iceberg |
|---------|------|---------|
| Table definition | Directories | List of files |
| ACID transactions | No | Yes |
| Concurrent writes | Unsafe | Safe |
| Partition evolution | Rewrite table | Add/drop fields |
| Schema evolution | Unsafe | Safe |
| Hidden partitioning | No | Yes |
| Time travel | No | Yes |
| Statistics | Async jobs | Per-write |
| Query planning | List directories | Metadata scans |

## Iceberg Architecture Summary

```
Catalog (atomic pointer to current metadata)
    ↓
Metadata File (schema, partition spec, snapshots)
    ↓
Manifest List (list of manifests for snapshot)
    ↓
Manifest Files (datafile list + column stats)
    ↓
Data Files (Parquet/ORC/Avro) + Delete Files
```

## When to Use Different Features

| Need | Feature |
|------|---------|
| Safe concurrent writes | ACID transactions + optimistic concurrency |
| Change partition scheme | Partition evolution |
| Query without partition awareness | Hidden partitioning |
| Update/delete rows | COW (read-heavy) or MOR (write-heavy) |
| Query historical data | Time travel |
| Undo mistakes | Version rollback |
| Isolate testing | Branching |
| Multi-table consistency | Nessie catalog |
| Faster filtered queries | Sorting, Z-order, partitioning |
| Fewer files | Compaction |
| Avoid throttling | Object storage optimization |
| Approximate distinct counts | Puffin files + Theta sketches |

---

*"Data is a primary asset from which organizations curate the information and insights needed to make critical business decisions."*