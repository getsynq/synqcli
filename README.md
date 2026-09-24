# Coalesce Quality CLI (`synqcli`)

Command-line tool for managing data quality tests, monitors and deployment rules on
[Coalesce Quality](https://synq.io).

**`AGENTS.md`, shipped beside the binary, is the operating guide** — the deploy
loop, what a reconcile deletes, how to confirm which workspace you are pointed at,
and what each command costs. It is written for a coding agent driving the tool, and
is also published at
[docs.synq.io/monitors/agent-workflow](https://docs.synq.io/monitors/agent-workflow).
This README covers what the tool is and how a config is structured.

## Features

- **Deploy** - Deploy data quality tests, monitors and deployment rules from YAML configuration files
- **Advisor** - Get AI-powered suggestions for data quality tests based on your schema
- **Export** - Export existing monitors, tests and deployment rules to YAML format

Reference documentation:

- [Every command and flag](https://docs.synq.io/monitors/cli), generated from this CLI
- [The YAML configuration format](https://schemas.synq.io/synq-monitors/v1/config.schema.json), field by field ([rendered](https://schemas.synq.io/synq-monitors/v1/config.html))
- [Defining monitors in code](https://docs.synq.io/monitors/monitors-as-code) and [SQL tests](https://docs.synq.io/monitors/sql-tests) on the documentation site
- [`README_SQL_TESTS.md`](README_SQL_TESTS.md) for the SQL-test YAML in depth

## Installation

### macOS — Homebrew

```bash
brew install getsynq/tap/synqcli
```

`brew upgrade synqcli` from then on. Homebrew owns the binary once it installs it,
so `synqcli upgrade` will point you back here rather than replace it.

### macOS and Linux — release archive

The archive filename carries the version, so the version has to be resolved
first. GitHub redirects `/releases/latest` to the newest release's tag, which
needs no API token and no login:

```bash
VERSION=$(curl -fsSLI -o /dev/null -w '%{url_effective}' \
  https://github.com/getsynq/synqcli/releases/latest | sed 's#.*/v##')
OS=$(uname -s | tr '[:upper:]' '[:lower:]')          # darwin or linux
ARCH=$(uname -m | sed 's/x86_64/amd64/; s/aarch64/arm64/')

curl -fL "https://github.com/getsynq/synqcli/releases/download/v${VERSION}/synqcli_${VERSION}_${OS}_${ARCH}.tar.gz" \
  | tar -xz
sudo mv synqcli /usr/local/bin/
```

To pin a version instead, set `VERSION` by hand from the
[releases page](https://github.com/getsynq/synqcli/releases).

Builds are published for macOS and Linux on both amd64 and arm64, and every
release ships a `checksums.txt` (`sha256sum -c checksums.txt --ignore-missing`).

### Windows

Download `synqcli_<version>_windows_amd64.zip` (or `_arm64`) from the
[releases page](https://github.com/getsynq/synqcli/releases) and extract
`synqcli.exe` to your PATH.

### Upgrading

```bash
synqcli upgrade --check      # what it would do, without doing it
synqcli upgrade
```

`upgrade` resolves the latest release, downloads the archive for this platform,
verifies it against the release's `checksums.txt`, and runs the new binary once to
prove it works on this machine before replacing anything. If the binary lives
somewhere you cannot write — `/usr/local/bin` usually is not — it says so and
changes nothing; re-run it with `sudo`. A binary installed by a package manager is
left to that package manager — a Homebrew install is upgraded with
`brew upgrade synqcli`, which `upgrade` will tell you.

`synqcli` also mentions a newer release on stderr, at most once a day. That check
reads a tag from a public GitHub URL and sends nothing but the tool name and
version — no credentials, no workspace, no identity. It never delays the command it
runs beside and never reports its own failure, so a machine with no route to the
internet behaves exactly like one that is up to date. It is already silent in CI,
when output is not a terminal, and inside a container or a Kubernetes pod. To
switch it off everywhere:

```bash
export QUALITY_NO_UPDATE_CHECK=1     # DO_NOT_TRACK=1 has the same effect
```

## Configuration

### API Credentials

For interactive use, log in through the browser once:

```bash
synqcli auth login             # EU, the default deployment
synqcli auth login --region us # or au
synqcli auth status
```

The credential is cached under `~/.synq/oauth/` and shared with the other Coalesce
Quality CLIs, so later commands need no flag — the login records which deployment it
authenticated against.

For CI, set client credentials via environment variables:

```bash
export QUALITY_CLIENT_ID="your-client-id"
export QUALITY_CLIENT_SECRET="your-client-secret"
export QUALITY_REGION="eu"     # eu (default), us, or au
```

Or create a `.env` file in your project root:

```bash
QUALITY_CLIENT_ID=your-client-id
QUALITY_CLIENT_SECRET=your-client-secret
QUALITY_REGION=eu
```

Or use command-line flags (highest priority):

```bash
synqcli deploy --client-id="your-id" --client-secret="your-secret" --region=eu
```

Pass `--endpoint` (or `QUALITY_API_ENDPOINT`) instead of `--region` to reach a staging
or self-hosted deployment.

**Priority order:** client credentials (flags > environment variables > `.env`) > a
pre-issued `QUALITY_TOKEN` > the cached browser login. The CI paths deliberately win,
so adding a browser login cannot change what an existing pipeline authenticates as.

### Advisor Credentials

For the `advisor` command, you need an OpenAI-compatible API key or AWS Bedrock credentials:

**OpenAI (default):**
```bash
export OPENAI_API_KEY="your-api-key"
```

**Custom endpoint (LiteLLM, Azure, etc.):**
```bash
export OPENAI_API_KEY="your-api-key"
export OPENAI_BASE_URL="https://your-endpoint.com/v1"
```

**AWS Bedrock (direct):**

To use Claude models hosted on AWS Bedrock directly:

```bash
# Set the Bedrock model ID (required for Bedrock)
export AWS_BEDROCK_MODEL_ID="anthropic.claude-sonnet-4-20250514-v1:0"

# Set the AWS region (optional, defaults to us-east-1)
export AWS_REGION="us-east-1"

# AWS credentials are loaded from the standard AWS credential chain:
# - Environment variables (AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_SESSION_TOKEN)
# - Shared credentials file (~/.aws/credentials)
# - IAM roles (when running on AWS infrastructure)
```

Example usage:
```bash
AWS_BEDROCK_MODEL_ID="anthropic.claude-sonnet-4-20250514-v1:0" \
AWS_REGION="us-east-1" \
  synqcli advisor \
  --entity-id "postgres::public::users" \
  --instructions "Suggest data quality tests"
```

Available Bedrock Claude models:
- `anthropic.claude-sonnet-4-20250514-v1:0` (Claude Sonnet 4)
- `anthropic.claude-3-5-sonnet-20241022-v2:0` (Claude 3.5 Sonnet v2)
- `anthropic.claude-3-5-sonnet-20240620-v1:0` (Claude 3.5 Sonnet)
- `anthropic.claude-3-haiku-20240307-v1:0` (Claude 3 Haiku)

### DWH Connection (Optional)

With a warehouse connection, the advisor profiles your data before suggesting tests, so it can propose real accepted values and real bounds instead of guesses. Pass a connections file with `--connections`.

The file uses the same `connections:` format as the Coalesce Quality DWH agent, and supports every warehouse the agent does: BigQuery, Snowflake, Databricks, Redshift, Postgres, MySQL, ClickHouse, Trino, SQL Server, Oracle, Microsoft Fabric and Athena. Every field of every warehouse is in the [configuration reference](https://schemas.synq.io/synq-dwh/v1/config.html).

```yaml
# connections.yaml
connections:
  my-postgres:
    postgres:
      host: localhost
      port: 5432
      database: mydb
      username: user
      password: ${PG_PASSWORD}
```

`${VAR}` is replaced with the environment variable of that name, so secrets can stay out of the file. A `*_file` field, such as `private_key_file`, is read relative to the connections file.

#### Snowflake

**Password:**
```yaml
connections:
  my-snowflake:
    snowflake:
      account: myaccount.us-east-1     # Account identifier, with the region if your account needs it
      warehouse: COMPUTE_WH
      role: ANALYST
      username: myuser
      password: ${SNOWFLAKE_PASSWORD}
      databases: ["PROD", "DEV"]       # Optional: limit to these databases
      use_get_ddl: true                # Optional: read view definitions with GET_DDL
```

**Key pair:**
```yaml
connections:
  my-snowflake-key:
    snowflake:
      account: myaccount.us-east-1
      warehouse: COMPUTE_WH
      role: ANALYST
      username: myuser
      private_key_file: ./rsa_key.p8
      private_key_passphrase: ${SNOWFLAKE_KEY_PASSPHRASE}   # Only if the key is encrypted
      databases: ["PROD"]
```

**SSO in the browser (`externalbrowser`):**

If your organization signs in to Snowflake through an identity provider (Okta, Microsoft Entra ID and so on), let the advisor open the browser:

```yaml
connections:
  my-snowflake-sso:
    snowflake:
      account: myaccount.us-east-1
      warehouse: COMPUTE_WH
      role: ANALYST
      username: myuser@company.com     # Your SSO username or email
      auth_type: externalbrowser
      databases: ["PROD"]
```

The first connection opens your default browser for the SSO sign-in. The token Snowflake returns is cached in your operating system's credential store (Keychain on macOS, Credential Manager on Windows; on Linux a file, which you have to opt in to), so later connections reuse it until it expires, after about four hours. Then the browser opens again.

This needs ID token caching enabled on the Snowflake account, and your identity provider set up in Snowflake:

```sql
ALTER ACCOUNT SET ALLOW_ID_TOKEN = TRUE;
```

```bash
synqcli advisor \
  --entity-id "snowflake::PROD::ANALYTICS::ORDERS" \
  --instructions "Suggest data quality tests" \
  --connections ./connections.yaml
```

#### The older formats

Earlier releases read a list of connections, and environment variables for a single one. Both still work, but only for BigQuery, Postgres, MySQL, Snowflake with a password, ClickHouse, Redshift and Databricks, and they won't learn new warehouses or options. Move to the `connections:` format above when you can.

```yaml
# connections.yaml, list format
- id: my-postgres
  type: postgres
  host: localhost
  port: 5432
  database: mydb
  username: user
  password: pass
```

```bash
# Environment variables, used when --connections is not given
export DWH_TYPE="postgres"           # postgres, mysql, bigquery, snowflake, clickhouse, redshift or databricks
export DWH_HOST="localhost"
export DWH_PORT="5432"
export DWH_DATABASE="mydb"
export DWH_USERNAME="user"
export DWH_PASSWORD="pass"
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
3. **Resolve** - Resolves the short entity ids in the YAML to full asset paths
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

`synqcli deploy --help`, or the [CLI reference](https://docs.synq.io/monitors/cli#synqcli-deploy).

---

### Advisor

Get AI-powered suggestions for data quality tests based on your table schema.

```bash
synqcli advisor [flags]
```

#### How It Works

1. **Fetch Context** - Retrieves table schema, existing checks, and code from Coalesce Quality
2. **Profile Data** (optional) - If DWH connection is configured, profiles columns to discover actual values, min/max bounds, and null rates
3. **Analyze** - AI analyzes the schema (and profiling results) to generate appropriate test suggestions
4. **Output** - Returns JSON (default) or writes YAML files to specified directory
5. **Deploy** - Optionally deploys generated tests immediately

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

# With DWH connection for data profiling (discovers actual values)
synqcli advisor \
  --entity-id "postgres::public::users" \
  --instructions "Suggest accepted_values tests for enum-like columns" \
  --connections ./connections.yaml \
  --output ./tests

# DWH connection via environment variables
DWH_TYPE=postgres DWH_HOST=localhost DWH_DATABASE=mydb \
  synqcli advisor \
  --entity-id "postgres::public::users" \
  --instructions "Suggest min/max tests based on actual data ranges"

# Verbose mode to see AI reasoning and tool calls
synqcli advisor \
  --entity-id "postgres::public::users" \
  --instructions "Suggest tests" \
  --connections ./connections.yaml \
  --verbose

# Filter suggestions to specific columns (comma-separated)
synqcli advisor \
  --entity-id "postgres::public::users" \
  --columns "status,email,role" \
  --instructions "Suggest accepted_values tests for these columns"

# Filter to specific columns (multiple flags)
synqcli advisor \
  --entity-id "postgres::public::orders" \
  --columns status \
  --columns priority \
  --columns region \
  --instructions "Suggest tests for these enum-like columns" \
  --output ./tests
```

#### Flags

`synqcli advisor --help`, or the [CLI reference](https://docs.synq.io/monitors/cli#synqcli-advisor).

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

#### Selective export by resource type

By default `export` writes every resource type (custom monitors, SQL tests,
deployment rules and SQL test deployment rules). Use `--type` (repeatable) to narrow,
or pass an ID-scoped flag and the type is inferred automatically.

```bash
# Only SQL tests
synqcli export --type=sql-tests generated/tests.yaml

# SQL tests + deployment rules, no custom monitors
synqcli export --type=sql-tests --type=deployment-rules generated/tests_and_rules.yaml

# One specific test (auto-narrows to --type=sql-tests)
synqcli export --sql-test=<test-uuid> generated/one_test.yaml

# All monitors plus one specific test (union — explicit --type widens, doesn't restrict)
synqcli export --type=monitors --sql-test=<test-uuid> generated/mix.yaml
```

The two kinds of deployment rule export differently, because each supports a
different kind of selection:

- **SQL test deployment rules** (`--type=sql-test-deployment-rules`) are
  query-based, and are written to the `deployment_rules` list as
  `type: sql_tests` entries.
- **Monitor deployment rules** (`--type=deployment-rules`) are written as
  single-asset `table_stats` monitors under the entity they cover. Their
  query-based form is authoring-only and is not exported.

`--monitored` and `--integration` narrow monitors and SQL tests. They do not
narrow a query-based rule: it carries no asset path of its own, and what its
selection matches is resolved server-side. An export that passes them therefore
still writes every SQL test deployment rule in the workspace, and says so. Use
`--sql-test-deployment-rule=<rule-uuid>` to pick specific rules, or leave the type
out with `--type`.

#### Flags

`synqcli export --help`, or the [CLI reference](https://docs.synq.io/monitors/cli#synqcli-export).

---

## YAML Configuration Format

`synqcli` uses the v1beta2 YAML format for defining tests and monitors.

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
        description: Ensure critical user identifiers are always present
        columns: [user_id, email]
    monitors:
      - type: table_stats
        metrics: [ROW_COUNT, DELAY]
```

### Complete Example

```yaml
# yaml-language-server: $schema=https://schemas.synq.io/synq-monitors/v1/config.schema.json
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
        description: Order ID, customer ID and total amount are required for all orders
        columns:
          - order_id
          - customer_id
          - total_amount

      # Ensure order_id is unique
      - type: unique
        description: Each order must have a unique identifier
        columns: [order_id]

      # Validate status values
      - type: accepted_values
        description: Order status must be one of the valid workflow states
        column: status
        values: [pending, processing, shipped, delivered, cancelled]

      # Ensure amounts are positive
      - type: min_value
        description: Order amounts cannot be negative
        column: total_amount
        min_value: 0

      # Business rule: ship_date must be after order_date
      - type: relative_time
        description: Ship date must be on or after order date
        column: ship_date
        relative_column: order_date

    monitors:
      # Table health: row count, freshness and change delays
      - type: table_stats
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
        description: Customer ID and email are required for all customers
        columns: [customer_id, email]

      - type: unique
        description: Email addresses must be unique across all customers
        columns: [email]

      - type: business_rule
        description: Updated timestamp must be on or after creation timestamp
        sql_expression: "created_at > updated_at"
```

### Schema Reference

Reference the JSON schema in your YAML files for IDE autocompletion and validation:

```yaml
# yaml-language-server: $schema=https://schemas.synq.io/synq-monitors/v1/config.schema.json
version: v1beta2
```

The published schema always describes the current release. To pin the schema to the
CLI version you deploy with, write it out and reference the local file instead:

```bash
synqcli schema > schema.json
```

```yaml
# yaml-language-server: $schema=./schema.json
version: v1beta2
```

A rendered, browsable version of the same schema is at
<https://schemas.synq.io/synq-monitors/v1/config.html>.

---

## Query-based deployment rules

A monitor or test under an `entities[].id` targets a single asset. To cover many assets
by a rule instead of listing each one, author
[query-based deployment rules](https://docs.synq.io/monitors/deployment-rules) at the
top level with a ResolverQL selection string — the same selection the app and the API
expose. New assets that match are covered automatically, with no YAML edit.

`type` says what the rule deploys: `table_stats` deploys monitors, `sql_tests` deploys
SQL tests. Both kinds live in the same `deployment_rules` list.

```yaml
# yaml-language-server: $schema=https://schemas.synq.io/synq-monitors/v1/config.schema.json
version: v1beta2
namespace: "data-team-pipeline"

# Inclusion: deploy monitors to every matching asset (full config).
deployment_rules:
  - name: snowflake tables row-count and delay
    type: table_stats
    resolver_ql: with_type("table", filter=with_platform("snowflake"))
    severity: ERROR
    sensitivity: RELAXED
    metrics:
      - ROW_COUNT
      - DELAY

# Exclusion: carve matching assets OUT of coverage. Only a selection — no
# metrics/severity/sensitivity (those describe how to monitor, not what to skip).
deployment_exclusions:
  - name: exclude staging tables
    type: table_stats
    resolver_ql: with_type("table", filter=with_tag("staging"))
```

Notes:

- `name` is required on every rule and exclusion, and it **is** the rule's identity
  within its namespace: editing the `resolver_ql`, the `metrics` or the `sensitivity`
  updates the rule in place, while renaming it replaces it. Iterate on the query and
  leave the `name` alone. A rule and an exclusion may share a name — they are two
  different things.
- `resolver_ql` is the only selection form supported here. The string is forwarded
  verbatim; the backend compiles and validates it, so an invalid query fails at deploy
  time. What is stored is the compiled selection, not the text, so reformatting a
  query or spelling it differently is not a change: when the credential can read
  entities, the deploy preview compares selections rather than strings. A type alias such as `"table"` is expanded to the types it
  names when the rule is deployed, so redeploy the rule to pick up a type added later.
- Query rules are authoring-only: `export` does not emit them (it writes single-asset
  `entities` rules). A full example is
  [`examples/v1beta2/query_deployment_rules.yaml`](examples/v1beta2/query_deployment_rules.yaml).
- The deploy preview (before confirm, and under `--dry-run`) shows each query rule's
  downstream effect — how many monitors it will create / delete / change, plus skipped
  assets. The asset lists are capped; pass `--verbose` to list every affected asset.

### Deploying SQL tests by rule

A `type: sql_tests` rule carries the tests to deploy onto every table or view its
selection matches, in the same shape as `entities[].tests[]`.

```yaml
deployment_rules:
  - name: PII email checks
    type: sql_tests
    resolver_ql: with_columns("email")
    schedule: daily              # omit to inherit `defaults.schedule`, else daily; `ondemand` deploys unscheduled
    timezone: Europe/London
    severity: ERROR
    save_failures: true
    # When a table stops matching, its deployed tests are deleted. Set true to keep them.
    keep_removed_tests: false
    tests:
      - type: not_null
        columns: [email]
      - type: business_rule
        name: email looks like an address
        sql_expression: email NOT LIKE '%@%'
      # Every matched table is checked against the same reference table: the
      # rule cannot name a different reference per match.
      - type: relationships
        references:
          - entity: ch-prod.default.customers
            columns:
              - source: customer_id
                reference: id
```

Notes:

- `schedule`, `timezone`, `severity` and `save_failures` are rule-level: they apply to
  every test the rule deploys. A test may override `severity`; the others cannot be set
  per test and are rejected there, because the deployed test has nowhere to carry them.
- `name` is required on `business_rule` and `business_query` tests. For a test a rule
  deploys, it is also that test's identity across redeployments, so renaming it is a
  different test — a test under `entities[].tests[]` carries an `id:` instead. For
  every other kind, leave it out and a name is generated from the test's kind and
  columns (`Unique on order_id`), the same as for a test under `entities[].tests[]`.
- `id`, `category`, `governance_category` and `business_query` `evaluators` are not
  supported on a *test* inside a rule (the rule's own `id` is separate), and are rejected rather than silently dropped.
  Author such a test under `entities[].tests[]` instead.
- Rules covering the same table merge additively: each deploys the tests that are not
  there yet and never touches a test another rule owns. The deploy preview names the
  tests it skipped and the rule that owns them.
- A `relationships` test in a rule names **one** reference table (per `references[]`
  entry), and every matched table is checked against that same table — the star shape,
  many tables carrying `customer_id` against one `dim.customers`. The reference is not
  resolved per match: a rule cannot express "each `staging.X` against its own `raw.X`".
  Write such tests under `entities[].tests[]`, one per table. The reference table must
  be in the same integration as the matched tables, since the test runs on the matched
  table's connection — nothing narrows the selection to that integration for you, so
  confine it yourself (`with_integration_ids(...)`). A match in another integration gets
  a test whose reference resolves on the wrong connection: it fails every run, or joins
  a table that happens to share the name.
- A test is deployed only onto the matched tables that have **every column it reads** —
  the columns it checks, any `select_columns`, a `unique` or `relationships` test's
  `time_partition_column`, and for `relationships` the columns on the referenced table
  too. The other tables appear in the deploy plan as skipped tests, and no test is
  created there: it could only fail every run against the warehouse. So a broad
  selection is safe to write — a rule carrying three tests covers each table with the
  ones that fit it.
  - A test is skipped whole. `not_null` on `[email, account_id]` where only `account_id`
    is missing is not narrowed to `email`: `unique(a, b)` is a different assertion from
    `unique(a)`, and a narrowed test would carry a different identity and a name that
    lies. Declare one test per column if you want that granularity.
  - A table whose schema is not known yet keeps the tests it already carries — they are
    left as they are and retried on the next sync rather than deleted. A test that is
    not deployed there yet waits instead of being created, since whether the columns are
    there cannot be answered either way. For a matched table that resolves once its
    schema is ingested; a `relationships` test's **referenced** table is named by hand
    and may never be ingested at all, in which case the test waits indefinitely and the
    deploy plan says so, naming the referenced table.
  - A column dropped from a table the rule still matches removes that table's test on
    the next sync, and `keep_removed_tests` does not protect it — that flag is about the
    table leaving the selection, not about the test losing what it reads. A table that
    has already left the selection is a different case: `keep_removed_tests` freezes its
    tests, the rule stops resyncing them altogether, and the column gate no longer
    reaches them either. Such a test is yours to remove if it starts failing.
  - A `business_rule` is gated on its `select_columns` alone: its SQL expression is
    never parsed, so a rule whose expression reads `status` without declaring it
    deploys anyway, while one that declares `select_columns: [status]` is skipped on
    every table without that column. A `business_query` declares no columns of its
    own and is never gated.
- `id` is optional and is the rule's address when present: a name of your own works,
  and so does a UUID, which is used verbatim. `export` writes one, so deploying an
  exported config updates the rules it was exported from rather than creating copies
  of them. With an `id`, renaming the rule is an update.
- Without an `id`, a rule is identified by its `name` within its namespace — every
  query-based rule under `deployment_rules`, both types, and every
  `deployment_exclusions` entry — so renaming it replaces the rule.
- The exception is the other `table_stats` shape: a `table_stats` monitor under an
  entity covers that one asset and is addressed by it, so its `id` takes only the
  UUID `export` writes and a name of your own is rejected there. See
  ["How a check is identified"](#how-a-check-is-identified).
- Either way, editing the `resolver_ql` updates the rule and resyncs it: tests appear
  on tables the selection now matches, go away from tables it no longer matches, and
  tables matching both before and after keep the tests they already had. Replacing a
  rule is the expensive one: **the old rule is deleted with every test it deployed**,
  with their check entities and their history, and the next scheduled sync builds them
  again from scratch — so give a rule an `id` before you need to rename it. Deleting a
  rule outright is not the same: nothing declares it any more, so nothing brings its
  tests back.
- A full example is
  [`examples/v1beta2/sql_test_deployment_rules.yaml`](examples/v1beta2/sql_test_deployment_rules.yaml).

---

## SQL Tests Reference

SQL tests are data quality validation rules that run SQL queries to check your data.
[`README_SQL_TESTS.md`](README_SQL_TESTS.md) covers the parts this summary leaves out:
`business_query` evaluators, `save_failures`, and exactly which edits reset a test.

### Test Types

#### not_null

Ensures specified columns do not contain null values.

```yaml
- type: not_null
  description: Critical user fields must always have values
  columns:
    - user_id
    - email
    - created_at
```

#### empty

Ensures column values are not empty — a null or whitespace-only value counts as empty.

```yaml
- type: empty
  description: Description and notes should contain meaningful content when present
  columns:
    - description
    - notes
```

#### unique

Ensures column values are unique, optionally within a time window.

```yaml
# Simple unique check
- type: unique
  description: Order ID must be unique across all orders
  columns: [order_id]

# Composite unique key
- type: unique
  description: Customer can only have one order per day
  columns:
    - customer_id
    - order_date

# Unique within time window (e.g., last 30 days)
- type: unique
  description: Transaction IDs must be unique within rolling 30-day window
  columns: [transaction_id]
  time_partition_column: created_at
  time_window_seconds: 2592000  # 30 days
```

#### accepted_values

Ensures column values are within a predefined list of acceptable values.

```yaml
# String values
- type: accepted_values
  description: Account status must be a valid lifecycle state
  column: status
  values:
    - active
    - inactive
    - pending

# Numeric values
- type: accepted_values
  description: Priority must be between 1 (highest) and 5 (lowest)
  column: priority
  values: [1, 2, 3, 4, 5]
```

#### rejected_values

Ensures column values are NOT in a predefined list of blocked values.

```yaml
- type: rejected_values
  description: Error codes must not contain placeholder or invalid values
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
  description: Users must be at least 18 years old
  column: age
  min_value: 18

# Strict comparison (greater than, not equal)
- type: min_value
  description: Quantity must be positive (greater than zero)
  column: quantity
  min_value: 0
  strictly: true

# Date minimum
- type: min_value
  description: Start date must be in 2024 or later
  column: start_date
  min_value: "2024-01-01"
```

#### max_value

Ensures column values are less than or equal to a maximum value.

```yaml
# Numeric maximum
- type: max_value
  description: Product price cannot exceed maximum allowed price
  column: price
  max_value: 1000.99

# Use SQL expression (e.g., no future dates)
- type: max_value
  description: Created timestamp cannot be in the future
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
  description: Percentage values must be between 0 and 100
  column: percentage
  min_value: 0
  max_value: 100

# Date range
- type: min_max
  description: Event dates must fall within the 2024 calendar year
  column: event_date
  min_value: "2024-01-01"
  max_value: "2024-12-31"

# Temperature range
- type: min_max
  description: Temperature readings must be within valid sensor range
  column: temperature
  min_value: -40
  max_value: 120
```

#### freshness

Ensures data is updated within a specified time window.

```yaml
- type: freshness
  description: Table should be updated at least every 2 hours
  time_partition_column: updated_at
  time_window_seconds: 7200  # 2 hours
```

#### relative_time

Ensures temporal relationships between columns (e.g., end_date >= start_date).

```yaml
- type: relative_time
  description: Ship date must be on or after order date
  column: ship_date
  relative_column: order_date

- type: relative_time
  description: End time must be after start time
  column: end_time
  relative_column: start_time
```

#### business_rule

Validates custom SQL expressions that represent business logic. The expression should return TRUE for **invalid** rows.

```yaml
# Accounting equation must balance
- type: business_rule
  description: Assets must equal liabilities plus equity (accounting equation)
  sql_expression: "assets != liabilities + equity"

# Discount cannot exceed total
- type: business_rule
  description: Discount amount cannot exceed order total
  sql_expression: "discount_amount > total_amount"

# Complex validation
- type: business_rule
  description: Shipped orders must have a ship date
  sql_expression: "status = 'shipped' AND ship_date IS NULL"
```

---

## Monitors Reference

Monitors continuously track metrics and detect anomalies in your data.

### Monitor Types

#### table_stats

The simplest way to monitor table health. Tracks row count, freshness and change
delays from the table metadata the warehouse already keeps, so it runs no query
over the table itself. Under an entity it covers that one asset — `sensitivity`
applies to every metric, and a metric may override it.

**It monitors tables, not views.** A view has no such metadata behind it, and
the deploy does not refuse one: the rule is created and the monitor then has
nothing to read. For what to use on a view instead, see
[`custom_table_stats`](#custom_table_stats) below.

```yaml
- type: table_stats
  severity: ERROR
  sensitivity: BALANCED
  metrics:
    - ROW_COUNT          # Monitor row count changes
    - DELAY              # Monitor data freshness
    - VOLUME_CHANGE_DELAY # Monitor when data typically changes
```

There is one `table_stats` entry per entity — a second is rejected. To vary
sensitivity across the metrics, set it per metric instead of on the monitor:

```yaml
- type: table_stats
  severity: ERROR
  metrics:
    - metric: ROW_COUNT
      sensitivity: BALANCED
    - metric: DELAY
      sensitivity: PRECISE
```

Being a rule, it takes no `id:` of your own — see
["How a check is identified"](#how-a-check-is-identified) — and no `mode`,
`schedule`, `filter` or `segmentation`. To cover many assets with one rule
instead of listing each, write a
[query-based deployment rule](#query-based-deployment-rules); for the same
metrics as an ordinary monitor, use `custom_table_stats` below.

#### custom_table_stats

The same three metrics as a `table_stats` rule, but authored as an ordinary
monitor: it takes an `id:` of your own, a `name`, a `schedule`, a `mode`, a
`timezone`, and there may be several on one entity.

```yaml
# All three metrics, monitor-level sensitivity
- id: orders_table_stats
  type: custom_table_stats
  schedule: hourly
  metrics: [ROW_COUNT, DELAY, VOLUME_CHANGE_DELAY]
  sensitivity: BALANCED

# Freshness read from a column you name, rather than from table metadata
- id: orders_freshness_from_column
  type: custom_table_stats
  metrics:
    - DELAY
    - metric: ROW_COUNT
      sensitivity: PRECISE
  freshness_source:
    field: updated_at
```

Leave `metrics` out and every metric is monitored.

`freshness_source` says where "last loaded" comes from: `field:` names a
timestamp column, `sql:` is an expression wrapped in `MAX(…)`
(`COALESCE(updated_at, created_at)`). Without it the metric comes from table
metadata, as it does for a `table_stats` rule.

**Which of the two to use.** A `table_stats` rule is the cheaper one and covers
tables; prefer it, and reach for `custom_table_stats` when you need a schedule,
a mode, several table-stats monitors on one asset, or a freshness source of your
own.

**On a view, `freshness_source` is what makes any of these metrics real.** It
is not only a freshness setting: it is what moves the whole monitor off the
table metadata and onto a query against the asset, and `ROW_COUNT` rides along
with it. Without one, a `custom_table_stats` monitor reads the same metadata a
`table_stats` rule does, which a view does not have. The deploy enforces this
for `DELAY` and `VOLUME_CHANGE_DELAY` only — they are rejected on a view with
no `freshness_source` — so `metrics: [ROW_COUNT]` on a view is accepted and
then reports whatever the warehouse says about a view it never counted, with no
error to go on.

So on a view: `custom_table_stats` with a `freshness_source` for `DELAY` or
`VOLUME_CHANGE_DELAY`, and [`volume`](#volume) when row count is all you want —
a `volume` monitor with no `expression` queries the asset for one whole-table
count per run and needs no timestamp column to do it.

Full examples:
[`examples/v1beta2/custom_table_stats_monitor.yaml`](examples/v1beta2/custom_table_stats_monitor.yaml)
and
[`examples/v1beta2/freshness_source.yaml`](examples/v1beta2/freshness_source.yaml).

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

#### Time segmentation

`time_partitioning_column` splits the asset into time segments (one data point per day or
hour) and is set on the monitor, the entity, or in `defaults`:

```yaml
entities:
  - id: bq-prod.dataset.orders
    time_partitioning_column: created_at
    monitors:
      - type: volume
        id: orders_volume
```

Omit it to monitor the asset **without time segmentation** — the metric is computed over the
whole asset once per run, and there is no historical backfill on the first run:

```yaml
entities:
  - id: bq-prod.dataset.reference_data
    monitors:
      - type: volume
        id: reference_data_row_count
```

`time_partitioning_interval` (ondemand schedules) requires a `time_partitioning_column`.

Without time segmentation there are no segments to skip, so `ignore_last` has no effect.

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

#### Categories (`category`, `governance_category`)

A monitor can declare its own categories. `category` is the **technical** dimension — what kind of check this is mechanically, e.g. `volume` or `freshness`. `governance_category` is what the check is *for*, the data quality dimension governance reports on, e.g. `timeliness`. Both are free-form strings — use whatever vocabulary your categorisation rules already use — and they resolve independently, so a monitor may set either, both, or neither:

```yaml
entities:
  - id: bq-prod.dataset.orders
    time_partitioning_column: created_at
    monitors:
      - type: volume
        id: orders_volume
        category: volume
        governance_category: timeliness

      - type: freshness
        id: orders_freshness
        expression: created_at
        category: freshness
        # governance category left to the categorisation rules
```

What a monitor declares here **takes precedence over your workspace's categorisation rules** for that monitor. Leave a category out and the rules decide it as before; delete one from the file and the next deploy hands that dimension back to the rules. Values are shown with underscores as spaces, so `snake_case` reads well.

A segmented monitor's segments take the monitor's categories. A segment is the same monitor sliced by a column value, so it cannot declare categories of its own.

There is deliberately no `defaults:` entry for either field. A default would categorise every monitor in the file, and because a declared category outranks the rules, that would switch the rules off for all of them rather than fill a gap.

Changing a category does not reset a monitor's learned baseline.

---

## Check Lifecycle

### How a check is identified

A check's id is derived, deterministically, so redeploying the same file updates
the checks instead of duplicating them. What the id is derived from is what
decides whether an edit updates a check or **replaces** it — and a replacement
deletes the old check, with its issue history and its error runs, and for a
monitor with the baseline it has learned.

**Write an `id:` of your own and it is the check's address.** The id is then
derived from the namespace, the entity, the check type and that `id:`, so every
other field is editable in place. A UUID in `id:` is used verbatim, which is what
`export` writes.

One check narrows that: a `table_stats` monitor under an entity is a rule on that
one asset, and there is one per asset, so the asset is its address. Its `id:`
accepts only the UUID `export` writes for it — which adopts that rule — and a name
of your own is rejected rather than reinterpreted.

**Omit `id:` and the check's content is its identity** — for a monitor its mode,
partitioning and segmentation plus, for a `custom_numeric`, its
`metric_aggregation` or, for a `category_distribution`, its column; for a test the
columns it names and the SQL of a `business_rule` or `business_query`. Editing one
of those fields then replaces the check.

Three edits replace a check whichever way it is identified: renaming its `id:`,
moving it to another entity, and changing its `type`.

`--dry-run` reports a renamed monitor as one line — `Replaced, id changed:` — and
a replaced test or rule as a delete plus a create.

The [agent guide](AGENTS.md) § 6 has the per-field tables, for monitors,
tests and deployment rules.

### Re-runs and resets

Some in-place updates disturb what a check already has, and the deploy plan flags
them.

**A test re-run discards nothing.** A test has no learned state; changing its
recurrence, severity, evaluators or template body re-arms it, so it runs
immediately rather than at its next slot. A test with no `schedule:` is not
re-armed at all.

**A monitor reset discards its learned baseline**, and it has no anomaly baseline
until it has collected enough history again. A monitor resets on a change to its
timezone, its mode, its schedule, its `metric_aggregation`, a
`category_distribution`'s column or `top_k_limit`, its time partitioning or its
segmentation.

Neither happens when only metadata changes (name, description, categories).

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
          # The archive filename carries the version, so resolve it first. Set
          # VERSION to a literal instead to pin the pipeline to a known release.
          VERSION=$(curl -fsSLI -o /dev/null -w '%{url_effective}' \
            https://github.com/getsynq/synqcli/releases/latest | sed 's#.*/v##')
          curl -fL "https://github.com/getsynq/synqcli/releases/download/v${VERSION}/synqcli_${VERSION}_linux_amd64.tar.gz" | tar -xz
          sudo mv synqcli /usr/local/bin/
          synqcli --version

      - name: Validate YAML files
        env:
          QUALITY_CLIENT_ID: ${{ secrets.QUALITY_CLIENT_ID }}
          QUALITY_CLIENT_SECRET: ${{ secrets.QUALITY_CLIENT_SECRET }}
          QUALITY_API_ENDPOINT: https://developer.synq.io
        run: |
          synqcli deploy data-quality/**/*.yaml --dry-run

  deploy:
    name: Deploy to Coalesce Quality
    runs-on: ubuntu-latest
    needs: validate
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    steps:
      - uses: actions/checkout@v4

      - name: Install synqcli
        run: |
          # The archive filename carries the version, so resolve it first. Set
          # VERSION to a literal instead to pin the pipeline to a known release.
          VERSION=$(curl -fsSLI -o /dev/null -w '%{url_effective}' \
            https://github.com/getsynq/synqcli/releases/latest | sed 's#.*/v##')
          curl -fL "https://github.com/getsynq/synqcli/releases/download/v${VERSION}/synqcli_${VERSION}_linux_amd64.tar.gz" | tar -xz
          sudo mv synqcli /usr/local/bin/
          synqcli --version

      - name: Deploy tests and monitors
        env:
          QUALITY_CLIENT_ID: ${{ secrets.QUALITY_CLIENT_ID }}
          QUALITY_CLIENT_SECRET: ${{ secrets.QUALITY_CLIENT_SECRET }}
          QUALITY_API_ENDPOINT: https://developer.synq.io
        run: |
          synqcli deploy data-quality/**/*.yaml --auto-confirm
```

This workflow:
- **On Pull Request**: Validates YAML files with `--dry-run` (no actual deployment)
- **On Merge to Main**: Deploys tests and monitors to Coalesce Quality

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
    # The archive filename carries the version, so resolve it first. Set VERSION
    # to a literal instead to pin the pipeline to a known release.
    - VERSION=$(curl -fsSLI -o /dev/null -w '%{url_effective}' https://github.com/getsynq/synqcli/releases/latest | sed 's#.*/v##')
    - curl -fL "https://github.com/getsynq/synqcli/releases/download/v${VERSION}/synqcli_${VERSION}_linux_amd64.tar.gz" | tar -xz
    - mv synqcli /usr/local/bin/
    - synqcli --version
    - synqcli deploy data-quality/**/*.yaml --dry-run
  variables:
    QUALITY_CLIENT_ID: $QUALITY_CLIENT_ID
    QUALITY_CLIENT_SECRET: $QUALITY_CLIENT_SECRET
    QUALITY_API_ENDPOINT: https://developer.synq.io
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      changes:
        - data-quality/**/*.yaml

deploy-data-quality:
  stage: deploy
  image: alpine:latest
  script:
    - apk add --no-cache curl
    # The archive filename carries the version, so resolve it first. Set VERSION
    # to a literal instead to pin the pipeline to a known release.
    - VERSION=$(curl -fsSLI -o /dev/null -w '%{url_effective}' https://github.com/getsynq/synqcli/releases/latest | sed 's#.*/v##')
    - curl -fL "https://github.com/getsynq/synqcli/releases/download/v${VERSION}/synqcli_${VERSION}_linux_amd64.tar.gz" | tar -xz
    - mv synqcli /usr/local/bin/
    - synqcli --version
    - synqcli deploy data-quality/**/*.yaml --auto-confirm
  variables:
    QUALITY_CLIENT_ID: $QUALITY_CLIENT_ID
    QUALITY_CLIENT_SECRET: $QUALITY_CLIENT_SECRET
    QUALITY_API_ENDPOINT: https://developer.synq.io
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      changes:
        - data-quality/**/*.yaml
```

### Required Secrets

Add these secrets to your CI/CD environment:

| Secret | Description |
|--------|-------------|
| `QUALITY_CLIENT_ID` | Your Coalesce Quality API client ID |
| `QUALITY_CLIENT_SECRET` | Your Coalesce Quality API client secret |

For GitHub: Settings → Secrets and variables → Actions → New repository secret

For GitLab: Settings → CI/CD → Variables

---

## Troubleshooting

### Common Issues

**Authentication errors:**
```
Error: failed to connect to Coalesce Quality API: authentication failed
```
- Verify `QUALITY_CLIENT_ID` and `QUALITY_CLIENT_SECRET` are correct
- Check you're using the correct `QUALITY_API_ENDPOINT` for your region

**Entity not found:**
```
Error: failed to resolve entity: postgres::public::users
```
- Verify the entity ID matches exactly what's shown in the app
- Check the entity exists and is synced to Coalesce Quality

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

Every Coalesce Quality customer has a shared Slack channel with a Technical Account
Manager. Ask there for anything — getting a configuration deployed, a platform you
want supported, or something that looks wrong.
[Support](https://docs.synq.io/support/support) has the details, and
[docs.synq.io](https://docs.synq.io) covers the rest of the platform.
