
## Chapter 5: Production-Ready Deployment

### Configuration Details

**Server Configuration (config.properties):**

| Property | Description | Example |
|----------|-------------|---------|
| `coordinator` | Is this node a coordinator? | true/false |
| `node-scheduler.include-coordinator` | Schedule work on coordinator? | false for large clusters |
| `http-server.http.port` | HTTP port | 8080 |
| `http-server.https.port` | HTTPS port | 8443 |
| `query.max-memory` | Total distributed memory per query | 5GB |
| `query.max-memory-per-node` | User memory per node per query | 1GB |
| `query.max-total-memory-per-node` | User + system memory per node | 2GB |
| `discovery-server.enabled` | Enable embedded discovery | true (coordinator only) |
| `discovery.uri` | Discovery service URI | http://localhost:8080 |

**Logging (log.properties):**
```properties
io.trino=INFO
io.trino.plugin.postgresql=DEBUG
```

**Log Files (var/log/):**
- `launcher.log` - Launcher stdout/stderr
- `server.log` - Main Trino logs
- `http-request.log` - All HTTP requests

**Node Configuration (node.properties):**
```properties
node.environment=production
node.id=ffffffff-ffff-ffff-ffff-ffffffffffff
node.data-dir=/var/trino/data
```

### Cluster Installation

**Coordinator config.properties:**
```properties
coordinator=true
node-scheduler.include-coordinator=false
http-server.http.port=8080
query.max-memory=5GB
query.max-memory-per-node=1GB
query.max-total-memory-per-node=2GB
discovery-server.enabled=true
discovery.uri=http://<coordinator-ip>:8080
```

**Worker config.properties:**
```properties
coordinator=false
http-server.http.port=8080
query.max-memory=5GB
query.max-memory-per-node=1GB
query.max-total-memory-per-node=2GB
discovery.uri=http://<coordinator-ip>:8080
```

**Start Cluster:**
```bash
# Start coordinator first, then workers
bin/launcher start

# Verify cluster
trino> SELECT * FROM system.runtime.nodes;
```

### RPM Installation

```bash
# Download RPM
wget https://repo.maven.apache.org/maven2/io/trino/trino-server-rpm/354/trino-server-rpm-354.rpm

# Install
sudo rpm -i trino-server-rpm-*.rpm

# Manage service
service trino [start|stop|restart|status]

# Uninstall
sudo rpm -e trino
```

**RPM Directory Structure:**
- `/usr/lib/trino/` - Binaries and plugins
- `/etc/trino/` - Configuration
- `/var/log/trino/` - Logs
- `/var/lib/trino/data/` - Data directory

### Cluster Sizing Considerations

**Factors affecting cluster size:**
- CPU and memory per node
- Network performance
- Number and type of data sources
- Query complexity and concurrency
- Storage read/write performance
- User count and usage patterns

---
