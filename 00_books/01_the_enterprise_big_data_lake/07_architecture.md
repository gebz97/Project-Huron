
## Chapter 7: Architecting the Data Lake

### Zone Architecture

```
External Sources
       ↓
┌──────────────┐
│ Landing Zone │ → Raw data, as-is
└──────────────┘
       ↓
┌──────────────┐
│  Gold Zone   │ → Cleansed, curated
└──────────────┘
    ↙        ↘
┌──────┐  ┌────────┐
│ Work │  │Sensitive│
│ Zone │  │  Zone  │
└──────┘  └────────┘
```

### Zone Details

| Zone | Contents | Users | Governance |
|------|----------|-------|------------|
| **Landing** | Raw ingested data | Technical only | Minimal |
| **Gold** | Cleansed, production-ready | All analysts | Strong |
| **Work** | Projects, experiments | Data scientists | Minimal |
| **Sensitive** | Encrypted data | Authorized only | Strict |

### Multiple Data Lakes

**Keep Separate When:**
- Regulatory constraints
- Organizational barriers
- Need predictable performance

**Merge When:**
- Better resource utilization
- Lower admin costs
- Reduce redundancy
- Enable enterprise projects

### On-Premises vs. Cloud

| Aspect | On-Premises | Cloud |
|--------|-------------|-------|
| Storage | Fixed by nodes | Virtually unlimited |
| Compute | Fixed capacity | Elastic (scale on demand) |
| Cost | Capital expense | Operating expense |
| Management | Your team | Provider |
| Example | Hadoop cluster | AWS S3 + EC2 + EMR |

---
