# Setup Guide

This guide details the steps required to provision and run the banking data platform locally and on AWS.

## Prerequisites

Ensure you have the following tools installed and configured:
- AWS Account with AdministratorAccess
- Snowflake Account
- Docker Desktop
- Python 3.11 or higher
- Terraform 1.5+
- dbt Core 1.7+

---

## Step-by-Step Deployment

### 1. Clone the Repository
Clone the codebase and navigate into the root directory:
```bash
git clone https://github.com/manushetgar/AWS_Snowflake_DBT__Project.git
cd AWS_Snowflake_DBT__Project
```

### 2. Configure Environment Variables
Copy the template file and fill in your actual AWS and Snowflake credentials:
```bash
cp .env.example .env
# Edit .env to add your credentials and configurations
```

### 3. Provision Infrastructure
Use Terraform to provision AWS resources (S3 buckets, EKS cluster, IAM roles) and Snowflake structures:
```bash
cd terraform/env/dev
terraform init
terraform plan
terraform apply
```

### 4. Configure and Run dbt
Navigate to the dbt project directory to set up dependencies, seeds, and build your schemas:
```bash
cd dbt
dbt deps
dbt debug
dbt seed
dbt run
dbt test
```

### 5. Launch Orchestration
Start Apache Airflow in Docker containers to manage ingestion and transformation pipelines:
```bash
docker-compose up -d
# Access the Airflow UI at http://localhost:8080
# Default credentials: username/password is airflow/airflow
```

---

## CI/CD Pipeline Configuration

The automated workflows in `.github/workflows/ci.yml` validate code quality, dbt compile, and Terraform configurations:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  dbt-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          pip install dbt-core dbt-snowflake
          cd dbt && dbt deps
      
      - name: Run dbt tests
        run: |
          cd dbt
          dbt compile --profiles-dir .
          dbt test --profiles-dir .
        env:
          SNOWFLAKE_ACCOUNT: ${{ secrets.SNOWFLAKE_ACCOUNT }}
          SNOWFLAKE_USER: ${{ secrets.SNOWFLAKE_USER }}
          SNOWFLAKE_PASSWORD: ${{ secrets.SNOWFLAKE_PASSWORD }}

  terraform-validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.5.0
      
      - name: Terraform Format
        run: |
          cd terraform/env/dev
          terraform fmt -check
      
      - name: Terraform Validate
        run: |
          cd terraform/env/dev
          terraform init -backend=false
          terraform validate

  docker-build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Build Docker images
        run: |
          docker build -f docker/airflow.Dockerfile -t airflow:test .
          docker build -f docker/spark.Dockerfile -t spark:test .
```

---

## Deployment & Setup Checklist

Before shifting workloads to production, ensure these security and operational steps are covered:
- [ ] Remove all plaintext secrets and ensure `.env` is ignored by git
- [ ] Configure the Terraform remote state backend in an S3 bucket with DynamoDB locking
- [ ] Set up Snowflake role-based access control (RBAC) and warehouses according to profiles
- [ ] Establish Airflow connections for AWS, Snowflake, and Spark integrations
- [ ] Verify that Snowflake and S3 data quality alerts are routed to your communication channels
