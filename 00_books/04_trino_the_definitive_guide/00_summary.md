# Trino: The Definitive Guide - Summary

## 📚 Book Overview

**Authors:** Matt Fuller, Manfred Moser, Martin Traverso
**Publisher:** O'Reilly Media
**Focus:** SQL at any scale, on any storage, in any environment

This book provides a comprehensive guide to Trino (formerly PrestoSQL), a distributed SQL query engine designed for querying data across diverse data sources at massive scale.

---

# 📊 Summary Tables

## Trino vs. Traditional Databases

| Aspect | Trino | Traditional Database |
|--------|-------|---------------------|
| Storage | External (HDFS, S3, RDBMS) | Internal |
| Compute | Distributed cluster | Single node or limited cluster |
| Query Scope | Cross-data source federation | Single database |
| Data Movement | None (query in place) | Required |
| Use Case | OLAP, analytics | OLTP, mixed |
| SQL Support | ANSI SQL | Dialect-specific |

## Configuration Files

| File | Purpose | Location |
|------|---------|----------|
| `config.properties` | Server configuration | etc/ |
| `node.properties` | Node-specific config | etc/ |
| `jvm.config` | JVM options | etc/ |
| `log.properties` | Logging levels | etc/ |
| `catalog/*.properties` | Data source catalogs | etc/catalog/ |
| `access-control.properties` | Access control | etc/ |
| `password-authenticator.properties` | LDAP auth | etc/ |
| `resource-groups.properties` | Resource groups | etc/ |

## Essential SQL Commands

| Category | Commands |
|----------|----------|
| **Discovery** | `SHOW CATALOGS`, `SHOW SCHEMAS`, `SHOW TABLES`, `DESCRIBE` |
| **Metadata** | `SHOW FUNCTIONS`, `SHOW STATS`, `SHOW SESSION` |
| **Query** | `SELECT`, `WITH`, `UNION`, `JOIN`, subqueries |
| **DDL** | `CREATE SCHEMA`, `CREATE TABLE`, `ALTER TABLE`, `DROP TABLE` |
| **DML** | `INSERT`, `DELETE` |
| **Utility** | `EXPLAIN`, `PREPARE`, `EXECUTE`, `CALL` |

## Performance Tuning Checklist

- [ ] Enable cost-based optimizer (`join_reordering_strategy = 'AUTOMATIC'`)
- [ ] Collect table statistics (`ANALYZE`)
- [ ] Review query plans (`EXPLAIN`)
- [ ] Check memory configuration
- [ ] Monitor Web UI for bottlenecks
- [ ] Configure resource groups for workload isolation
- [ ] Tune JVM garbage collection
- [ ] Consider caching (RubiX)

---

*"Trino is a tool designed to efficiently query vast amounts of data by using distributed execution."*