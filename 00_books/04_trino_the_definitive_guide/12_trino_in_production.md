
## Chapter 12: Trino in Production

### Trino Web UI Monitoring

**Cluster-Level Metrics:**
- Running/Queued/Blocked Queries
- Active Workers
- Runnable Drivers
- Reserved Memory
- Rows/Sec, Bytes/Sec
- Worker Parallelism

**Query List:**
- Filter by user, source, state, query ID
- Sort by time, duration
- States: RUNNING, FINISHED, QUEUED, BLOCKED, PLANNING, FAILED, USER ERROR, USER CANCELLED

**Query Details:**
- Completed/Running/Queued Splits
- Wall Time, CPU Time
- Memory usage (current, peak, cumulative)
- Stages and tasks
- Live Plan view
- Stage Performance view
- Splits timeline
- JSON export

### Performance Tuning

**Check Statistics:**
```sql
SHOW STATS FOR flights;
```

**Analyze Plan:**
```sql
EXPLAIN (TYPE DISTRIBUTED) SELECT ...;
```

**Join Tuning Properties:**
```properties
# Session
SET SESSION join_reordering_strategy = 'AUTOMATIC';
SET SESSION join_reordering_strategy = 'ELIMINATE_CROSS_JOINS';
SET SESSION join_reordering_strategy = 'NONE';

# Configuration
optimizer.join-reordering-strategy=AUTOMATIC
optimizer.max-reordered-joins=9
```

**Partial Aggregation Tuning:**
```sql
SET SESSION push_aggregation_through_join = true|false;
```

**Configuration:**
```properties
task.max-partial-aggregation-memory=16MB
```

### Memory Management

**Memory Types:**
- **User memory** - Aggregations, sorting
- **System memory** - Readers, writers, buffers, scans

**Memory Properties:**

| Property | Description |
|----------|-------------|
| `query.max-memory-per-node` | Max user memory per node |
| `query.max-total-memory-per-node` | Max user + system memory per node |
| `query.max-memory` | Max user memory cluster-wide |
| `query.max-total-memory` | Max user + system memory cluster-wide |
| `memory.heap-headroom-per-node` | JVM headroom |

**Error Messages:**
- `EXCEEDED_LOCAL_MEMORY_LIMIT` - Per-node limit exceeded
- `EXCEEDED_GLOBAL_MEMORY_LIMIT` - Cluster limit exceeded

**Out-of-Memory Policy:**
```properties
query.low-memory-killer.policy=total-reservation|total-reservation-on-blocked-nodes
```

**Example Configuration (10 workers, 50GB RAM each):**
```properties
query.max-memory-per-node=13GB
query.max-total-memory-per-node=16GB
query.max-memory=50GB
query.max-total-memory=60GB
memory.heap-headroom-per-node=9GB
```

### Task Concurrency

```properties
task.max-worker-threads=24  # CPUs × 2
task.concurrency=16          # Default
```

### Worker Scheduling

```properties
node-scheduler.max-splits-per-node=100
node-scheduler.max-pending-splits-per-task=10
node-scheduler.network-topology=flat  # or legacy
```

### Network Data Exchange

```properties
exchange.client-threads=25
sink.max-buffer-size=32MB
exchange.max-buffer-size=32MB
```

### JVM Tuning

**jvm.config (production):**
```
-server
-XX:+UseG1GC
-XX:+ExplicitGCInvokesConcurrent
-XX:+ExitOnOutOfMemoryError
-XX:+UseGCOverheadLimit
-XX:+HeapDumpOnOutOfMemoryError
-XX:-UseBiasedLocking
-Djdk.attach.allowAttachSelf=true
-Xms64G
-Xmx64G
-XX:G1HeapRegionSize=32M
-XX:ReservedCodeCacheSize=512M
-Djdk.nio.maxCachedBufferSize=2000000
```

### Resource Groups

**Configuration (etc/resource-groups.properties):**
```properties
resource-groups.configuration-manager=file
resource-groups.config-file=etc/resource-groups.json
```

**Resource Groups JSON:**
```json
{
  "rootGroups": [
    {
      "name": "ddl",
      "maxQueued": 100,
      "hardConcurrencyLimit": 10,
      "softMemoryLimit": "10%"
    },
    {
      "name": "ad-hoc",
      "maxQueued": 50,
      "hardConcurrencyLimit": 1,
      "softMemoryLimit": "100%"
    }
  ],
  "selectors": [
    {
      "queryType": "DATA_DEFINITION",
      "group": "ddl"
    },
    {
      "group": "ad-hoc"
    }
  ]
}
```

**Resource Group Properties:**

| Property | Description |
|----------|-------------|
| `name` | Group name |
| `maxQueued` | Max queued queries |
| `hardConcurrencyLimit` | Max concurrent queries |
| `softMemoryLimit` | Max distributed memory (absolute or %) |
| `softCpuLimit` | Soft CPU limit |
| `hardCpuLimit` | Hard CPU limit |
| `schedulingPolicy` | fair, query_priority, weighted_fair |
| `schedulingWeight` | Weight for weighted_fair |
| `jmxExport` | Export to JMX |

**Selector Rules:**

| Property | Description |
|----------|-------------|
| `user` | Username (regex) |
| `source` | Source (regex, e.g., trino-cli) |
| `queryType` | DATA_DEFINITION, DELETE, DESCRIBE, EXPLAIN, INSERT, SELECT |
| `clientTags` | Client tags |
| `group` | Target resource group |

**CLI with tags:**
```bash
trino --user mfuller --source mfuller-cli --client-tags adhoc-queries
```

---
