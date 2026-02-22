
## Chapter 12: Governance and Security

### Three Levels of Security

1. **Datafiles** (storage layer)
2. **Semantic Layer** (abstraction layer)
3. **Catalog** (metadata layer)

### Security Best Practices

- **Least privilege access** - Minimum permissions needed
- **Encryption at rest and in transit**
- **Strong authentication** - MFA, strong passwords
- **Audit trails and logging**
- **Data retention and disposal policies**
- **Continuous monitoring**

### HDFS Security

**ACLs:**
```bash
hdfs dfs -setfacl -m user:username:rwx /path/to/file
hdfs dfs -setfacl -m group:groupname:r-x /path/to/file
```

**Encryption:**
```bash
hdfs crypto -createZone -keyName myEncryptionKey -path /path/to/encryption/zone
```

**Permissions:**
```bash
hdfs dfs -chown newowner:newgroup /path/to/file
hdfs dfs -chmod 700 /path/to/file
```

### Amazon S3 Security

**SSE-S3 (S3-managed keys):**
```bash
aws s3api put-bucket-encryption --bucket YOUR_BUCKET \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "AES256"}
    }]
  }'
```

**SSE-KMS (AWS KMS):**
```bash
aws s3api put-bucket-encryption --bucket YOUR_BUCKET \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms", 
        "KMSMasterKeyID": "YOUR_KMS_KEY_ID"
      }
    }]
  }'
```

**Bucket Policy (IP restriction):**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowSpecificIP",
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::YOUR_BUCKET/*",
    "Condition": {
      "IpAddress": {"aws:SourceIp": "YOUR_IP_ADDRESS"}
    }
  }]
}
```

**IAM Policy:**
```bash
aws iam put-user-policy --user-name YOUR_USER --policy-name S3Access \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::YOUR_BUCKET/*"
    }]
  }'
```

**Object ACL:**
```bash
aws s3api put-object-acl --bucket your-bucket --key path/to/object \
  --acl private --grant-read id="YOUR_IAM_USER_ID"
```

### Azure Data Lake Storage (ADLS) Security

**Create Key Vault:**
```bash
az keyvault create --name YourKeyVault --resource-group YourGroup --location YourLocation
```

**Create Key:**
```bash
az keyvault key create --vault-name YourKeyVault --name YourKey --kty RSA
```

**Link ADLS to Key Vault:**
```bash
az storage account update --name YourStorageAccount --resource-group YourGroup \
  --assign-identity [system] --keyvault YourKeyVault
```

**Set Key Vault Policy:**
```bash
az keyvault set-policy --name YourKeyVault --resource-group YourGroup \
  --spn YourServicePrincipal --key-permissions get list
```

**RBAC Role Assignment:**
```bash
az role assignment create --assignee user@example.com \
  --role "Storage Blob Data Contributor" \
  --scope /subscriptions/YourSub/resourceGroups/YourGroup/providers/Microsoft.DataLakeStore/accounts/YourADLSAccount
```

**Set ACLs:**
```bash
az dls fs access set --account YourADLSAccount --path /your/directory \
  --acl-spec "user::rwx,group::r--,other::---"
```

### Google Cloud Storage Security

**Create KMS Key:**
```bash
gcloud kms keyrings create your-keyring --location global
gcloud kms keys create your-key --location global --keyring your-keyring --purpose encryption
```

**Assign CMEK to bucket:**
```bash
gsutil kms authorize -k projects/your-project/locations/global/keyRings/your-keyring/cryptoKeys/your-key
gsutil defacl set private gs://your-bucket
```

**IAM Service Account:**
```bash
gcloud iam service-accounts create your-service-account
gcloud projects add-iam-policy-binding your-project \
  --member serviceAccount:your-service-account@your-project.iam.gserviceaccount.com \
  --role roles/storage.objectAdmin
```

**Bucket Policy:**
```json
{
  "bindings": [{
    "role": "roles/storage.objectViewer",
    "members": ["user:email@example.com"]
  }]
}
```

**Object ACL:**
```bash
gsutil acl ch -u user:email@example.com:READ gs://your-bucket/your-object
```

### Dremio Semantic Layer Security

**Role-Based Access Control:**
```sql
CREATE ROLE analyst;
CREATE ROLE manager;

GRANT SELECT ON VDS_sales_data TO analyst;
GRANT SELECT, UPDATE ON VDS_employee_data TO manager;

GRANT analyst TO user1;
GRANT manager TO user2;
```

**Column-Based Access Control:**
```sql
CREATE FUNCTION mask_salary(salary VARCHAR)
RETURNS VARCHAR
RETURN SELECT CASE 
  WHEN query_user()='user@example.com' OR is_member('HR') THEN salary
  ELSE 'XXX-XX'
END;

CREATE TABLE employee_salaries (
  id INT,
  salary VARCHAR MASKING POLICY mask_salary(salary),
  department VARCHAR
);
```

**Row-Based Access Control:**
```sql
CREATE FUNCTION restrict_region(region VARCHAR)
RETURNS BOOLEAN
RETURN SELECT CASE 
  WHEN query_user()='user@example.com' OR is_member('HR') THEN true
  WHEN region = 'North' THEN true
  ELSE false
END;

CREATE TABLE regional_employee_data (
  id INT,
  role VARCHAR,
  department VARCHAR,
  salary FLOAT,
  region VARCHAR,
  ROW ACCESS POLICY restrict_region(region)
);
```

### Trino Security

**Create Role:**
```sql
CREATE ROLE analyst;
CREATE ROLE data_scientist;
```

**Grant Role to User:**
```sql
GRANT analyst TO USER alice;
GRANT data_scientist TO USER bob;
```

**Grant Privileges:**
```sql
GRANT SELECT ON orders TO analyst;
GRANT INSERT, SELECT ON customer_data TO data_scientist;
```

**Revoke:**
```sql
REVOKE SELECT ON orders FROM analyst;
REVOKE data_scientist FROM USER bob;
```

**Create Role with Admin:**
```sql
CREATE ROLE admin WITH ADMIN USER admin_user;
GRANT admin TO USER alice GRANTED BY admin_user;
```

### Nessie Security

**Branch Listing Permission:**
```properties
nessie.server.authorization.rules.allow_branch_listing=op=='VIEW_REFERENCE' && role in ['analyst', 'viewer']
```

**Branch Creation Permission:**
```properties
nessie.server.authorization.rules.allow_branch_creation=op=='CREATE_REFERENCE' &&
role=='data_admin' && ref.matches('.*prod.*')
```

**Branch Deletion Permission:**
```properties
nessie.server.authorization.rules.allow_branch_deletion=op=='DELETE_REFERENCE' &&
role in ['data_admin', 'super_admin']
```

**Read Entity Permission:**
```properties
nessie.server.authorization.rules.allow_reading_entity_value=op=='READ_ENTITY_VALUE' &&
role in ['analyst', 'viewer'] && path.startsWith('data/')
```

### AWS Lake Formation Tag-Based Access Control

**Assign Tags:**
```bash
aws glue update-table --database-name mydb --table-input '{
  "Name": "mytable",
  "Parameters": {
    "TAGS": {
      "sensitivity": "Confidential",
      "project": "ProjectA"
    }
  }
}'
```

**Create Policy:**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "lakeformation:GetDataAccess",
    "Resource": "*",
    "Condition": {
      "StringEquals": {"glue:ResourceTag/project": "ProjectA"}
    },
    "Principal": {
      "AWS": "arn:aws:iam::123456789012:group/ProjectA-Team"
    }
  }]
}
```

---
