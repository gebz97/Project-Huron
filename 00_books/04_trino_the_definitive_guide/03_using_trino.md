
## Chapter 3: Using Trino

### Trino Command-Line Interface (CLI)

**Download CLI:**
```bash
# Download executable JAR
wget -O trino https://repo.maven.apache.org/maven2/io/trino/trino-cli/354/trino-cli-354-executable.jar

# Make executable
chmod +x trino

# Add to PATH (optional)
mv trino ~/bin
export PATH=~/bin:$PATH

# Check version
trino --version
```

**Connect to Trino:**
```bash
# Connect to localhost:8080 (default)
trino

# Connect to specific server
trino --server https://trino.example.com:8080

# Connect with catalog and schema
trino --catalog tpch --schema sf1
```

**CLI Commands:**
```sql
trino> help
trino> SHOW CATALOGS;
trino> SHOW SCHEMAS FROM tpch;
trino> SHOW TABLES FROM tpch.sf1;
trino> USE tpch.sf1;
trino> SELECT count(*) FROM nation;
trino> quit
```

**Execute Query and Exit:**
```bash
# Single query
trino --execute 'SELECT nationkey, name FROM tpch.sf1.nation LIMIT 5'

# With catalog and schema
trino --catalog tpch --schema sf1 --execute 'SELECT name FROM nation'

# Execute from file
trino -f nations.sql

# Output formats
trino --output-format=VERTICAL --execute 'SELECT * FROM nation'
```

**CLI Options:**
- `--debug` - Enable debug information
- `--ignore-errors` - Continue on errors
- `TRINO_PAGER` - Set pager (default: less)
- `TRINO_HISTORY_FILE` - Set history file (default: ~/trino_history)

### Trino JDBC Driver

**Download JDBC Driver:**
```bash
wget https://repo.maven.apache.org/maven2/io/trino/trino-jdbc/354/trino-jdbc-354.jar
```

**JDBC Connection Properties:**
```properties
# Required
URL: jdbc:trino://host:port/catalog/schema
User: username

# Optional
Password: secret
SSL: true/false
SSLTrustStorePath: /path/to/truststore
SSLTrustStorePassword: truststore_password
applicationNamePrefix: my-app
```

**DBeaver Setup:**
1. File → New → Database Connection
2. Search "trino" or "prestosql"
3. Configure connection
4. Username required (even without authentication)

**SQuirreL SQL Client Setup:**
1. Add JAR to classpath (lib folder)
2. Register driver:
   - Class name: `io.trino.jdbc.TrinoDriver`
   - Example URL: `jdbc:trino://localhost:8080`
3. Create alias with connection details

### Trino Web UI

Access at: `http://localhost:8080`

**Features:**
- Main dashboard with cluster metrics
- Query list with status, duration, user
- Query details with execution plan
- Stage and task-level metrics

---
