# SYNQ CLI

Command-line tool for managing data quality tests and monitors on the [SYNQ](https://synq.io) platform.

## Features

- **Deploy** - Deploy data quality tests and monitors from YAML configuration files
- **Advisor** - Get AI-powered suggestions for data quality tests based on your schema
- **Export** - Export existing monitors to YAML format

## Installation

### macOS

```bash
# Apple Silicon
curl -L https://github.com/getsynq/synqcli/releases/latest/download/synqcli_darwin_arm64.tar.gz | tar -xz
sudo mv synqcli /usr/local/bin/

# Intel
curl -L https://github.com/getsynq/synqcli/releases/latest/download/synqcli_darwin_amd64.tar.gz | tar -xz
sudo mv synqcli /usr/local/bin/
```

### Linux

```bash
# AMD64
curl -L https://github.com/getsynq/synqcli/releases/latest/download/synqcli_linux_amd64.tar.gz | tar -xz
sudo mv synqcli /usr/local/bin/

# ARM64
curl -L https://github.com/getsynq/synqcli/releases/latest/download/synqcli_linux_arm64.tar.gz | tar -xz
sudo mv synqcli /usr/local/bin/
```

### Windows

Download the latest release from the [releases page](https://github.com/getsynq/synqcli/releases) and extract `synqcli.exe` to your PATH.

## Configuration

### SYNQ API Credentials

Set your SYNQ API credentials via environment variables:

```bash
export SYNQ_CLIENT_ID="your-client-id"
export SYNQ_CLIENT_SECRET="your-client-secret"
export SYNQ_API_URL="https://developer.synq.io"  # or https://api.us.synq.io for US region
```

Or create a `.env` file in your project root:

```bash
SYNQ_CLIENT_ID=your-client-id
SYNQ_CLIENT_SECRET=your-client-secret
SYNQ_API_URL=https://developer.synq.io
```

Or use command-line flags (highest priority):

```bash
synqcli deploy --client-id="your-id" --client-secret="your-secret" --api-url="https://developer.synq.io"
```

**Priority order:** Command-line flags > Environment variables > .env file

### Advisor Credentials

For the `advisor` command, you also need an OpenAI-compatible API key:

```bash
export OPENAI_API_KEY="your-api-key"
export OPENAI_BASE_URL="https://api.openai.com/v1"  # optional, for custom endpoints like LiteLLM
```

---

## Commands

### Deploy

Deploy data quality tests and monitors from YAML configuration files.

```bash
synqcli deploy [FILES...] [flags]
```

#### How It Works

1. **File Discovery** - If no files specified, discovers all `.yaml` files in current directory
2. **Parse** - Parses YAML files and converts to API format
3. **Resolve** - Resolves entity paths using SYNQ path resolution
4. **Preview** - Shows configuration changes and delta (creates, updates, deletes)
5. **Confirm** - Asks for confirmation (unless `--auto-confirm` is used)
6. **Deploy** - Applies the configuration changes

#### Examples

```bash
# Deploy specific files
synqcli deploy tests.yaml monitors.yaml

# Deploy all YAML files in current directory
synqcli deploy

# Deploy all YAML files recursively
synqcli deploy **/*.yaml

# Preview changes without deploying (dry run)
synqcli deploy --dry-run

# Deploy with auto-confirmation (for CI/CD)
synqcli deploy --auto-confirm

# Deploy only specific namespaces
synqcli deploy --namespace=data-team-pipeline

# Deploy with debug output
synqcli deploy -p  # prints protobuf messages in JSON format
```

#### Flags

| Flag | Short | Description |
|------|-------|-------------|
| `--auto-confirm` | | Skip confirmation prompts |
| `--dry-run` | | Preview changes without deploying |
| `--namespace` | | Only deploy changes for specified namespace |
| `--client-id` | | SYNQ client ID |
| `--client-secret` | | SYNQ client secret |
| `--api-url` | | SYNQ API URL |
| `--print-protobuf` | `-p` | Print protobuf messages in JSON format |

---

### Advisor

Get AI-powered suggestions for data quality tests based on your table schema.

```bash
synqcli advisor [flags]
```

#### How It Works

1. **Fetch Context** - Retrieves table schema, existing checks, and code from SYNQ
2. **Analyze** - AI analyzes the schema and generates appropriate test suggestions
3. **Output** - Returns JSON (default) or writes YAML files to specified directory
4. **Deploy** - Optionally deploys generated tests immediately

#### Examples

```bash
# Get suggestions for a single entity (outputs JSON)
synqcli advisor \
  --entity-id "postgres::public::users" \
  --instructions "Suggest basic data quality tests"

# Generate YAML files for multiple entities
synqcli advisor \
  --entity-id "postgres::public::users" \
  --entity-id "postgres::public::orders" \
  --entity-id "postgres::public::products" \
  --instructions "Suggest comprehensive tests for e-commerce tables" \
  --output ./generated-tests

# Use instructions from a file
synqcli advisor \
  --entity-id "snowflake::analytics::customers" \
  --instructions-file ./test-instructions.txt \
  --output ./tests

# Generate and deploy in one step
synqcli advisor \
  --entity-id "bigquery::dataset::events" \
  --instructions "Suggest freshness and volume monitors" \
  --output ./tests \
  --deploy \
  --auto-confirm

# Customize namespace and severity
synqcli advisor \
  --entity-id "postgres::public::transactions" \
  --instructions "Suggest tests for financial data" \
  --output ./tests \
  --namespace "finance-team" \
  --severity "ERROR"

# Force overwrite existing files
synqcli advisor \
  --entity-id "postgres::public::users" \
  --instructions "Suggest tests" \
  --output ./tests \
  --force
```

#### Flags

| Flag | Short | Description |
|------|-------|-------------|
| `--entity-id` | `-e` | Entity ID (table FQN) to suggest tests for (can be repeated) |
| `--instructions` | `-i` | Instructions for what tests to suggest |
| `--instructions-file` | `-I` | Path to file containing instructions |
| `--output` | `-o` | Output directory for generated YAML files |
| `--namespace` | `-n` | Namespace for generated YAML (default: synq-advisor) |
| `--severity` | `-s` | Default severity for tests/monitors (INFO, WARNING, ERROR) |
| `--force` | `-f` | Overwrite existing YAML files |
| `--deploy` | | Deploy generated files after creation |
| `--auto-confirm` | | Skip confirmation prompts during deployment |

---

### Export

Export existing monitors to YAML format.

```bash
synqcli export [flags] <output-file>
```

#### Examples

```bash
# Export all app-created monitors
synqcli export --namespace=exported-monitors output.yaml

# Export monitors for a specific table
synqcli export \
  --namespace=orders-monitors \
  --monitored="bq-prod.dataset.orders" \
  output.yaml

# Export monitors from multiple tables
synqcli export \
  --namespace=sales-monitors \
  --monitored="bq-prod.dataset.orders" \
  --monitored="bq-prod.dataset.customers" \
  output.yaml

# Export all monitors (including API-created)
synqcli export --namespace=all-monitors --source=all output.yaml

# Export monitors from a specific integration
synqcli export \
  --namespace=dbt-monitors \
  --integration="dbt-cloud-prod" \
  output.yaml
```

#### Flags

| Flag | Description |
|------|-------------|
| `--namespace` | Namespace for the exported config (required) |
| `--monitored` | Filter by monitored asset path (can be repeated) |
| `--integration` | Filter by integration ID |
| `--monitor` | Filter by monitor ID |
| `--source` | Filter by source: `app`, `api`, or `all` (default: `app`) |

---

## YAML Configuration Format

SYNQ CLI uses v1beta2 YAML format for defining tests and monitors.

### Basic Structure

```yaml
version: v1beta2
namespace: my-project

defaults:
  severity: WARNING

entities:
  - id: postgres::public::users
    tests:
      - type: not_null
        columns: [user_id, email]
    monitors:
      - type: automated
        metrics: [ROW_COUNT, DELAY]
```

### Complete Example

```yaml
# yaml-language-server: $schema=https://raw.githubusercontent.com/getsynq/synqcli/main/schema.json
version: v1beta2
namespace: data-team-pipeline

defaults:
  severity: ERROR
  schedule:
    type: daily
    query_delay: 2h
  mode:
    anomaly_engine:
      sensitivity: BALANCED

entities:
  - id: bq-prod.dataset.orders
    time_partitioning_column: created_at

    tests:
      # Ensure critical columns are never null
      - type: not_null
        columns:
          - order_id
          - customer_id
          - total_amount

      # Ensure order_id is unique
      - type: unique
        columns: [order_id]

      # Validate status values
      - type: accepted_values
        column: status
        values: [pending, processing, shipped, delivered, cancelled]

      # Ensure amounts are positive
      - type: min_value
        column: total_amount
        min_value: 0

      # Business rule: ship_date must be after order_date
      - type: relative_time
        column: ship_date
        relative_column: order_date

    monitors:
      # Automated monitoring for volume, freshness, and delays
      - type: automated
        metrics: [ROW_COUNT, DELAY, VOLUME_CHANGE_DELAY]
        severity: ERROR
        sensitivity: BALANCED

      # Volume monitoring segmented by region
      - id: orders_by_region
        type: volume
        segmentation:
          expression: region
        filter: "region IN ('US', 'EU', 'APAC')"

      # Field statistics monitoring
      - id: order_stats
        type: field_stats
        columns:
          - total_amount
          - discount_amount

  - id: bq-prod.dataset.customers
    tests:
      - type: not_null
        columns: [customer_id, email]

      - type: unique
        columns: [email]

      - type: business_rule
        sql_expression: "created_at <= updated_at"
```

### Schema Reference

Reference the JSON schema in your YAML files for IDE autocompletion and validation:

```yaml
# yaml-language-server: $schema=https://raw.githubusercontent.com/getsynq/synqcli/main/schema.json
version: v1beta2
```

Generate a local schema file:

```bash
synqcli schema > schema.json
```

---

## SQL Tests Reference

SQL tests are data quality validation rules that run SQL queries to check your data.

### Test Types

#### not_null

Ensures specified columns do not contain null values.

```yaml
- type: not_null
  columns:
    - user_id
    - email
    - created_at
```

#### empty

Ensures specified columns are not empty strings.

```yaml
- type: empty
  columns:
    - description
    - notes
```

#### unique

Ensures column values are unique, optionally within a time window.

```yaml
# Simple unique check
- type: unique
  columns: [order_id]

# Composite unique key
- type: unique
  columns:
    - customer_id
    - order_date

# Unique within time window (e.g., last 30 days)
- type: unique
  columns: [transaction_id]
  time_partition_column: created_at
  time_window_seconds: 2592000  # 30 days
```

#### accepted_values

Ensures column values are within a predefined list of acceptable values.

```yaml
# String values
- type: accepted_values
  column: status
  values:
    - active
    - inactive
    - pending

# Numeric values
- type: accepted_values
  column: priority
  values: [1, 2, 3, 4, 5]
```

#### rejected_values

Ensures column values are NOT in a predefined list of blocked values.

```yaml
- type: rejected_values
  column: error_code
  values:
    - -1
    - 0
    - 999
```

#### min_value

Ensures column values are greater than or equal to a minimum value.

```yaml
# Numeric minimum
- type: min_value
  column: age
  min_value: 18

# Strict comparison (greater than, not equal)
- type: min_value
  column: quantity
  min_value: 0
  strictly: true

# Date minimum
- type: min_value
  column: start_date
  min_value: "2024-01-01"
```

#### max_value

Ensures column values are less than or equal to a maximum value.

```yaml
# Numeric maximum
- type: max_value
  column: price
  max_value: 1000.99

# Use SQL expression (e.g., no future dates)
- type: max_value
  column: created_at
  max_value:
    type: expression
    value: NOW()
  strictly: true
```

#### min_max

Ensures column values fall within a specified range.

```yaml
# Numeric range
- type: min_max
  column: percentage
  min_value: 0
  max_value: 100

# Date range
- type: min_max
  column: event_date
  min_value: "2024-01-01"
  max_value: "2024-12-31"

# Temperature range
- type: min_max
  column: temperature
  min_value: -40
  max_value: 120
```

#### freshness

Ensures data is updated within a specified time window.

```yaml
- type: freshness
  time_partition_column: updated_at
  time_window_seconds: 7200  # 2 hours
```

#### relative_time

Ensures temporal relationships between columns (e.g., end_date >= start_date).

```yaml
- type: relative_time
  column: ship_date
  relative_column: order_date

- type: relative_time
  column: end_time
  relative_column: start_time
```

#### business_rule

Validates custom SQL expressions that represent business logic. The expression should return TRUE for **invalid** rows.

```yaml
# Accounting equation must balance
- type: business_rule
  sql_expression: "assets = liabilities + equity"

# Discount cannot exceed total
- type: business_rule
  sql_expression: "discount_amount <= total_amount"

# Complex validation
- type: business_rule
  sql_expression: "status = 'shipped' AND ship_date IS NOT NULL OR status != 'shipped'"
```

---

## Monitors Reference

Monitors continuously track metrics and detect anomalies in your data.

### Monitor Types

#### automated

The simplest way to monitor table health. Tracks volume, freshness, and change delays automatically.

```yaml
- type: automated
  severity: ERROR
  sensitivity: BALANCED
  metrics:
    - ROW_COUNT          # Monitor row count changes
    - DELAY              # Monitor data freshness
    - VOLUME_CHANGE_DELAY # Monitor when data typically changes
```

#### volume

Monitors row count with optional segmentation and filtering.

```yaml
# Basic volume monitoring
- type: volume

# Volume with segmentation (creates separate time series per segment)
- id: orders_by_region
  type: volume
  segmentation:
    expression: region
    include_values:
      - US
      - EU

# Volume with filter
- id: high_value_orders
  type: volume
  filter: "total_amount > 1000"
```

#### freshness

Monitors data freshness based on a timestamp column.

```yaml
- id: orders_freshness
  type: freshness
  expression: created_at
```

#### field_stats

Monitors column-level statistics including null rates, distinct values, and min/max values.

```yaml
- id: customer_stats
  type: field_stats
  columns:
    - email
    - status
    - created_at
  mode:
    anomaly_engine:
      sensitivity: BALANCED
```

#### custom_numeric

Monitors custom SQL aggregations.

```yaml
# Monitor active user count
- id: active_users
  type: custom_numeric
  metric_aggregation: "COUNT(DISTINCT user_id)"
  mode:
    fixed_thresholds:
      min: 100
      max: 100000

# Monitor average order value
- id: avg_order_value
  type: custom_numeric
  metric_aggregation: "AVG(total_amount)"
  mode:
    anomaly_engine:
      sensitivity: HIGH

# Monitor with segmentation
- id: revenue_by_country
  type: custom_numeric
  metric_aggregation: "SUM(revenue)"
  segmentation:
    expression: country
```

### Monitor Options

#### Severity

```yaml
severity: INFO | WARNING | ERROR
```

#### Schedule

```yaml
# Daily schedule
schedule:
  type: daily
  query_delay: 2h  # Wait 2 hours after midnight before running

# Hourly schedule
schedule:
  type: hourly
  query_delay: 15m
```

#### Mode

```yaml
# Anomaly detection
mode:
  anomaly_engine:
    sensitivity: LOW | BALANCED | HIGH

# Fixed thresholds
mode:
  fixed_thresholds:
    min: 0
    max: 1000
```

---

## Test Lifecycle

### UUID Generation

Tests and monitors use deterministic UUID generation based on their configuration:

- **Same configuration = Same UUID**: Redeploying with identical configuration updates the existing test
- **Changed configuration = New UUID**: Changing critical fields creates a new test

### Test Reset Behavior

Tests are reset (re-triggered) when these fields change:

- Schedule/recurrence
- Severity
- Test type
- Template configuration (columns, values, expressions, etc.)

Tests are NOT reset when only metadata changes (name, description).

---

## Production Deployment

The recommended workflow for production environments separates test generation from deployment:

1. **Local Development**: Use `advisor` to generate YAML files locally
2. **Code Review**: Commit and review generated files via pull request
3. **Automated Deployment**: CI/CD automatically deploys on merge to main

### Recommended Project Structure

```
my-data-project/
├── data-quality/
│   ├── orders.yaml           # Tests for orders table
│   ├── customers.yaml        # Tests for customers table
│   └── products.yaml         # Tests for products table
├── .github/
│   └── workflows/
│       └── deploy-data-quality.yml
└── README.md
```

### Local Workflow

```bash
# 1. Generate tests using advisor
synqcli advisor \
  --entity-id "bq-prod.dataset.orders" \
  --entity-id "bq-prod.dataset.customers" \
  --instructions "Suggest comprehensive data quality tests" \
  --output ./data-quality \
  --namespace "production-tests"

# 2. Review generated files
cat data-quality/*.yaml

# 3. Make any manual adjustments if needed
# Edit files as necessary

# 4. Commit and push
git add data-quality/
git commit -m "Add data quality tests for orders and customers"
git push origin feature/add-dq-tests

# 5. Create PR for review
# After approval and merge, CI/CD handles deployment
```

### GitHub Actions Workflow

Create `.github/workflows/deploy-data-quality.yml`:

```yaml
name: Deploy Data Quality Tests

on:
  push:
    branches: [main]
    paths:
      - 'data-quality/**/*.yaml'
  pull_request:
    branches: [main]
    paths:
      - 'data-quality/**/*.yaml'

jobs:
  validate:
    name: Validate Configuration
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install synqcli
        run: |
          curl -L https://github.com/getsynq/synqcli/releases/latest/download/synqcli_linux_amd64.tar.gz | tar -xz
          sudo mv synqcli /usr/local/bin/

      - name: Validate YAML files
        env:
          SYNQ_CLIENT_ID: ${{ secrets.SYNQ_CLIENT_ID }}
          SYNQ_CLIENT_SECRET: ${{ secrets.SYNQ_CLIENT_SECRET }}
          SYNQ_API_URL: https://developer.synq.io
        run: |
          synqcli deploy data-quality/**/*.yaml --dry-run

  deploy:
    name: Deploy to SYNQ
    runs-on: ubuntu-latest
    needs: validate
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    steps:
      - uses: actions/checkout@v4

      - name: Install synqcli
        run: |
          curl -L https://github.com/getsynq/synqcli/releases/latest/download/synqcli_linux_amd64.tar.gz | tar -xz
          sudo mv synqcli /usr/local/bin/

      - name: Deploy tests and monitors
        env:
          SYNQ_CLIENT_ID: ${{ secrets.SYNQ_CLIENT_ID }}
          SYNQ_CLIENT_SECRET: ${{ secrets.SYNQ_CLIENT_SECRET }}
          SYNQ_API_URL: https://developer.synq.io
        run: |
          synqcli deploy data-quality/**/*.yaml --auto-confirm
```

This workflow:
- **On Pull Request**: Validates YAML files with `--dry-run` (no actual deployment)
- **On Merge to Main**: Deploys tests and monitors to SYNQ

### GitLab CI

Create `.gitlab-ci.yml`:

```yaml
stages:
  - validate
  - deploy

validate-data-quality:
  stage: validate
  image: alpine:latest
  script:
    - apk add --no-cache curl
    - curl -L https://github.com/getsynq/synqcli/releases/latest/download/synqcli_linux_amd64.tar.gz | tar -xz
    - mv synqcli /usr/local/bin/
    - synqcli deploy data-quality/**/*.yaml --dry-run
  variables:
    SYNQ_CLIENT_ID: $SYNQ_CLIENT_ID
    SYNQ_CLIENT_SECRET: $SYNQ_CLIENT_SECRET
    SYNQ_API_URL: https://developer.synq.io
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      changes:
        - data-quality/**/*.yaml

deploy-data-quality:
  stage: deploy
  image: alpine:latest
  script:
    - apk add --no-cache curl
    - curl -L https://github.com/getsynq/synqcli/releases/latest/download/synqcli_linux_amd64.tar.gz | tar -xz
    - mv synqcli /usr/local/bin/
    - synqcli deploy data-quality/**/*.yaml --auto-confirm
  variables:
    SYNQ_CLIENT_ID: $SYNQ_CLIENT_ID
    SYNQ_CLIENT_SECRET: $SYNQ_CLIENT_SECRET
    SYNQ_API_URL: https://developer.synq.io
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      changes:
        - data-quality/**/*.yaml
```

### Required Secrets

Add these secrets to your CI/CD environment:

| Secret | Description |
|--------|-------------|
| `SYNQ_CLIENT_ID` | Your SYNQ API client ID |
| `SYNQ_CLIENT_SECRET` | Your SYNQ API client secret |

For GitHub: Settings → Secrets and variables → Actions → New repository secret

For GitLab: Settings → CI/CD → Variables

---

## Troubleshooting

### Common Issues

**Authentication errors:**
```
Error: failed to connect to SYNQ API: authentication failed
```
- Verify `SYNQ_CLIENT_ID` and `SYNQ_CLIENT_SECRET` are correct
- Check you're using the correct `SYNQ_API_URL` for your region

**Entity not found:**
```
Error: failed to resolve entity: postgres::public::users
```
- Verify the entity ID matches exactly what's shown in SYNQ
- Check the entity exists and is synced to SYNQ

**Invalid YAML:**
```
Error: failed to parse YAML: ...
```
- Validate your YAML syntax
- Reference the JSON schema for field names and types

### Debug Mode

Use `-p` flag to print detailed protobuf messages:

```bash
synqcli deploy tests.yaml -p
```

---

## Support

- **Documentation**: [docs.synq.io](https://docs.synq.io)
- **Issues**: [github.com/getsynq/synqcli/issues](https://github.com/getsynq/synqcli/issues)
- **Email**: support@synq.io
