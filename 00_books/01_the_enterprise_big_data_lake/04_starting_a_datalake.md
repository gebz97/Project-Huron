## Chapter 4: Starting a Data Lake

### Why Hadoop?

| Feature | Benefit |
|---------|---------|
| Extreme scalability | Add nodes as needed |
| Cost-effective | Commodity hardware, open source |
| Modularity | Same file accessible by multiple engines |
| Schema on read | Frictionless ingestion |

### Three Popular Approaches

#### 1. Offload Existing Functionality
- Move ETL processing to Hadoop
- Process non-tabular data
- Real-time/streaming processing
- Scale existing projects

#### 2. New Projects
- Data science initiatives
- IoT/machine data processing
- Social media analytics

#### 3. Central Point of Governance
- Consistent security policies
- Uniform data quality
- Centralized compliance

### Decision Tree for Data Lake Strategy

```
Are there data puddles?
├─ Yes → Can they move to central cluster?
│        ├─ Yes → Justify with project cost
│        └─ No → Justify to prevent proliferation
└─ No → Are groups asking for big data?
         ├─ Yes → Use data science/analytics route
         └─ No → Is there a governance initiative?
                  ├─ Yes → Single point of governance
                  └─ No → Find workloads to offload
```

---
