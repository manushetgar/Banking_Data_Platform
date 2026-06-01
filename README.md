# Banking Data Platform

An end-to-end data platform designed to process high-volume banking transactions and events. The platform orchestrates PySpark processing on AWS EKS, structures data in a multi-tier Medallion architecture (S3 Bronze to Silver), and builds historical Data Vault 2.0 structures inside Snowflake via dbt Core, exposing clean Information Marts (Gold) for BI, AML, and compliance workflows.

---

## Architecture Overview

The system ingests core banking events, conforms schemas, models historical states, and feeds downstream applications through a highly organized pipeline.

```text
  [ Data Sources ]       [ Ingestion / Storage Layer ]        [ Modeling & Analytics ]
 
  +--------------+       +---------------------------+       +-------------------------+
  | Ingestion    | ----> | AWS S3 (Raw Bronze Bucket) | ----> | PySpark Processing      |
  | Sources      |       | (Raw CSVs and events)     |       | (AWS EKS Compute Nodes) |
  +--------------+       +---------------------------+       +-------------------------+
                                                                          |
                                                                          v
  +--------------+       +---------------------------+       +-------------------------+
  | Data Marts   | <---- | Snowflake (Gold Vault)    | <---- | AWS S3 (Silver Bucket)  |
  | (Gold Layer) |       | (Clean Hubs/Links/Sats)   |       | (Structured Parquet)    |
  +--------------+       +---------------------------+       +-------------------------+
         |                             ^
         |                             |
         v                             +--- Orchestrated via Apache Airflow DAGs
  +--------------+                     +--- Automated transformations via dbt
  | Downstream   |
  | BI & Reports |
  +--------------+
```

### Infrastructure & Technology Stack

- **Storage Layer**: AWS S3 structured as a Medallion Lakehouse (Bronze, Silver, Gold).
- **Processing Layer**: Apache Spark running distributed jobs inside AWS Elastic Kubernetes Service (EKS).
- **Modeling & Transformation**: dbt Core defining staging structures and building historical Data Vault 2.0 tables inside Snowflake.
- **Orchestration**: Apache Airflow dynamically executing jobs and transformation DAGs.
- **Infrastructure as Code**: Terraform configurations automating reproducible AWS and Snowflake environments.
- **Containerization**: Docker and Kubernetes (K8s) managing application runtimes and pipelines.

---

## Repository Map

```text
manushetgar-banking-data-platform/
├── .github/workflows/          # CI/CD pipelines (GitHub Actions)
├── requirements.txt            # Python dependencies
├── profiles.yml                # dbt connection profiles
├── docs/                       # Architectural and setup guides
│   ├── SETUP.md                # Deployment instructions
│   └── images.md               # Guide to documentation assets
│
├── terraform/                  # Infrastructure as Code
│   ├── variables.tf            # Global variables
│   ├── modules/                # Reusable infrastructure blocks
│   │   ├── aws/                # S3, EKS, and IAM configuration
│   │   └── snowflake/          # Databases, warehouses, and RBAC roles
│   └── env/                    # Environment instances
│       ├── dev/                # Sandbox environment (Spot instances)
│       └── prod/               # Production environment (On-demand instances)
│
├── k8s/                        # Kubernetes workloads and configuration
│   ├── namespace.yaml
│   ├── airflow/                # Airflow deployments, services, and maps
│   ├── spark/                  # Spark job definitions
│   └── dbt/                    # Container runner definitions
│
├── dags/                       # Airflow orchestrators
│   ├── ingestion_dag.py        # Bronze and Silver pipeline trigger
│   ├── dbt_dag.py              # Data Vault and Mart transformation runner
│   └── utils/                  # Ingestion utility scripts
│
├── dbt/                        # dbt project structures
│   ├── dbt_project.yml         # Main project properties
│   ├── packages.yml            # Dependencies (e.g., automate-dv)
│   ├── models/                 # Stage, Vault (Hubs, Links, Sats), and Marts layers
│   └── tests/                  # Integrity and validation tests
│
├── spark_jobs/                 # Spark processing code
│   ├── bronze_ingestion.py     # RAW ingestion to S3 Bronze
│   ├── silver_transform.py     # Bronze cleanup to S3 Silver Parquet
│   └── utils/                  # Spark configurations and quality checks
│
└── docker/                     # Deployment Dockerfiles
    ├── airflow.Dockerfile
    ├── dbt.Dockerfile
    └── spark.Dockerfile
```

---

## Data Vault 2.0 Modeling

The platform adheres to Data Vault 2.0 patterns to guarantee historical auditable records, fast loading, and highly decoupled entities.

```text
               +-----------------------------+
               |        hub_customer         |
               |  - customer_hash_key (PK)   |
               |  - customer_id              |
               |  - load_date                |
               +-----------------------------+
                              |
                              | (1)
                              v
               +-----------------------------+
               |    link_customer_account    |
               |  - customer_account_hash_key|
               |  - customer_hash_key (FK)   |
               |  - account_hash_key (FK)    |
               |  - load_date                |
               +-----------------------------+
                              ^
                              | (1)
                              |
               +-----------------------------+
               |         hub_account         |
               |  - account_hash_key (PK)    |
               |  - account_number           |
               |  - load_date                |
               +-----------------------------+
                              |
                              | (2)
                              v
               +-----------------------------+
               |     sat_account_details     |
               |  - account_hash_key (FK)    |
               |  - balance                  |
               |  - status                   |
               |  - load_date                |
               +-----------------------------+
```

### Table Definitions and Structure

| Type | Pattern | Examples | Purpose |
| :--- | :--- | :--- | :--- |
| **Hub** | `hub_<business_key>` | `hub_customer`, `hub_account` | Stores immutable business keys and tracking metadata. |
| **Link** | `link_<relation>` | `link_customer_account` | Represents relationships or interactions between hub structures. |
| **Satellite** | `sat_<entity>_<context>` | `sat_account_details` | Houses historical context, status, and descriptive attributes. |

---

## Quick-Start Instructions

For comprehensive prerequisites and installation steps, please consult the [Setup Guide](docs/SETUP.md).

```bash
# 1. Clone the repository
git clone https://github.com/manushetgar/AWS_Snowflake_DBT__Project.git
cd AWS_Snowflake_DBT__Project

# 2. Setup your credentials and parameters
./scripts/setup_env.sh

# 3. Apply infrastructure configurations
cd terraform/env/dev
terraform init
terraform apply -auto-approve

# 4. Build and push container images
export IMAGE_TAG=$(git rev-parse --short HEAD)
docker build -t ghcr.io/manushetgar/airflow:$IMAGE_TAG -f docker/airflow.Dockerfile .
docker push ghcr.io/manushetgar/airflow:$IMAGE_TAG

# 5. Deploy workload components
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/

# 6. Run ingestion DAG
airflow dags trigger ingestion_dag
```

---

## CI/CD Pipeline

Automated checks are executed via GitHub Actions workflows on every commit or pull request to validate code quality and prevent infrastructure drift.

```text
       [ Developer Push or Pull Request ]
                       |
                       v
       +-------------------------------+
       |       CI Validation run       |
       +-------------------------------+
         /      |             |      \
        /       |             |       \
       v        v             v        v
   +-------+ +-----+      +-------+ +------+
   | PyLint| | SQL |      |  dbt  | | PyTest|
   | Check | |Lint |      |Compile| | Run  |
   +-------+ +-----+      +-------+ +------+
       \        |             |       /
        \       |             |      /
         v      v             v     v
       +-------------------------------+
       |       Terraform validate      |
       +-------------------------------+
                       |
                       v
       +-------------------------------+
       |      Docker Build & Push      |
       +-------------------------------+
                       |
                       v
       [ Automated Canary Rollout (K8s) ]
```

---

## Operational Observability

Production monitoring targets query execution speed, orchestration statuses, and compute loads across the system:

| Monitor | Endpoint | Application |
| :--- | :--- | :--- |
| **Airflow Dashboard** | `https://airflow.prod.manushetgar.io` | SLA alerts, DAG runs, and execution status. |
| **Snowflake UI** | Native Console | Cost tracking, query history, and table performance. |
| **Prometheus Metrics** | `https://prometheus.prod.manushetgar.io` | Pod resources, memory consumption, Spark metrics. |
| **Grafana Dashboards** | `https://grafana.prod.manushetgar.io` | Custom visualization of transactional volumes. |

---

## Security & Compliance Controls

- **Data Protection**: AES-256 server-side encryption at rest (KMS-CMK with 90-day rotation), TLS 1.3 encryption in-transit.
- **Credential Storage**: Managed in AWS Secrets Manager and loaded dynamically; zero plaintext credentials in git.
- **Isolated Networks**: EKS nodes and RDS components configured within private subnets, bound by secure security groups.
- **Identity & Access Management**: Role-Based Access Control (RBAC) enforced within Snowflake and AWS EKS.

---

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE) for details.

Copyright © 2025 Manu Shetgar.
