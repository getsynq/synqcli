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

### How a test is identified

A test's id is derived, deterministically, so redeploying the same file updates
the tests instead of duplicating them. What the id is derived from is what decides
whether an edit updates a test or replaces it.

**Write an `id:` and it is the test's address.** The derived id is then a hash of
the namespace (`namespace:`, sent as the config id), the entity, the test type and
that `id:` — and of nothing else, so every other field is editable in place. A
UUID in `id:` is used verbatim, which is what `export` writes.

**Omit `id:` and the test's content is its identity**: the namespace, the entity,
the test type, the columns it names, and the SQL of a `business_rule` or a
`business_query`. That is what lets two tests of one type sit on one entity
without either naming an id, and the cost is that editing a column or the SQL
replaces the test.

Either way the hash is seeded with the workspace, so the same file deployed to two
workspaces produces two independent sets of tests.

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

**Important**: renaming `id:` — or moving a test to another entity, or changing its
type — is a deletion of the old test and the creation of a new one, even when the
configuration is otherwise identical. The new test is a new check entity, so the
old test's issues and its error runs do not carry over. The deploy plan shows this
as a delete plus a create; read both sections after editing an `id:`.

## Test Re-Run Behavior

When a test is updated, `synqcli` decides whether to re-trigger it. This is a
re-run, not a reset: **a test has no learned state, and nothing is discarded.** A
test that carries a `schedule:` is re-armed and runs immediately instead of
waiting for its next slot; a test with no `schedule:`, triggered by something
outside `synqcli`, is not re-armed at all and simply runs at its next external
trigger.

### When Tests Are Re-Triggered

A test is re-armed when any of the following changes occur:

1. **Recurrence Rule Changes**: The schedule or frequency of the test changes
2. **Severity Changes**: The severity level (INFO, WARNING, ERROR) changes
3. **Test Type Changes**: the test type itself changes (e.g., from `not_null` to `unique`) — note this also *replaces* the test, since the type is part of its address
4. **Template Configuration Changes**: Any field within the test template changes:

   - For `not_null`, `empty`, `unique`, `accepted_values`, `rejected_values`, `freshness`, `relative_time`, `business_rule`: any field change re-arms it
   - For `min_max`, `min_value`, `max_value`: changes to `strictly`, `min_value`, `max_value`, or `column` re-arm it
5. **Evaluator Changes**: adding, removing, or modifying any evaluator on a `business_query` test re-arms it

### When Tests Are Not Re-Triggered

Tests are updated without a re-run when:

- Only metadata fields change (name, description) that don't affect the test logic
- Changes that don't impact the test execution behavior
- `save_failures` changes (enables or disables failure run persistence) — stored immediately, no historical data is affected

A re-run keeps the test's results and its history. Losing those takes a
*replacement* — see "How a test is identified" above.

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

Changing `save_failures` does not discard a test's historical results — no edit to
a test does. Only replacing the test does, and only because the replacement is a
different check.

### Categories (`category`, `governance_category`)

A test can declare its own categories. `category` is the **technical** dimension — what kind of check this is mechanically, e.g. `nullness` or `uniqueness`. `governance_category` is what the check is *for*, the data quality dimension governance reports on, e.g. `completeness`. Both are free-form strings — use whatever vocabulary your categorisation rules already use — and they resolve independently, so a test may set either, both, or neither:

```yaml
entities:
  - id: project.dataset.table
    tests:
      - type: not_null
        columns: [id]
        category: nullness
        governance_category: completeness

      - type: unique
        columns: [id]
        category: uniqueness
        # governance category left to the categorisation rules
```

What a test declares here **takes precedence over your workspace's categorisation rules** for that test. Leave a category out and the rules decide it as before; delete one from the file and the next deploy hands that dimension back to the rules. Values are shown with underscores as spaces, so `snake_case` reads well.

There is deliberately no `defaults:` entry for either field. A default would categorise every test in the file, and because a declared category outranks the rules, that would switch the rules off for all of them rather than fill a gap.

Changing a category does not discard a test's historical results, and does not even re-run it.

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

### Re-run behavior

Any change to evaluators (add/remove/edit) re-arms the test, so it runs again
straight away. Nothing is discarded — its previous results and its history stay.

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

## The `{{ table }}` placeholder

`business_rule` and `business_query` carry SQL you write yourself, so nothing in
them names the table unless you do. Write `{{ table }}` and it is replaced with
the fully qualified name of the table the test is anchored to, quoted for your
warehouse. It is the only supported placeholder — any other `{{ name }}` token is
rejected when the test is saved.

The substitution is textual, so the token is replaced everywhere it appears —
inside a string constant or a comment as well as in a `FROM` clause. There is no
escape; write the name out yourself if you need the literal text `{{ table }}` in
a result.

On a single test it saves repeating a name you already gave under `entities:`. On
a `sql_tests` deployment rule it is the difference between a test and a copy: the
rule deploys its `tests:` onto every table the selection matches, so without the
placeholder all of them run the same query against whichever table you typed.

```yaml
deployment_rules:
  - name: No negative order totals
    type: sql_tests
    resolver_ql: with_columns("total")
    tests:
      - type: business_query
        name: Orders with a negative total
        sql_query: |
          SELECT * FROM {{ table }} WHERE total < 0
```

The placeholder renders a table name, so put it where a table name goes. In a
`business_query` you own the whole statement, so you can alias it and qualify
columns off that alias — which is what a correlated subquery needs:

```yaml
- type: business_query
  name: Every order is in the audit log
  sql_query: |
    SELECT t.* FROM {{ table }} t
    WHERE t.id NOT IN (SELECT id FROM audit a WHERE a.ref = t.id)
```

In a `business_rule` the `FROM` clause is generated for you and carries no alias,
so the placeholder is for a subquery of your own rather than for qualifying a
column:

```yaml
- type: business_rule
  name: No order beats the running maximum
  sql_expression: "total > (SELECT max(total) FROM {{ table }})"
```

Writing `{{ table }}.column` is not portable: on BigQuery the fully qualified name
is a single quoted identifier and cannot qualify a column. Reach for
`business_query` when you need to name the anchor's columns.

Two things to know before you add one to a test that already exists:

- The placeholder is stored unrendered and substituted per table, so one rule test
  covers N tables and editing it is one edit.
- A `business_rule` or `business_query` without an `id:` is identified by its
  content, SQL included (see "How a test is identified"), so adding a placeholder
  to a deployed test replaces it and it starts a fresh history. Give the test an
  `id:` to edit it in place instead. A test a deployment rule deploys is identified
  by its `name:`, so there the edit is in place either way — but renaming it mints
  a new test.

## Configuration Reference

For complete field specifications and validation rules, refer to the [published schema](https://schemas.synq.io/synq-monitors/v1/config.schema.json). You can also reference it from your YAML files for IDE support:

```yaml
# yaml-language-server: $schema=https://schemas.synq.io/synq-monitors/v1/config.schema.json
version: v1beta2
```
