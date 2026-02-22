# 🌊 Project Huron

> A comprehensive monorepo for self-hosted data lake deployment, configuration, and management.

Project Huron is a one-stop reference and automation hub for engineers planning to design, deploy, and operate production-grade data lakes in self-hosted environments. It brings together Ansible playbooks, architecture guides, configuration references, and runbooks across every layer of the modern data lake stack — all in one place.

---

## 📐 Philosophy

Data lake infrastructure is sprawling by nature. Tooling decisions, version compatibility, networking topology, and operational concerns are scattered across dozens of official docs, blog posts, and tribal knowledge. Project Huron exists to consolidate that knowledge into a structured, opinionated, and actionable monorepo — one that you can fork, adapt, and build on top of for your own environment.

Everything here assumes a **self-hosted, on-premise or private-cloud** context. Managed cloud services (EMR, Glue, Databricks, etc.) are out of scope.

---

## 🗂️ Repository Structure

```
huron/
├── storage/                    # Storage layer — object stores, HDFS, and file formats
│   ├── minio/
│   ├── hdfs/
│   ├── ceph/
│   └── formats/                # Parquet, ORC, Iceberg, Delta Lake, Hudi guides
│
├── metastore/                  # Metadata and catalog layer
│   ├── hive-metastore/
│   ├── apache-atlas/
│   └── nessie/
│
├── compute/                    # Distributed compute and analytics engines
│   ├── spark/                  # Spark cluster setup, tuning, and job guides
│   ├── flink/                  # Stream processing with Apache Flink
│   └── mapreduce/
│
├── query/                      # Interactive query engines
│   ├── trino/                  # Trino (formerly PrestoSQL) deployment and config
│   ├── presto/                 # PrestoDB guides
│   └── hive/                   # Hive on Tez / LLAP
│
├── orchestration/              # Workflow orchestration and scheduling
│   ├── airflow/
│   ├── prefect/
│   └── dagster/
│
├── ingestion/                  # Data ingestion and CDC tooling
│   ├── kafka/
│   ├── nifi/
│   ├── debezium/
│   └── sqoop/
│
├── governance/                 # Data quality, lineage, and access control
│   ├── great-expectations/
│   ├── apache-ranger/
│   └── openmetadata/
│
├── monitoring/                 # Observability, metrics, and alerting
│   ├── prometheus/
│   ├── grafana/
│   └── opensearch/
│
├── ansible/                    # Ansible roles and playbooks for the full stack
│   ├── roles/
│   ├── inventories/
│   └── playbooks/
│
├── architecture/               # Reference architectures, diagrams, and ADRs
│   ├── diagrams/
│   ├── adr/                    # Architecture Decision Records
│   └── patterns/
│
└── docs/                       # General documentation and getting started guides
    ├── getting-started.md
    ├── glossary.md
    └── contributing.md
```

---

## 🧱 Domain Overview

### Storage Layer (`/storage`)
Guides and automation for the physical and logical storage layer, including object storage (MinIO, Ceph), distributed file systems (HDFS), and open table formats (Apache Iceberg, Delta Lake, Apache Hudi). Covers layout strategies, erasure coding, replication, and S3-compatible API configuration.

### Metadata & Catalog (`/metastore`)
Everything needed to stand up and operate a metadata layer. Covers Hive Metastore configuration with various backends, Apache Atlas for data governance and lineage, and Project Nessie for catalog versioning with Iceberg.

### Distributed Compute (`/compute`)
Focused primarily on Apache Spark — cluster deployment (standalone, YARN, Kubernetes), Spark configuration tuning, job submission patterns, and integration with object storage and table formats. Also covers Apache Flink for streaming workloads.

### Query Engines (`/query`)
Deployment and operational guides for interactive query engines, with emphasis on Trino and Presto. Covers connector configuration, resource group tuning, federation patterns, and query performance optimisation.

### Orchestration (`/orchestration`)
Runbooks and Ansible automation for deploying and managing workflow orchestration platforms including Apache Airflow, Prefect, and Dagster. Covers DAG design patterns, dependency management, and integration with the compute layer.

### Ingestion (`/ingestion`)
End-to-end ingestion pipeline guides covering batch and streaming patterns. Includes Kafka cluster setup, NiFi dataflow design, Debezium CDC pipelines, and bulk ingestion with Sqoop.

### Governance (`/governance`)
Data quality frameworks (Great Expectations), access control (Apache Ranger), and metadata management (OpenMetadata). Covers policy enforcement, column-level security, and audit logging.

### Monitoring (`/monitoring`)
Observability stack guides using Prometheus, Grafana, and OpenSearch. Includes pre-built dashboard references, alerting rule templates, and JMX exporter configuration for JVM-based services.

### Ansible (`/ansible`)
Reusable Ansible roles and playbooks for provisioning and configuring the full data lake stack. Designed to be composable — pick the roles relevant to your stack and wire them together with a top-level playbook.

### Architecture (`/architecture`)
Reference architectures for common data lake topologies, Architecture Decision Records (ADRs) for key design choices made within this project, and reusable design patterns (e.g. medallion architecture, lambda vs kappa).

---

## 🚀 Getting Started

1. **Clone the repository**
   ```bash
   git clone https://github.com/gebz97/Project-Huron.git
   cd Project-Huron
   ```

2. **Browse by domain** — navigate to the subdirectory most relevant to what you're deploying. Each domain directory contains its own `README.md` with a local overview and table of contents.

3. **Use the Ansible playbooks** — if you're deploying infrastructure, head to `/ansible` and review the `inventories/` structure and available roles before running any playbooks.

4. **Read the architecture guides first** — if you're greenfielding a data lake, start with `/architecture/patterns` and `/docs/getting-started.md` before diving into individual tooling guides.

---

## 🛠️ Prerequisites

Requirements vary by domain, but generally you'll want:

- Linux hosts (RHEL/Rocky/Alma/Oracle Linux)
- Ansible 2.14+ on your control node
- Python 3.9+ on managed nodes
- SSH access and sudo privileges on target hosts
- Familiarity with the JVM-based tooling ecosystem

Specific version requirements are documented within each domain directory.

---

## 🤝 Contributing

Contributions are welcome. If you're adding a new guide, Ansible role, or architecture pattern, please read [`docs/contributing.md`](docs/contributing.md) first. The short version:

- Follow the existing directory and naming conventions
- Include a `README.md` in every new subdirectory
- Ansible roles should include a working `molecule` test scenario where feasible
- Architecture docs should be written in plain Markdown with diagrams in Mermaid or as committed SVGs

---

## 📄 License

Creative Commons. See [`LICENSE`](LICENSE) for details.

---

> *Lake Huron — one of the five Great Lakes. Deep, vast, and connecting.*
