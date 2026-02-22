## Chapter 2: Historical Perspective

### Evolution of Data Management

#### 1. Spreadsheets
First self-service tool, but doesn't scale

#### 2. Relational Databases (RDBMS)
- **Schema on write** - Define structure before loading
- **Normalization** - Break data into smallest chunks
- **Primary/Foreign Keys** - Relationships between tables

**Normalization Example:**

```
❌ Bad Design (redundant):
Customer: Mary, F, Married, 94301, Order1, $987.19
Customer: Mary, F, Married, 94301, Order2, $12.20

✅ Good Design (normalized):
Customers: ID112211, Mary, F, Married, 94301
Orders: ID112211, Order1, $987.19
Orders: ID112211, Order2, $12.20
```

#### 3. Data Warehousing
- Separate analytics from operations
- **Star Schemas** - Fact tables + Dimension tables
- **Slowly Changing Dimensions** - Track history

#### 4. Data Warehouse Ecosystem

```
Source Systems → ETL → Data Warehouse → Data Marts → BI Tools
                      ↓                           ↓
                Metadata Repository          Reporting
```

**Key Technologies:**
- ETL/ELT tools
- Data quality tools
- Data modeling tools
- Business glossaries
- MDM systems

---
