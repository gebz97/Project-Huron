
## Chapter 8: Cataloging the Data Lake

### Why Catalogs?
- Massive number of data sets
- Cryptic field names
- No search capability
- Can't understand content without opening

### Types of Metadata

#### Technical Metadata
- Field names, data types
- Profiling statistics
- Cardinality, selectivity, density
- Formats

#### Business Metadata
- Business terms
- Descriptions
- Tags
- Ratings

### Tagging Approaches

**Manual:**
- SMEs document knowledge
- Crowdsource from analysts
- Only popular data gets tagged

**Automated:**
- Fingerprint fields
- Learn from manual tags
- Suggest tags for similar fields

### Tag-Based Management

**Access Control:**
```
Tag: PII (Personally Identifiable Information)
Policy: Restrict access to authorized users
Result: Any file with PII tag gets protected
```

**Data Quality:**
```
Tag: Age
Rule: Must be number between 0-125
Quality Score: % of rows that comply
```

**Sensitive Data:**
```
Tag: CreditCard
Action: Automatically encrypt or mask
```

### Catalog Tools Comparison

| Tool | Big Data Support | Tagging | Business UI |
|------|-----------------|---------|-------------|
| Waterline Data | Native | Automated | Yes |
| Informatica EDC | Native | Manual | |
| Cloudera Navigator | Native | Manual | |
| Apache Atlas | Native | Manual | |
| Alation | Hive only | Manual | Yes |

---
