
## Chapter 2: The Architecture of Apache Iceberg

### Three Layers of Iceberg

```
┌─────────────────────────────────────┐
│          Catalog Layer              │ ← Maps table → current metadata
├─────────────────────────────────────┤
│         Metadata Layer              │ ← Metadata files, manifest lists, manifests
├─────────────────────────────────────┤
│           Data Layer                │ ← Datafiles + Delete files
└─────────────────────────────────────┘
```

### Data Layer

**Datafiles:**
- File format agnostic (Parquet, ORC, Avro)
- Parquet is most common (columnar, stats per row group)
- Immutable (never modified in place)

**Parquet File Structure:**
```
Parquet File
├── Row Group 0
│   ├── Column A (all rows' values for col A)
│   │   ├── Page 0 (subset of values)
│   │   ├── Page 1
│   │   └── ...
│   ├── Column B
│   └── ...
├── Row Group 1
└── ...
└── Footer (schema, stats, offsets)
```

**Delete Files (Iceberg v2):**
- Track deleted records without rewriting files
- Enable merge-on-read (MOR)

**Two Types of Delete Files:**

| Type | How It Works | Pros | Cons |
|------|--------------|------|------|
| **Positional Delete** | Specifies file + row position to delete | Fast reads | Write must know positions |
| **Equality Delete** | Specifies column values to delete | Fast writes | Slower reads (must check all records) |

**Sequence Numbers:** Ensure deletes don't apply to newer inserts with same values

### Metadata Layer

**Manifest Files:**
- List of datafiles + statistics per file
- One manifest = subset of files (data OR deletes, not both)
- Written per write operation
- Avro format

**Statistics in Manifests:**
- File path
- Record count
- Min/max values per column
- Null value counts

**Manifest Lists:**
- Snapshot of table at a point in time
- List of manifest files for that snapshot
- Statistics about manifests (partition bounds, file counts)

**Manifest List Schema:**
| Field | Type | Description |
|-------|------|-------------|
| `manifest_path` | string | Location of manifest file |
| `manifest_length` | long | Size in bytes |
| `partition_spec_id` | int | Partition spec used |
| `content` | int | 0=data, 1=deletes |
| `sequence_number` | long | Order of addition |
| `added_rows_count` | long | Total rows added |
| `existing_rows_count` | long | Total rows existing |
| `deleted_rows_count` | long | Total rows deleted |

**Metadata Files:**
- Table schema, partition spec, snapshots
- Created per transaction
- Current metadata file tracked by catalog

**Metadata File Schema:**
| Field | Description |
|-------|-------------|
| `format-version` | 1 or 2 |
| `table-uuid` | Unique table identifier |
| `location` | Table base location |
| `last-sequence-number` | Highest sequence number |
| `last-updated-ms` | Last update timestamp |
| `schemas` | List of all schemas (by ID) |
| `current-schema-id` | Current schema ID |
| `partition-specs` | List of partition specs |
| `snapshots` | List of snapshots |
| `refs` | Branches and tags |

### Puffin Files

- Store advanced statistics and indexes
- Currently supports Theta sketches (approximate distinct counts)
- Structure: blobs + metadata

### Catalog

**Role:** Store current metadata pointer atomically

**How Different Catalogs Store Pointer:**

| Catalog | Storage Method |
|---------|----------------|
| Hadoop | `version-hint.txt` file in metadata folder |
| Hive Metastore | `location` table property |
| AWS Glue | `metadata_location` table property |
| Nessie | `metadataLocation` table property |

---
