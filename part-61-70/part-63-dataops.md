# Part 63: DataOps & Data Pipeline CI/CD

## บทนำ: DataOps คืออะไร?

DataOps เป็น methodology ที่นำ Agile, DevOps, และ Lean Manufacturing principles มาใช้กับ data analytics และ data engineering โดยเน้นการ automate ทุก step ในกระบวนการจัดการ data ตั้งแต่ ingestion จนถึง delivery

### DataOps vs DevOps

| หัวข้อ | DevOps | DataOps |
|--------|--------|---------|
| Focus | Application deployment | Data pipeline delivery |
| Testing | Code/API tests | Data quality tests |
| Versioning | Source code | Code + Data + Schema |
| Artifacts | Docker images | Datasets, models |
| Monitoring | App performance | Data freshness, quality |
| Tools | Jenkins, GitHub Actions | dbt, Airflow, Great Expectations |

---

## 1. dbt (Data Build Tool)

### 1.1 dbt คืออะไร และทำงานอย่างไร

dbt เปลี่ยนวิธีที่ data team สร้าง transformation logic โดยใช้ SQL เป็นหลัก แต่เพิ่ม software engineering practices อย่าง testing, documentation, และ version control

### 1.2 โครงสร้าง dbt Project

```
dbt_project/
├── dbt_project.yml           # Project configuration
├── profiles.yml              # Database connections
├── packages.yml              # dbt packages
├── models/
│   ├── staging/              # Source data cleaning
│   │   ├── stg_orders.sql
│   │   ├── stg_customers.sql
│   │   └── schema.yml        # Tests & docs
│   ├── intermediate/         # Business logic
│   │   ├── int_order_items.sql
│   │   └── schema.yml
│   └── marts/                # Analytics-ready models
│       ├── finance/
│       │   ├── fct_orders.sql
│       │   ├── dim_customers.sql
│       │   └── schema.yml
│       └── marketing/
│           ├── fct_sessions.sql
│           └── schema.yml
├── tests/                    # Custom data tests
│   ├── assert_positive_revenue.sql
│   └── check_referential_integrity.sql
├── macros/                   # Reusable SQL macros
│   ├── cents_to_dollars.sql
│   └── generate_schema_name.sql
├── seeds/                    # Static reference data
│   ├── country_codes.csv
│   └── product_categories.csv
├── snapshots/               # SCD Type 2
│   └── customer_snapshot.sql
└── analyses/                # Ad-hoc queries
    └── customer_lifetime_value.sql
```

### 1.3 dbt_project.yml

```yaml
# dbt_project.yml
name: 'ecommerce_analytics'
version: '1.0.0'
config-version: 2

profile: 'ecommerce'

model-paths: ["models"]
analysis-paths: ["analyses"]
test-paths: ["tests"]
seed-paths: ["seeds"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]

target-path: "target"
clean-targets:
  - "target"
  - "dbt_packages"

models:
  ecommerce_analytics:
    staging:
      +materialized: view
      +schema: staging
    intermediate:
      +materialized: ephemeral  # ไม่ create table จริง
    marts:
      +materialized: table
      finance:
        +schema: finance
        +tags: ['finance', 'daily']
      marketing:
        +schema: marketing
        +tags: ['marketing', 'hourly']

vars:
  start_date: '2024-01-01'
  payment_methods: ['credit_card', 'debit_card', 'bank_transfer']
```

### 1.4 Staging Models

```sql
-- models/staging/stg_orders.sql
-- ทำความสะอาด raw data และ standardize column names

WITH source AS (
    SELECT * FROM {{ source('raw', 'orders') }}
),

renamed AS (
    SELECT
        -- Identifiers
        id                          AS order_id,
        user_id                     AS customer_id,
        
        -- Timestamps
        created_at                  AS created_at,
        updated_at                  AS updated_at,
        {{ convert_timezone('UTC', 'Asia/Bangkok', 'created_at') }} AS created_at_bkk,
        
        -- Order details
        status,
        UPPER(payment_method)       AS payment_method,
        
        -- Financial amounts (convert cents to dollars)
        {{ cents_to_dollars('amount') }}        AS amount,
        {{ cents_to_dollars('shipping_amount') }} AS shipping_amount,
        {{ cents_to_dollars('discount_amount') }} AS discount_amount,
        
        -- Derived
        amount - discount_amount    AS net_amount,
        
        -- Metadata
        _loaded_at
    
    FROM source
    WHERE
        id IS NOT NULL
        AND status NOT IN ('test', 'demo')
        AND created_at >= '{{ var("start_date") }}'
)

SELECT * FROM renamed
```

```sql
-- models/staging/schema.yml
version: 2

models:
  - name: stg_orders
    description: "Cleaned and standardized orders from raw source"
    
    columns:
      - name: order_id
        description: "Primary key for orders"
        tests:
          - unique
          - not_null
      
      - name: customer_id
        description: "FK to customers table"
        tests:
          - not_null
          - relationships:
              to: ref('stg_customers')
              field: customer_id
      
      - name: status
        tests:
          - not_null
          - accepted_values:
              values: ['pending', 'processing', 'shipped', 'delivered', 'cancelled', 'refunded']
      
      - name: payment_method
        tests:
          - accepted_values:
              values: "{{ var('payment_methods') }}"
      
      - name: amount
        tests:
          - not_null
          - dbt_utils.expression_is_true:
              expression: ">= 0"
      
      - name: created_at
        tests:
          - not_null
          - dbt_utils.expression_is_true:
              expression: ">= '2024-01-01'"
```

### 1.5 Fact Tables

```sql
-- models/marts/finance/fct_orders.sql
WITH orders AS (
    SELECT * FROM {{ ref('stg_orders') }}
),

order_items AS (
    SELECT * FROM {{ ref('int_order_items') }}
),

customers AS (
    SELECT * FROM {{ ref('dim_customers') }}
),

order_summary AS (
    SELECT
        o.order_id,
        o.customer_id,
        c.customer_segment,
        c.acquisition_channel,
        
        o.created_at,
        o.created_at_bkk,
        DATE_TRUNC('day', o.created_at)   AS order_date,
        DATE_TRUNC('month', o.created_at) AS order_month,
        
        o.status,
        o.payment_method,
        
        -- Financial
        o.amount                        AS gross_amount,
        o.discount_amount,
        o.shipping_amount,
        o.net_amount,
        
        -- Item counts
        oi.item_count,
        oi.unique_product_count,
        
        -- Derived metrics
        o.net_amount / NULLIF(oi.item_count, 0) AS avg_item_value,
        
        -- Flags
        CASE WHEN o.discount_amount > 0 THEN TRUE ELSE FALSE END AS has_discount,
        CASE WHEN o.status = 'cancelled' THEN TRUE ELSE FALSE END AS is_cancelled,
        
        o._loaded_at
    
    FROM orders o
    LEFT JOIN order_items oi USING (order_id)
    LEFT JOIN customers c USING (customer_id)
)

SELECT * FROM order_summary
```

### 1.6 dbt Tests

```yaml
# models/marts/finance/schema.yml
version: 2

models:
  - name: fct_orders
    description: "Order-level fact table with all order details"
    
    tests:
      - dbt_utils.recency:
          datepart: hour
          field: created_at
          interval: 4  # Data should be at most 4 hours old
    
    columns:
      - name: order_id
        tests:
          - unique
          - not_null
      
      - name: gross_amount
        tests:
          - not_null
          - dbt_utils.expression_is_true:
              expression: ">= 0"
              config:
                severity: error
      
      - name: net_amount
        tests:
          - dbt_utils.expression_is_true:
              expression: "<= gross_amount"
              name: net_amount_should_not_exceed_gross
      
      - name: order_date
        tests:
          - not_null
          - dbt_utils.expression_is_true:
              expression: ">= '2024-01-01'"

sources:
  - name: raw
    description: "Raw data from operational databases"
    database: raw_db
    schema: public
    
    freshness:
      warn_after: {count: 2, period: hour}
      error_after: {count: 6, period: hour}
    
    tables:
      - name: orders
        description: "Raw orders table from application database"
        loaded_at_field: _loaded_at
        
        columns:
          - name: id
            tests:
              - unique
              - not_null
```

### 1.7 Custom Data Tests

```sql
-- tests/assert_positive_revenue.sql
-- ทดสอบว่า revenue ทุก row เป็นบวก

SELECT
    order_id,
    net_amount
FROM {{ ref('fct_orders') }}
WHERE
    status NOT IN ('cancelled', 'refunded')
    AND net_amount < 0
```

```sql
-- tests/check_no_duplicate_payments.sql
-- ทดสอบว่าไม่มี payment ที่ duplicate

SELECT
    payment_reference,
    COUNT(*) AS cnt
FROM {{ ref('fct_payments') }}
GROUP BY 1
HAVING COUNT(*) > 1
```

---

## 2. Apache Airflow สำหรับ Data Pipelines

### 2.1 Airflow DAG Structure

```python
# dags/ecommerce_pipeline.py
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator, BranchPythonOperator
from airflow.operators.bash import BashOperator
from airflow.providers.amazon.aws.operators.s3 import S3CreateObjectOperator
from airflow.providers.postgres.operators.postgres import PostgresOperator
from airflow.utils.trigger_rule import TriggerRule
import logging

logger = logging.getLogger(__name__)

# Default arguments
default_args = {
    'owner': 'data-engineering',
    'depends_on_past': False,
    'start_date': datetime(2024, 1, 1),
    'email': ['data-alerts@company.com'],
    'email_on_failure': True,
    'email_on_retry': False,
    'retries': 3,
    'retry_delay': timedelta(minutes=5),
    'retry_exponential_backoff': True,
    'max_retry_delay': timedelta(minutes=30),
    'execution_timeout': timedelta(hours=2)
}

with DAG(
    dag_id='ecommerce_daily_pipeline',
    description='Daily ecommerce data pipeline',
    default_args=default_args,
    schedule_interval='0 2 * * *',  # 2 AM daily
    catchup=False,
    max_active_runs=1,
    tags=['ecommerce', 'daily', 'finance'],
    doc_md="""
    ## Ecommerce Daily Pipeline
    
    This DAG orchestrates the daily ecommerce data pipeline:
    1. Ingest orders and customer data
    2. Validate data quality
    3. Run dbt transformations
    4. Generate reports
    
    **SLA**: Must complete by 6 AM
    
    **On Failure**: Alert #data-alerts Slack channel
    """,
    sla_miss_callback=lambda dag, task_list, *args: 
        notify_slack(f"SLA missed for {dag.dag_id}")
) as dag:

    # ==================== Data Ingestion ====================
    
    def ingest_orders(**context):
        """Ingest orders from source systems"""
        from operators.ingestion import OrdersIngester
        
        execution_date = context['ds']
        ingester = OrdersIngester(date=execution_date)
        
        records_ingested = ingester.run()
        
        # Push to XCom for downstream tasks
        context['ti'].xcom_push(
            key='orders_count',
            value=records_ingested
        )
        
        logger.info(f"Ingested {records_ingested} orders for {execution_date}")
        return records_ingested
    
    ingest_orders_task = PythonOperator(
        task_id='ingest_orders',
        python_callable=ingest_orders,
        provide_context=True
    )
    
    ingest_customers_task = PythonOperator(
        task_id='ingest_customers',
        python_callable=lambda **ctx: ingest_table('customers', ctx['ds']),
        provide_context=True
    )
    
    # ==================== Data Validation ====================
    
    def validate_data_quality(**context):
        """Run Great Expectations data validation"""
        from operators.validation import DataValidator
        
        execution_date = context['ds']
        
        validator = DataValidator()
        results = validator.validate_all(date=execution_date)
        
        # Check if any critical tests failed
        failed_critical = [
            r for r in results
            if not r['success'] and r['severity'] == 'critical'
        ]
        
        if failed_critical:
            raise ValueError(
                f"Critical data quality checks failed: {failed_critical}"
            )
        
        # Log warnings
        failed_warnings = [r for r in results if not r['success']]
        if failed_warnings:
            logger.warning(f"Data quality warnings: {failed_warnings}")
        
        context['ti'].xcom_push(key='validation_results', value=results)
        return len(failed_critical) == 0
    
    validate_task = PythonOperator(
        task_id='validate_data_quality',
        python_callable=validate_data_quality,
        provide_context=True
    )
    
    # ==================== dbt Transformations ====================
    
    dbt_staging = BashOperator(
        task_id='dbt_run_staging',
        bash_command="""
            cd /opt/dbt/ecommerce_analytics &&
            dbt run \
                --profiles-dir /opt/dbt/profiles \
                --target prod \
                --models staging \
                --vars '{"execution_date": "{{ ds }}"}'
        """
    )
    
    dbt_marts = BashOperator(
        task_id='dbt_run_marts',
        bash_command="""
            cd /opt/dbt/ecommerce_analytics &&
            dbt run \
                --profiles-dir /opt/dbt/profiles \
                --target prod \
                --models marts \
                --vars '{"execution_date": "{{ ds }}"}'
        """
    )
    
    dbt_test = BashOperator(
        task_id='dbt_test',
        bash_command="""
            cd /opt/dbt/ecommerce_analytics &&
            dbt test \
                --profiles-dir /opt/dbt/profiles \
                --target prod \
                --models staging marts \
                --store-failures
        """
    )
    
    # ==================== Report Generation ====================
    
    def generate_daily_report(**context):
        """Generate daily summary report"""
        from operators.reporting import ReportGenerator
        
        execution_date = context['ds']
        orders_count = context['ti'].xcom_pull(
            task_ids='ingest_orders',
            key='orders_count'
        )
        
        generator = ReportGenerator(date=execution_date)
        report = generator.create_daily_summary(orders_count=orders_count)
        
        # Send to Slack
        generator.send_to_slack(report)
        
        return report
    
    generate_report_task = PythonOperator(
        task_id='generate_daily_report',
        python_callable=generate_daily_report,
        provide_context=True,
        trigger_rule=TriggerRule.ALL_SUCCESS
    )
    
    # ==================== Failure Handler ====================
    
    def on_pipeline_failure(**context):
        """Handle pipeline failure"""
        task_instance = context['task_instance']
        dag_id = context['dag'].dag_id
        execution_date = context['ds']
        
        message = f"""
        Pipeline Failure Alert!
        
        DAG: {dag_id}
        Task: {task_instance.task_id}
        Date: {execution_date}
        Error: {context.get('exception', 'Unknown error')}
        
        Logs: {task_instance.log_url}
        """
        
        notify_slack(message, channel='#data-alerts', severity='error')
    
    failure_handler = PythonOperator(
        task_id='handle_failure',
        python_callable=on_pipeline_failure,
        provide_context=True,
        trigger_rule=TriggerRule.ONE_FAILED
    )
    
    # ==================== Pipeline Dependencies ====================
    
    [ingest_orders_task, ingest_customers_task] >> validate_task
    validate_task >> dbt_staging >> dbt_marts >> dbt_test
    dbt_test >> generate_report_task
    
    # Failure handler runs if anything fails
    [ingest_orders_task, ingest_customers_task, validate_task, 
     dbt_staging, dbt_marts, dbt_test] >> failure_handler
```

---

## 3. Great Expectations สำหรับ Data Quality

### 3.1 Setup Great Expectations

```python
# great_expectations_setup.py
import great_expectations as gx

# Initialize GE context
context = gx.get_context()

# Add Datasource (PostgreSQL)
datasource = context.sources.add_postgres(
    name="ecommerce_postgres",
    connection_string="${DB_CONNECTION_STRING}"
)

# Add Data Asset
orders_asset = datasource.add_table_asset(
    name="orders",
    table_name="raw.orders"
)

# Add Batch Definition
batch_definition = orders_asset.add_daily_partitioner_with_parameter_defaults(
    name="daily_orders"
)

print("Great Expectations setup complete!")
```

### 3.2 Expectation Suite สำหรับ Orders

```python
# expectations/orders_suite.py
import great_expectations as gx
from great_expectations.core.expectation_configuration import ExpectationConfiguration

context = gx.get_context()

# Create suite
suite = context.add_expectation_suite(
    expectation_suite_name="orders.critical"
)

# ==================== Schema Expectations ====================

# Expect specific columns to exist
suite.add_expectation(ExpectationConfiguration(
    expectation_type="expect_table_columns_to_match_ordered_list",
    kwargs={
        "column_list": [
            "id", "user_id", "status", "amount", 
            "payment_method", "created_at", "updated_at"
        ]
    }
))

# ==================== Column Existence ====================

suite.add_expectation(ExpectationConfiguration(
    expectation_type="expect_column_to_exist",
    kwargs={"column": "id"}
))

# ==================== Null Checks ====================

for col in ["id", "user_id", "status", "amount", "created_at"]:
    suite.add_expectation(ExpectationConfiguration(
        expectation_type="expect_column_values_to_not_be_null",
        kwargs={"column": col},
        meta={"severity": "critical"}
    ))

# ==================== Uniqueness ====================

suite.add_expectation(ExpectationConfiguration(
    expectation_type="expect_column_values_to_be_unique",
    kwargs={"column": "id"},
    meta={"severity": "critical"}
))

# ==================== Value Ranges ====================

suite.add_expectation(ExpectationConfiguration(
    expectation_type="expect_column_values_to_be_between",
    kwargs={
        "column": "amount",
        "min_value": 0,
        "max_value": 1000000  # สูงสุด 1 ล้านบาท
    }
))

# ==================== Categorical Values ====================

suite.add_expectation(ExpectationConfiguration(
    expectation_type="expect_column_values_to_be_in_set",
    kwargs={
        "column": "status",
        "value_set": ["pending", "processing", "shipped", 
                      "delivered", "cancelled", "refunded"]
    }
))

suite.add_expectation(ExpectationConfiguration(
    expectation_type="expect_column_values_to_be_in_set",
    kwargs={
        "column": "payment_method",
        "value_set": ["credit_card", "debit_card", "bank_transfer", 
                      "wallet", "cod"]
    }
))

# ==================== Row Count ====================

# Expect at least 100 orders per day
suite.add_expectation(ExpectationConfiguration(
    expectation_type="expect_table_row_count_to_be_between",
    kwargs={
        "min_value": 100,
        "max_value": 100000
    },
    meta={"severity": "warning"}
))

# ==================== Statistical ====================

# Amount ควรมี mean ระหว่าง 500-5000 บาท
suite.add_expectation(ExpectationConfiguration(
    expectation_type="expect_column_mean_to_be_between",
    kwargs={
        "column": "amount",
        "min_value": 100,
        "max_value": 50000
    },
    meta={"severity": "warning"}
))

context.save_expectation_suite(suite)
print("Expectation suite saved!")
```

### 3.3 Run Validation

```python
# scripts/validate_data.py
import great_expectations as gx
from great_expectations.core.batch import RuntimeBatchRequest
import json
import sys
from datetime import date

def validate_orders(execution_date: str) -> dict:
    context = gx.get_context()
    
    # Get batch
    batch_request = RuntimeBatchRequest(
        datasource_name="ecommerce_postgres",
        data_connector_name="default_runtime_data_connector_name",
        data_asset_name="orders",
        runtime_parameters={
            "query": f"""
                SELECT * FROM raw.orders 
                WHERE DATE(created_at) = '{execution_date}'
            """
        },
        batch_identifiers={
            "default_identifier_name": f"orders_{execution_date}"
        }
    )
    
    # Create validator
    validator = context.get_validator(
        batch_request=batch_request,
        expectation_suite_name="orders.critical"
    )
    
    # Run validation
    checkpoint_result = context.run_checkpoint(
        checkpoint_name="orders_daily_checkpoint",
        validations=[
            {
                "batch_request": batch_request,
                "expectation_suite_name": "orders.critical"
            }
        ]
    )
    
    # Process results
    results = {
        "success": checkpoint_result.success,
        "execution_date": execution_date,
        "statistics": checkpoint_result.run_results
    }
    
    if not checkpoint_result.success:
        failed_checks = [
            result for result in checkpoint_result.run_results.values()
            if not result["success"]
        ]
        results["failed_checks"] = failed_checks
        
        # Check severity
        critical_failures = [
            r for r in failed_checks
            if r.get("meta", {}).get("severity") == "critical"
        ]
        
        if critical_failures:
            print(f"CRITICAL: Data validation failed with {len(critical_failures)} critical errors")
            sys.exit(1)
        else:
            print(f"WARNING: Data validation had {len(failed_checks)} warnings")
    
    return results

if __name__ == "__main__":
    import argparse
    parser = argparse.ArgumentParser()
    parser.add_argument("--date", default=str(date.today()))
    args = parser.parse_args()
    
    results = validate_orders(args.date)
    print(json.dumps(results, indent=2, default=str))
```

---

## 4. Data Versioning

### 4.1 DVC สำหรับ Datasets

```yaml
# dvc.yaml สำหรับ data pipeline
stages:
  extract:
    cmd: python src/extract.py --date ${execution_date}
    deps:
      - src/extract.py
    params:
      - extraction.sources
    outs:
      - data/raw/${execution_date}/orders.parquet
      - data/raw/${execution_date}/customers.parquet

  validate:
    cmd: python src/validate.py --date ${execution_date}
    deps:
      - src/validate.py
      - data/raw/${execution_date}/orders.parquet
      - expectations/orders_suite.json
    outs:
      - data/validation/${execution_date}/report.json
    metrics:
      - data/validation/${execution_date}/metrics.json:
          cache: false

  transform:
    cmd: |
      dbt run \
        --profiles-dir profiles \
        --vars '{"execution_date": "${execution_date}"}' \
        --models staging marts
    deps:
      - models/
      - data/raw/${execution_date}/
    outs:
      - data/transformed/${execution_date}/
```

### 4.2 Delta Lake สำหรับ Data Versioning

```python
# src/storage/delta_lake.py
from delta import DeltaTable, configure_spark_with_delta_pip
from pyspark.sql import SparkSession
import pyspark.sql.functions as F

def create_spark():
    builder = SparkSession.builder \
        .appName("EcommerceDataPipeline") \
        .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension") \
        .config("spark.sql.catalog.spark_catalog", 
                "org.apache.spark.sql.delta.catalog.DeltaCatalog")
    
    return configure_spark_with_delta_pip(builder).getOrCreate()

spark = create_spark()

def upsert_orders(new_data_df, table_path: str):
    """Upsert orders ด้วย Delta Lake merge"""
    
    if DeltaTable.isDeltaTable(spark, table_path):
        delta_table = DeltaTable.forPath(spark, table_path)
        
        # Merge: update existing + insert new
        delta_table.alias("target") \
            .merge(
                new_data_df.alias("source"),
                "target.order_id = source.order_id"
            ) \
            .whenMatchedUpdate(set={
                "status": "source.status",
                "updated_at": "source.updated_at",
                "amount": "source.amount"
            }) \
            .whenNotMatchedInsertAll() \
            .execute()
        
        print(f"Upserted {new_data_df.count()} records")
        
    else:
        # Initial write
        new_data_df.write \
            .format("delta") \
            .mode("overwrite") \
            .partitionBy("order_date") \
            .save(table_path)
        
        print(f"Created Delta table with {new_data_df.count()} records")

def time_travel_query(table_path: str, version: int = None, timestamp: str = None):
    """Query data at specific version or timestamp"""
    
    if version is not None:
        df = spark.read \
            .format("delta") \
            .option("versionAsOf", version) \
            .load(table_path)
    elif timestamp is not None:
        df = spark.read \
            .format("delta") \
            .option("timestampAsOf", timestamp) \
            .load(table_path)
    else:
        df = spark.read.format("delta").load(table_path)
    
    return df

def get_table_history(table_path: str) -> None:
    """แสดง history ของ Delta table"""
    delta_table = DeltaTable.forPath(spark, table_path)
    
    history = delta_table.history()
    history.select(
        "version", "timestamp", "operation", 
        "operationParameters", "operationMetrics"
    ).show(truncate=False)
```

---

## 5. CI/CD สำหรับ Data Pipelines

### 5.1 GitHub Actions สำหรับ dbt

```yaml
# .github/workflows/dbt-ci.yml
name: dbt CI Pipeline

on:
  pull_request:
    branches: [main]
    paths:
      - 'models/**'
      - 'tests/**'
      - 'macros/**'
      - 'dbt_project.yml'

jobs:
  dbt-test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:14
        env:
          POSTGRES_DB: test_db
          POSTGRES_USER: dbt_user
          POSTGRES_PASSWORD: ${{ secrets.TEST_DB_PASSWORD }}
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.10'
          cache: 'pip'
      
      - name: Install dbt
        run: |
          pip install dbt-postgres dbt-utils
      
      - name: Setup test profiles
        run: |
          mkdir -p ~/.dbt
          cat > ~/.dbt/profiles.yml << EOF
          ecommerce:
            target: ci
            outputs:
              ci:
                type: postgres
                host: localhost
                port: 5432
                dbname: test_db
                user: dbt_user
                password: ${{ secrets.TEST_DB_PASSWORD }}
                schema: dbt_ci_${{ github.event.pull_request.number }}
                threads: 4
          EOF
      
      - name: Load seed data
        run: |
          cd dbt_project
          dbt seed --profiles-dir ~/.dbt --target ci
      
      - name: Compile dbt models
        run: |
          cd dbt_project
          dbt compile --profiles-dir ~/.dbt --target ci
      
      - name: Run dbt models
        run: |
          cd dbt_project
          dbt run --profiles-dir ~/.dbt --target ci --full-refresh
      
      - name: Run dbt tests
        run: |
          cd dbt_project
          dbt test --profiles-dir ~/.dbt --target ci --store-failures
      
      - name: Generate docs
        if: always()
        run: |
          cd dbt_project
          dbt docs generate --profiles-dir ~/.dbt --target ci
      
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: dbt-test-results
          path: |
            dbt_project/target/run_results.json
            dbt_project/target/manifest.json
      
      - name: Comment PR with results
        if: always()
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const runResults = JSON.parse(fs.readFileSync('dbt_project/target/run_results.json', 'utf8'));
            
            const passed = runResults.results.filter(r => r.status === 'pass').length;
            const failed = runResults.results.filter(r => r.status === 'fail').length;
            const errors = runResults.results.filter(r => r.status === 'error').length;
            
            const body = `## dbt Test Results
            
            | Status | Count |
            |--------|-------|
            | ✅ Pass | ${passed} |
            | ❌ Fail | ${failed} |
            | ⚠️ Error | ${errors} |
            
            ${failed > 0 || errors > 0 ? '> ⚠️ Some tests failed. Please check the details.' : '> ✅ All tests passed!'}
            `;
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: body
            });
      
      - name: Cleanup CI schema
        if: always()
        run: |
          psql "postgresql://dbt_user:${{ secrets.TEST_DB_PASSWORD }}@localhost:5432/test_db" \
            -c "DROP SCHEMA IF EXISTS dbt_ci_${{ github.event.pull_request.number }} CASCADE;"
```

### 5.2 Airflow Pipeline Testing

```python
# tests/test_dag.py
import pytest
from airflow.models import DagBag
from airflow.utils.state import DagRunState
from airflow.utils.types import DagRunType
from datetime import datetime

class TestEcommercePipeline:
    
    @pytest.fixture
    def dagbag(self):
        return DagBag(dag_folder='dags/', include_examples=False)
    
    def test_dag_loads_without_errors(self, dagbag):
        """ทดสอบว่า DAG load ได้ไม่มี error"""
        assert len(dagbag.import_errors) == 0, \
            f"DAG import errors: {dagbag.import_errors}"
    
    def test_dag_exists(self, dagbag):
        """ทดสอบว่า DAG มีอยู่จริง"""
        assert 'ecommerce_daily_pipeline' in dagbag.dags
    
    def test_dag_structure(self, dagbag):
        """ทดสอบ structure ของ DAG"""
        dag = dagbag.dags['ecommerce_daily_pipeline']
        
        # ตรวจสอบ tasks ที่ต้องมี
        required_tasks = [
            'ingest_orders',
            'ingest_customers', 
            'validate_data_quality',
            'dbt_run_staging',
            'dbt_run_marts',
            'dbt_test',
            'generate_daily_report'
        ]
        
        for task_id in required_tasks:
            assert dag.has_task(task_id), f"Missing task: {task_id}"
    
    def test_task_dependencies(self, dagbag):
        """ทดสอบ task dependencies"""
        dag = dagbag.dags['ecommerce_daily_pipeline']
        
        # validate ต้องรันหลัง ingest
        validate_task = dag.get_task('validate_data_quality')
        upstream_task_ids = {t.task_id for t in validate_task.upstream_list}
        
        assert 'ingest_orders' in upstream_task_ids
        assert 'ingest_customers' in upstream_task_ids
    
    def test_dag_schedule(self, dagbag):
        """ทดสอบ schedule interval"""
        dag = dagbag.dags['ecommerce_daily_pipeline']
        
        assert dag.schedule_interval == '0 2 * * *'
        assert dag.catchup == False
    
    def test_dag_retries(self, dagbag):
        """ทดสอบ retry configuration"""
        dag = dagbag.dags['ecommerce_daily_pipeline']
        
        for task in dag.tasks:
            assert task.retries >= 1, f"Task {task.task_id} should have retries"
    
    @pytest.mark.integration
    def test_dag_run(self, dagbag):
        """Integration test: รัน DAG จริงกับ test data"""
        dag = dagbag.dags['ecommerce_daily_pipeline']
        
        dag_run = dag.create_dagrun(
            run_type=DagRunType.MANUAL,
            execution_date=datetime(2024, 1, 1),
            state=DagRunState.RUNNING,
            conf={'test_mode': True, 'test_date': '2024-01-01'}
        )
        
        # รัน DAG
        dag.run(
            start_date=dag_run.execution_date,
            end_date=dag_run.execution_date
        )
        
        # ตรวจสอบผลลัพธ์
        assert dag_run.state == DagRunState.SUCCESS
```

---

## 6. Pipeline Testing Strategies

### 6.1 Unit Testing Data Transformations

```python
# tests/unit/test_transformations.py
import pytest
import pandas as pd
import numpy as np
from src.transformations import OrderTransformer

class TestOrderTransformer:
    
    @pytest.fixture
    def transformer(self):
        return OrderTransformer()
    
    @pytest.fixture
    def sample_orders(self):
        return pd.DataFrame({
            'id': [1, 2, 3, 4],
            'amount_cents': [1000, 2500, 500, 10000],
            'status': ['delivered', 'cancelled', 'pending', 'delivered'],
            'created_at': pd.to_datetime(['2024-01-01', '2024-01-02', 
                                          '2024-01-03', '2024-01-04']),
            'payment_method': ['CREDIT_CARD', 'DEBIT_CARD', 'BANK_TRANSFER', 'WALLET']
        })
    
    def test_converts_cents_to_dollars(self, transformer, sample_orders):
        result = transformer.transform(sample_orders)
        
        assert result['amount'].tolist() == [10.0, 25.0, 5.0, 100.0]
    
    def test_lowercase_payment_method(self, transformer, sample_orders):
        result = transformer.transform(sample_orders)
        
        assert result['payment_method'].tolist() == [
            'credit_card', 'debit_card', 'bank_transfer', 'wallet'
        ]
    
    def test_adds_is_cancelled_flag(self, transformer, sample_orders):
        result = transformer.transform(sample_orders)
        
        assert result['is_cancelled'].tolist() == [False, True, False, False]
    
    def test_handles_empty_dataframe(self, transformer):
        empty_df = pd.DataFrame(columns=['id', 'amount_cents', 'status', 
                                          'created_at', 'payment_method'])
        result = transformer.transform(empty_df)
        
        assert len(result) == 0
        assert 'amount' in result.columns
    
    def test_handles_null_amounts(self, transformer, sample_orders):
        sample_orders.loc[0, 'amount_cents'] = np.nan
        
        result = transformer.transform(sample_orders)
        
        assert pd.isna(result.loc[0, 'amount']) or result.loc[0, 'amount'] == 0
```

### 6.2 Integration Testing Pipeline

```python
# tests/integration/test_pipeline.py
import pytest
import pandas as pd
from datetime import date
import psycopg2

@pytest.fixture(scope='session')
def test_db():
    """Create test database connection"""
    conn = psycopg2.connect(
        host='localhost',
        database='test_db',
        user='test_user',
        password='test_pass'
    )
    yield conn
    conn.close()

@pytest.fixture(autouse=True)
def setup_test_data(test_db):
    """Setup test data before each test"""
    with test_db.cursor() as cur:
        # Create test orders
        cur.execute("""
            INSERT INTO raw.orders (id, user_id, amount, status, payment_method, created_at)
            VALUES 
                (1, 'user1', 1500, 'delivered', 'credit_card', '2024-01-01'),
                (2, 'user2', 3000, 'cancelled', 'debit_card', '2024-01-01'),
                (3, 'user1', 750, 'processing', 'bank_transfer', '2024-01-01')
            ON CONFLICT (id) DO NOTHING
        """)
        test_db.commit()
    
    yield
    
    # Cleanup
    with test_db.cursor() as cur:
        cur.execute("DELETE FROM raw.orders WHERE id IN (1, 2, 3)")
        test_db.commit()

def test_orders_pipeline_integration(test_db):
    """Test complete order pipeline end-to-end"""
    from src.pipeline import run_daily_pipeline
    
    # Run pipeline for test date
    result = run_daily_pipeline(
        execution_date='2024-01-01',
        db_connection=test_db
    )
    
    assert result['status'] == 'success'
    assert result['orders_processed'] == 3
    
    # Verify transformed data
    with test_db.cursor() as cur:
        cur.execute("""
            SELECT COUNT(*) 
            FROM marts.fct_orders 
            WHERE order_date = '2024-01-01'
        """)
        count = cur.fetchone()[0]
    
    assert count == 3

def test_data_quality_gates_block_bad_data(test_db):
    """Test that bad data triggers quality gate failure"""
    from src.validation import run_validations
    
    # Insert bad data (negative amount)
    with test_db.cursor() as cur:
        cur.execute("""
            INSERT INTO raw.orders (id, user_id, amount, status, payment_method, created_at)
            VALUES (999, 'user1', -100, 'delivered', 'credit_card', '2024-01-02')
        """)
        test_db.commit()
    
    with pytest.raises(ValueError, match="Critical data quality checks failed"):
        run_validations(execution_date='2024-01-02', db_connection=test_db)
    
    # Cleanup
    with test_db.cursor() as cur:
        cur.execute("DELETE FROM raw.orders WHERE id = 999")
        test_db.commit()
```

---

## 7. แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: Build dbt Pipeline

```
Task: สร้าง dbt project สำหรับ e-commerce analytics

Dataset: Brazilian E-Commerce Dataset (Olist) จาก Kaggle

Requirements:
1. Staging models สำหรับ:
   - stg_orders
   - stg_customers  
   - stg_products
   - stg_reviews
   
2. Mart models:
   - fct_orders (order-level facts)
   - dim_customers (customer dimension)
   - fct_reviews (review analysis)
   - rpt_daily_sales (daily aggregation)

3. Tests สำหรับทุก model:
   - Uniqueness, not null
   - Referential integrity
   - Business rule tests

4. CI/CD ด้วย GitHub Actions:
   - Run tests ใน PR
   - Deploy to staging เมื่อ merge
   - Auto-cleanup CI schemas

Bonus:
- Document ทุก model และ column
- ตั้ง freshness checks บน sources
- สร้าง custom macros
```

### แบบฝึกหัดที่ 2: Data Quality Dashboard

```python
# สร้าง script ที่:
# 1. Run Great Expectations validations
# 2. Generate HTML report
# 3. Track quality metrics over time
# 4. Alert เมื่อ quality ต่ำกว่า threshold

def build_quality_dashboard():
    """
    TODO:
    1. Load expectation suites จาก GE
    2. Run validations กับ last 7 days data
    3. Calculate quality score per day
    4. Generate trend visualization
    5. Send email/Slack report พร้อม trend
    """
    pass
```

### แบบฝึกหัดที่ 3: Airflow Pipeline with Tests

```
สร้าง Airflow DAG ที่:
1. Ingest data จาก Mock API ทุก 6 ชั่วโมง
2. Run Great Expectations validations
3. Transform ด้วย dbt (staging + marts)
4. Generate summary metrics
5. Alert on failure ผ่าน Slack

Tests ที่ต้องเขียน:
- Unit tests สำหรับ operator functions
- DAG structure tests
- Dependency tests
- Integration test (dry run)
```

### สรุปบทที่ 63

ในบทนี้เราได้เรียนรู้:
- **dbt**: SQL-based transformation ด้วย version control, testing, documentation
- **Apache Airflow**: Orchestrating data pipelines ด้วย DAGs
- **Great Expectations**: Data quality validation framework
- **Delta Lake**: Data versioning และ time travel สำหรับ data lakes
- **Pipeline Testing**: Unit และ integration tests สำหรับ data code
- **CI/CD สำหรับ Data**: GitHub Actions pipeline สำหรับ dbt และ Airflow

บทถัดไปเราจะเรียนรู้ Mobile App CI/CD ด้วย Fastlane และ tools อื่น ๆ
