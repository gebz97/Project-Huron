
## Chapter 5: From Data Ponds to Data Lakes

### Data Warehouse Limitations
- Expensive
- Slow to change
- Aggregates historical data
- Limited variety

### Moving to a Data Pond

**Partitioning in Hive:**
```
/transactions/
  Year=2016/
    Month=3/
      Day=2/
        file1.json
        file2.json
```

### Preserving History

**Three Approaches:**

1. **Slowly Changing Dimensions** (Traditional)
   - Track only critical attributes
   - Complex ETL

2. **Snapshots** (Data Pond)
   - Store complete data daily
   - Simple ingestion
   - More storage, simpler analytics

3. **Denormalization** (Data Pond)
   - Add attributes to transaction files
   - Avoid joins
   - Data duplication

### New Data Types in Data Lakes

| Type | Traditional Approach | Data Lake Approach |
|------|---------------------|-------------------|
| Raw data | Throw away | Keep all |
| External data | Multiple purchases | Central repository |
| IoT/streaming | Aggregate | Store raw |
| Social media | Ignore | Store native format |

### Lambda Architecture

```
Real-time Stream
       ↓
┌──────────────┐
│  Speed Layer │ → Real-time views
└──────────────┘
       ↓
┌──────────────┐
│  Batch Layer │ → Batch views
└──────────────┘
       ↓
    Query
```

---
