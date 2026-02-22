
## Chapter 2: Installing and Configuring Trino

### Trying Trino with Docker

```bash
# Run Trino container
docker run -d --name trino-trial trinodb/trino

# Connect to Trino CLI
docker exec -it trino-trial trino

# Run a query
trino> SELECT count(*) FROM tpch.sf1.nation;
 _col0
-------
    25

# Stop and remove container
docker stop trino-trial
docker rm trino-trial
docker rmi trinodb/trino
```

### Installing from Archive File

**Prerequisites:**
- Java 11 (11.0.7+)
- Python 2.6+

```bash
# Check Java version
java --version

# Check Python version
python --version

# Download Trino server
wget https://repo.maven.apache.org/maven2/io/trino/trino-server/354/trino-server-354.tar.gz

# Extract
tar xvzf trino-server-354.tar.gz
cd trino-server-354
```

**Directory Structure:**
```
trino-server-354/
├── bin/          # Launch scripts
├── lib/          # Java archives
├── plugins/      # Connectors and plugins
├── etc/          # Configuration (created by user)
└── var/          # Logs and data (created at runtime)
```

### Configuration Files

**etc/config.properties (coordinator):**
```properties
coordinator=true
node-scheduler.include-coordinator=true
http-server.http.port=8080
query.max-memory=5GB
query.max-memory-per-node=1GB
query.max-total-memory-per-node=2GB
discovery-server.enabled=true
discovery.uri=http://localhost:8080
```

**etc/node.properties:**
```properties
node.environment=demo
node.data-dir=/var/trino/data
```

**etc/jvm.config:**
```
-server
-Xmx4G
-XX:-UseBiasedLocking
-XX:+UseG1GC
-XX:G1HeapRegionSize=32M
-XX:+ExplicitGCInvokesConcurrent
-XX:+HeapDumpOnOutOfMemoryError
-XX:+ExitOnOutOfMemoryError
-XX:ReservedCodeCacheSize=512M
-XX:PerMethodRecompilationCutoff=10000
-XX:PerBytecodeRecompilationCutoff=10000
-Djdk.nio.maxCachedBufferSize=2000000
-Djdk.attach.allowAttachSelf=true
```

### Adding a Data Source (TPC-H Connector)

**etc/catalog/tpch.properties:**
```properties
connector.name=tpch
```

### Running Trino

```bash
# Start in foreground
bin/launcher run

# Expected output after successful start
INFO main io.trino.server.Server ======== SERVER STARTED

# Stop with Ctrl-C
```

---
