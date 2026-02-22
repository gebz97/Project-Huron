## Chapter 9: Governing Data Access

### Challenges in Data Lakes
- Massive number of data sets
- Many users
- Frictionless ingestion
- Exploratory work (catch-22)

### Traditional Access Control
```
Command: hdfs dfs -setfacl -m group:hr:r-- /salaries.csv
Problem: Must set for each of millions of files
```

### Tag-Based Access Control

**Quarantine Process:**
```
1. Ingest → Add "quarantine" tag → Restricted access
2. Review → Add sensitivity tags → Remove quarantine
3. Policy → Enforce based on tags
```

### Deidentifying Sensitive Data

**Three Levels:**

| Level | Method | Use Case |
|-------|--------|----------|
| Transparent | Auto encrypt/decrypt | Prevent disk access |
| Explicit | Encrypt each value | Block all access |
| Deidentification | Replace with fake data | Preserve analytics |

**Deidentification Example:**
```
Original: Guido Sarducci, 1212 Main St, Menlo Park
Deidentified: Marco Rossi, 1543 Oak Ave, Menlo Park
```

### Self-Service Access Management

**Process:**
1. **Publish** - Data owners publish to catalog
2. **Find** - Analysts search metadata
3. **Request** - Analysts request access
4. **Approve** - Data owners approve
5. **Provision** - Data provided (copy, view, etc.)

**Benefits:**
- No up-front permission work
- Data owners control usage
- Time-limited access
- Complete audit trail

### Data Sovereignty

**Track provenance:**
```
Data Set Properties:
- Originating Country: Germany
- Referenced Countries: Germany, France, UK
Policy: Cannot copy outside EU
```

---
