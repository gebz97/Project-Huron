
## Chapter 10: Security

### Security Layers

```
Client (CLI, JDBC, etc.)
    ↓ ← TLS/HTTPS
┌──────────────┐
│ Coordinator  │
└──────────────┘
    ↓ ← TLS/HTTPS
    Workers
    ↓ ← Kerberos/TLS
Data Sources
```

### Authentication Methods

| Method | Description | Configuration |
|--------|-------------|---------------|
| **Password/LDAP** | Username + password | `http-server.authentication.type=PASSWORD` |
| **Certificate** | Mutual TLS | `http-server.authentication.type=CERTIFICATE` |
| **Kerberos** | Kerberos authentication | `http-server.authentication.type=KERBEROS` |

### LDAP Authentication

**config.properties:**
```properties
http-server.authentication.type=PASSWORD
```

**password-authenticator.properties:**
```properties
password-authenticator.name=ldap
ldap.url=ldaps://ldap-server:636
ldap.user-bind-pattern=${USER}@example.com

# With group filtering
ldap.user-base-dn=OU=people,DC=example,DC=com
ldap.group-auth-pattern=(&(objectClass=inetOrgPerson)(uid=${USER})(memberof=CN=developers,OU=groups,DC=example,DC=com))
```

**CLI Connection:**
```bash
trino --user matt --password
```

### System Access Control

**allow-all (default):**
```properties
access-control.name=allow-all
```

**read-only:**
```properties
access-control.name=read-only
```

**file-based (rules.json):**
```json
{
  "catalogs": [
    {
      "user": "admin",
      "catalog": "system",
      "allow": true
    },
    {
      "catalog": "hive",
      "allow": true
    },
    {
      "user": "alice",
      "catalog": "postgresql",
      "allow": true
    },
    {
      "catalog": "system",
      "allow": false
    }
  ],
  "principals": [
    {
      "principal": "(.*)",
      "principal_to_user": "$1",
      "allow": "all"
    }
  ]
}
```

### Connector Access Control

**GRANT/REVOKE:**
```sql
GRANT SELECT ON hive.ontime.flights TO matt;
GRANT SELECT, DELETE ON hive.ontime.flights TO matt WITH GRANT OPTION;

REVOKE DELETE ON hive.ontime.flights FROM matt;
```

**Roles:**
```sql
CREATE ROLE admin;
GRANT SELECT, DELETE ON hive.ontime.flights TO admin;
GRANT admin TO USER matt, martin;

REVOKE admin FROM USER matt;
SET ROLE developer;
```

### Encryption

**HTTPS Configuration:**
```properties
http-server.https.enabled=true
http-server.https.port=8443
http-server.https.keystore.path=/etc/trino/keystore.jks
http-server.https.keystore.key=slickpassword
http-server.http.enabled=false  # After testing
```

**Create Keystore:**
```bash
# Self-signed certificate
keytool -genkeypair \
  -alias trino_server \
  -dname CN=*.example.com \
  -validity 10000 -keyalg RSA -keysize 2048 \
  -keystore keystore.jks \
  -keypass password \
  -storepass password

# Export certificate
keytool -exportcert \
  -alias trino_server \
  -file trino_server.cer \
  -keystore keystore.jks \
  -storepass password

# Create truststore
keytool -importcert \
  -alias trino_server \
  -file trino_server.cer \
  -keystore truststore.jks \
  -storepass password
```

**CLI with HTTPS:**
```bash
trino --server https://coordinator:8443 \
  --truststore-path ~/truststore.jks \
  --truststore-password password
```

**Internal Communication:**
```properties
internal-communication.https.required=true
internal-communication.https.keystore.path=/etc/trino/keystore.jks
internal-communication.https.keystore.key=slickpassword
```

### Certificate Authentication

**Server config.properties:**
```properties
http-server.authentication.type=CERTIFICATE
http-server.https.truststore.path=/etc/trino/truststore.jks
http-server.https.truststore.key=slickpassword
node.internal-address-source=FQDN
```

**Client keystore:**
```bash
keytool -genkeypair \
  -alias matt \
  -dname CN=matt \
  -validity 10000 -keyalg RSA -keysize 2048 \
  -keystore client-keystore.jks \
  -keypass password \
  -storepass password
```

**CLI with client certificate:**
```bash
trino --server https://coordinator:8443 \
  --truststore-path ~/truststore.jks \
  --truststore-password password \
  --keystore-path ~/client-keystore.jks \
  --keystore-password password \
  --user matt
```

**Access control mapping (rules.json):**
```json
{
  "principals": [
    {
      "principal": "CN=(.*)",
      "principal_to_user": "$1",
      "allow": true
    }
  ]
}
```

### Kerberos Authentication

**Server config.properties:**
```properties
http-server.authentication.type=KERBEROS
http.server.authentication.krb5.service-name=trino
http.server.authentication.krb5.keytab=/etc/trino/trino.keytab
```

**Internal Kerberos:**
```properties
internal-communication.kerberos.enabled=true
```

### Hive Connector Kerberos

**Metastore Authentication:**
```properties
hive.metastore.authentication.type=KERBEROS
hive.metastore.service.principal=hive/_HOST@EXAMPLE.COM
hive.metastore.client.principal=trino@EXAMPLE.COM
hive.metastore.client.keytab=/etc/trino/hive.keytab
```

**HDFS Authentication:**
```properties
hive.hdfs.authentication.type=KERBEROS
hive.hdfs.trino.principal=hdfs@EXAMPLE.COM
hive.hdfs.trino.keytab=/etc/trino/hdfs.keytab
```

---
