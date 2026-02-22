
## Chapter 12: Lakehouse Governance and Security

### Lakehouse Governance Components

```
┌─────────────────────────────────────────────┐
│            Identity & Access Management     │ (1)
├─────────────────────────────────────────────┤
│            Data Catalogs & Metastores       │ (2)
├─────────────────────────────────────────────┤
│            Elastic Data Management          │ (3) Filesystem
├─────────────────────────────────────────────┤
│            Audit Logging                     │ (4)
├─────────────────────────────────────────────┤
│            Monitoring                         │ (5)
├─────────────────────────────────────────────┤
│            Data Sharing                       │ (6)
├─────────────────────────────────────────────┤
│            Data Lineage                       │ (7)
├─────────────────────────────────────────────┤
│            Data Discovery                     │ (8)
└─────────────────────────────────────────────┘
```

### Data Life Cycle

1. Creation
2. Storage
3. Usage
4. Sharing
5. Archiving
6. Destruction

### Data Asset Model

```
Data Asset (TABLE, VIEW)
    ↓
Securable Object
    ↓
Permissions (GRANT/REVOKE)
    ↓
Principal (USER, GROUP)
```

**SQL Grants:**
```sql
-- Data Control Language (DCL)
GRANT SELECT ON table TO user;
REVOKE SELECT ON table FROM user;

-- Data Definition Language (DDL)
CREATE TABLE ...;
ALTER TABLE ... SET TBLPROPERTIES;
DROP TABLE ...;

-- Data Manipulation Language (DML)
SELECT ...;
INSERT ...;
UPDATE ...;
DELETE ...;
```

### Filesystem Permissions

```bash
$ ls -lah /lakehouse/bronze/
drwxr-x--x@ 338 dataeng eng_analysts 11K _delta_log
```

| Field | Meaning |
|-------|---------|
| `d` | Directory type |
| `rwx` | Owner (dataeng) - read/write/execute |
| `r-x` | Group (eng_analysts) - read/execute |
| `--x` | Others - execute only |

### Identity and Access Management (IAM)

**Identity:** User (human) or Service Principal (headless)
**Authentication:** Prove identity (tokens, certificates)
**Authorization:** What actions allowed (policies)

### Role-Based Access Control (RBAC)

**Common Personas:**

| Role | Access Pattern | Name |
|------|---------------|------|
| Engineering | All (read, write, admin) | `role/developerRole` |
| Analyst | Primarily read-only | `role/analystRole` |
| Scientist | Read-only + create tables | `role/scientistRole` |
| Business | Mostly read-only | `role/businessRole` |

### Data Classification (Stop-Light Pattern)

| Level | Description | Example |
|-------|-------------|---------|
| **Green** | General access | Public datasets (USGS earthquake data) |
| **Yellow** | Restricted, need-to-know | Internal pricing, margins |
| **Red** | Highly sensitive | PII, payroll, credit cards, HIPAA |

### Lakehouse Namespace Pattern

```
s3://com.common_foods.[dev|prod]/
└── common_foods/
    ├── consumer/
    │   ├── _apps/
    │   │   └── clickstream/
    │   │       ├── app.yaml
    │   │       ├── v1.0.0/
    │   │       │   ├── _checkpoints/
    │   │       │   ├── config/
    │   │       │   └── sources/
    │   │       └── clickstream_app_v1.0.0.whl
    │   └── clickstream/ (Delta table)
    │       ├── _delta_log
    │       └── event_date=2024-02-17/
    └── {other tables}
```

### S3 Access Grants

**Create bucket:**
```bash
aws s3api create-bucket \
  --bucket com.dldgv2.production.v1 \
  --region us-west-1 \
  --create-bucket-configuration LocationConstraint=us-west-1
```

**Create Access Grants instance:**
```bash
aws s3control create-access-grants-instance \
  --account-id $ACCOUNT_ID
```

**Trust policy (trust-policy.json):**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "access-grants.s3.amazonaws.com"},
    "Action": ["sts:AssumeRole", "sts:SetSourceIdentity", "sts:SetContext"]
  }]
}
```

**Create IAM role:**
```bash
aws iam create-role --role-name s3ag-location-role \
  --assume-role-policy-document file://trust-policy.json
```

**Grant READ:**
```bash
aws s3control create-access-grant \
  --account-id $ACCOUNT_ID \
  --access-grants-location-id default \
  --access-grants-location-configuration S3SubPrefix="warehouse/gold/analysis/*" \
  --permission READ \
  --grantee GranteeType=IAM,GranteeIdentifier=arn:aws:iam::$ACCOUNT_ID:role/analystRole
```

**Grant READWRITE:**
```bash
aws s3control create-access-grant \
  --account-id $ACCOUNT_ID \
  --access-grants-location-id default \
  --access-grants-location-configuration S3SubPrefix="warehouse/gold/analysis/*" \
  --permission READWRITE \
  --grantee GranteeType=IAM,GranteeIdentifier=arn:aws:iam::$ACCOUNT_ID:role/developerRole
```

### Fine-Grained Access with Dynamic Views

```sql
CREATE VIEW consumer.prod.orders_redacted AS
SELECT
  order_id,
  region,
  items,
  amount,
  CASE
    WHEN has_tag_value('pii') AND is_account_group_member('consumer_privileged')
    THEN user
    ELSE named_struct(
      'user_id', sha1(user.user_id),
      'email', null,
      'age', null
    )
  END AS user
FROM consumer.prod.orders;
```

---
