# SQL Tests

SQL tests are data quality validation rules that can be defined on entities in your YAML configuration. They enable you to ensure data integrity, consistency, and correctness across your data assets.

## Basics

SQL tests are defined under the `tests` section of an entity in your YAML configuration. Each test validates a specific aspect of your data, such as ensuring columns are not null, values are within acceptable ranges, or custom business rules are satisfied.

### Example Structure

```yaml
version: v1beta2
namespace: financial_data

entities:
  - id: ch-prod.default.financial_statement
    tests:
      - type: not_null
        columns:
          - total_revenue
          - total_tips
```

## Supported Test Types

The following test types are supported:

- **`not_null`** - Ensures specified columns do not contain null values
- **`empty`** - Ensures specified columns are empty
- **`unique`** - Ensures column values are unique (optionally within a time window)
- **`accepted_values`** - Ensures column values are within a predefined list of acceptable values
- **`rejected_values`** - Ensures column values are not in a predefined list of rejected values
- **`min_max`** - Ensures column values fall within a specified numeric range
- **`min_value`** - Ensures column values are greater than (or equal to) a minimum value
- **`max_value`** - Ensures column values are less than (or equal to) a maximum value
- **`freshness`** - Ensures data is updated within a specified time window
- **`relative_time`** - Ensures temporal relationships between columns (e.g., ship_date >= order_date)
- **`business_rule`** - Validates custom SQL expressions that represent business logic

## Examples

See [`examples/v1beta2/all_sql_tests_types.yaml`](examples/v1beta2/all_sql_tests_types.yaml) for comprehensive examples of all supported SQL test types with their configuration options.

## Test Creation and Updates

### UUID Generation

SQL tests use deterministic UUID generation based on their configuration. This ensures that:

- **Same configuration = Same UUID**: If you redeploy a test with identical configuration, it will receive the same UUID, allowing the system to recognize it as an update rather than a new test.

The UUID is generated from the following fields:

- Test ID (if provided)
- Template hash (SQL expression for business rule tests)
- Template identifier (Synq path to the monitored entity)
- Test type

If a test ID is already a valid UUID, it is used directly. Otherwise, a deterministic UUID is generated using SHA-1 hashing with a workspace-based seed.

### Creating New Tests

When you add a new test to your YAML configuration and deploy:

1. The system generates a deterministic UUID based on the test configuration
2. If no test with that UUID exists, a new test is created
3. The test begins running according to its schedule

### Updating Existing Tests

When you modify an existing test in your YAML configuration and deploy:

1. The system generates a UUID based on the updated configuration
2. If the UUID matches an existing test, the system recognizes it as an update
3. The test configuration is updated accordingly

**Important**: If you change the test ID, the system will treat it as a deletion of the old test and creation of a new test, even if the configuration is otherwise identical.

## Test Reset Behavior

When a test is updated, the system determines whether the test should be reset (re-triggered) based on the `ShouldReset` flag. A test is reset when certain aspects of its configuration change that require re-evaluation of historical data or test state.

### When Tests Are Reset

Tests will be reset/re-triggered when any of the following changes occur:

1. **Recurrence Rule Changes**: The schedule or frequency of the test changes
2. **Severity Changes**: The severity level (INFO, WARNING, ERROR) changes
3. **Test Type Changes**: The test type itself changes (e.g., from `not_null` to `unique`)
4. **Template Configuration Changes**: Any field within the test template changes:

   - For `not_null`, `empty`, `unique`, `accepted_values`, `rejected_values`, `freshness`, `relative_time`, `business_rule`: Any field change triggers a reset
   - For `min_max`, `min_value`, `max_value`: Changes to `strictly`, `min_value`, `max_value`, or `column` trigger a reset
5. **Evaluator Changes**: Adding, removing, or modifying any evaluator on a `business_query` test triggers a reset

### When Tests Are Not Reset

Tests are updated without reset when:

- Only metadata fields change (name, description) that don't affect the test logic
- Changes that don't impact the test execution behavior
- `save_failures` changes (enables or disables failure run persistence) — stored immediately, no historical data is affected

When a test is reset, all previous test results and state are cleared, and the test begins fresh execution from the point of the update.

### Saving Failure Runs (`save_failures`)

By default, failure run details are not persisted. Set `save_failures: true` to store the full result of each failed test execution. This can be set per-test or in `defaults` (per-test takes precedence):

```yaml
defaults:
  save_failures: true   # apply to all tests

entities:
  - id: project.dataset.table
    tests:
      - type: not_null
        columns: [id]
        # inherits save_failures: true from defaults

      - type: unique
        columns: [id]
        save_failures: false   # override: disable for this test
```

Changing `save_failures` does not reset a test's historical results.

## Evaluators (business_query only)

Evaluators let you attach multiple named, severity-tagged SQL boolean FAIL conditions to a `business_query` test. Each evaluator independently checks a row-level expression against the result rows; when any evaluator fails, the test outcome becomes the worst severity among failing evaluators.

### When to use evaluators

Use evaluators when a single business-query test covers multiple distinct concerns that you want to track and alert on separately (e.g., revenue must be positive **and** tips must not exceed revenue **and** gross margin must be within range).

### Evaluator fields

| Field | Required | Description |
|---|---|---|
| `id` | One of id/name | Stable identifier, unique within the test. If provided, must match `^[A-Za-z0-9_-]+$`. Inferred from `name` when omitted (lowercase, runs of non-alphanumerics → `-`). |
| `name` | One of id/name | Human-readable label shown in alerts and audit reports. Inferred from `id` when omitted (`-`/`_` → spaces). |
| `sql_expression` | Yes | SQL boolean FAIL condition: a row is flagged when the expression evaluates to `TRUE`. `NULL` evaluates as pass. |
| `severity` | No | `INFO`, `WARNING`, or `ERROR`. Defaults to the parent test's severity when omitted. |

> At least one of `id` or `name` must be provided. When only one is given, the other is inferred.

### UUID stability

Adding, removing, or changing evaluators does **not** change the test's UUID, so the test is updated in place and retains its deployment identity.

### Reset behavior

Any change to evaluators (add/remove/edit) resets the test's historical state.

### Example

The `sql_query` of the `business_query` template is the SELECT that produces result rows (e.g. a grouped aggregate query). Each evaluator then defines a named FAIL condition applied to every row of that result set — a row is flagged when the evaluator expression evaluates to `TRUE`.

```yaml
- type: business_query
  id: revenue-metrics-by-region
  name: Daily revenue metrics per region
  sql_query: |
    SELECT
      region,
      COUNT(*)                                    AS statement_count,
      SUM(total_revenue)                          AS total_revenue,
      AVG(total_tips / NULLIF(total_revenue, 0)) AS avg_tip_rate
    FROM financial_statement
    WHERE date = CURRENT_DATE
    GROUP BY region
  evaluators:
    - id: min-statements-per-region
      name: Each region must have at least 5 statements
      sql_expression: "statement_count < 5"
      severity: WARNING
    - id: negative-regional-revenue
      name: Total revenue per region must not be negative
      sql_expression: "total_revenue < 0"
      severity: ERROR
    - id: tip-rate-too-high
      name: Average tip rate must not exceed 30%
      sql_expression: "avg_tip_rate > 0.30"
      # severity omitted → inherits from parent test severity
```

> **Note:** Evaluators are only supported on `business_query` tests. Other test types do not accept the `evaluators:` field.

## Configuration Reference

For complete field specifications and validation rules, refer to `schema.json`. You can also reference the schema in your YAML files for IDE support:

```yaml
# yaml-language-server: $schema=https://schemas.synq.io/synq-monitors/v1/config.schema.json
version: v1beta2
```
