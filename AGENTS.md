# Driving `synqcli`

This guide is for a coding agent that uses `synqcli` to declare data quality monitors, SQL tests and deployment rules as code. It gives the loop to follow, how to read the output, what each refusal means, and the mistakes that cost the most time. Follow it from the top.

`synqcli deploy` reconciles a workspace with a YAML file. You declare which monitors and tests belong on which assets, and `deploy` makes the workspace match. So it creates, it updates, and it deletes: it removes anything that the namespace of the file owns and the file no longer declares. If you remember this, you prevent most of the damage that this tool can do.

Two more facts control almost everything else (§ 3, § 6):

- The `id:` of a check is its address. With an `id:`, you can edit the rest of the check. Without one, the content of the check is its identity, so some edits replace the check.
- A check belongs to exactly one namespace. If a second config declares it, `deploy` refuses. It does not merge.

---

## 1. The loop

```bash
synqcli auth status                        # 1. know which workspace you are about to change
synqcli export --source api existing.yaml  # 2. see what is already declared
synqcli deploy config.yaml --dry-run       # 3. read the plan, especially the deletes
synqcli deploy config.yaml                 # 4. apply, after the same plan is shown again
synqcli deploy config.yaml --dry-run       # 5. confirm the plan is now empty
```

Do not skip step 5. An empty plan is the only proof that the file and the workspace agree, and that the file is now the source of truth. If the plan is not empty, the next person who runs `deploy` gets changes that they did not write.

Get the schema into the editor first. It is the field reference, and it has more authority than any prose:

```bash
synqcli schema > schema.json
```

Start every config file with this line. It gives completion and validation while you type:

```yaml
# yaml-language-server: $schema=https://schemas.synq.io/synq-monitors/v1/config.schema.json
```

The URL serves the same schema, so the local copy is optional.

---

## 2. Entity ids: resolve them, never guess them

A wrong entity id is the most common failure, and the CLI can help with it the least. A wrong id is not a typo that it can correct: the asset is not there.

You can write an entity id in three forms. `deploy` resolves all three against the workspace before it deploys anything:

| Form | Example | Resolved by |
|---|---|---|
| Full path, `::` separated | `ch-prod::default::runs` | direct lookup |
| Full path, dot separated | `ch-prod.default.runs` | dots are changed to `::`, then looked up |
| Warehouse coordinates (SQL FQN) | `analytics.public.orders` | matched against known warehouse objects |

The first two forms are the same path. The third is the name of the table in the warehouse. It is matched only for ids that the first lookup did not find.

Resolution works only for an asset that the platform already knows, and only for a table, a view or an equivalent model.

- An id of the same asset under another integration (a Coalesce node or a dbt model) is accepted. It resolves to the warehouse table, or to its view if there is no table. If you write such an id as a full `::` path, `deploy` prints the replacement as `<id> → <path>`.
- Other known entities (a dashboard, a job) cannot be monitored and fail with the message below.

Get real ids. Do not construct them. Two sources are reliable:

- Run `synqcli export --source all out.yaml` and read the `entities[].id` values. They are already in the form that the config accepts.
- The page of the asset in the web app shows the same path.

### Reading the failure

```
The following entity IDs could not be resolved:
  - ch-prod.default.runz
```

The id names nothing in this workspace. It is misspelled, or the asset is not ingested yet, or (more often than it seems) you are on the wrong workspace. Check `auth status` before you check the spelling.

```
The following entity IDs resolved to multiple entities:
  - orders
      - ch-prod::default::orders
      - bq-prod::analytics::orders
  Please specify more specific entity IDs to resolve to a single entity.
```

A short or coordinate-shaped id matched more than one asset. Replace it with one of the full paths listed below it. They are printed so that you can paste them into the file. A Coalesce node or dbt model that builds several views and no table gives the same message.

```
coalesce-prod::env::node-1 (coalesce node) has no warehouse table or view to monitor
```

The id names a real asset, but nothing in the warehouse that a check can run against: a node that is not materialised, or an asset that is not a table. Point the check at the table or view itself.

An unresolved id fails the whole run, also `--dry-run`, and no plan is printed. Fix every id before you expect a plan.

---

## 3. The namespace is the reconcile boundary

```yaml
namespace: "team-analytics"
```

`namespace` groups checks, so that different teams reconcile independently. It is the most important field in the file:

- Deletes are limited to it. `deploy` removes checks that this namespace owns and that the file no longer declares. It never changes the checks of another namespace.
- A missing `namespace` is also a namespace, the default one. It is not "no grouping".
- A wrong namespace is not a no-op. A typo means "this namespace declares nothing that it owned before", which is a request to delete.
- Several files can share one namespace. They are parsed together and reconciled as one unit, so the plan for a namespace includes every file that declares it.

`deploy --namespace <name>` selects which namespaces of the parsed files this run processes. You can repeat it. It does not change what a namespace owns.

`export --namespace` is a different thing. On `export`, `--namespace` is the value written into the generated YAML: a label, not a filter. `export` has no namespace filter. It returns everything that matches the other scopes and writes your namespace onto all of it. If you export the whole workspace under one namespace and deploy that file, one config takes the checks of all other configs.

---

## 4. `--dry-run` and the confirmation prompt are not the same thing

- `--dry-run` computes and prints the plan, then exits without a deploy. It never prompts, and it writes nothing.
- Without `--dry-run`, the same plan is printed. Then the run asks, per namespace, if it applies the plan. Any answer other than `y` cancels that namespace, and the run continues with the next one.
- `--auto-confirm` answers `y` for you. The plan is still printed, but nobody reviews it.

This is important for an agent:

- Without `--auto-confirm`, a non-interactive run waits on the prompt. It does not continue and it does not fail.
- `--auto-confirm` removes the review step. It is safe only after you read a `--dry-run` of the same file.

The order that does not block and is reviewed:

```bash
synqcli deploy config.yaml --dry-run     # read this output, it is the review
synqcli deploy config.yaml --auto-confirm
```

`--dry-run` and `--auto-confirm` together mean `--dry-run`: nothing is deployed.

---

## 5. Authoring the config

A complete, minimal file:

```yaml
# yaml-language-server: $schema=https://schemas.synq.io/synq-monitors/v1/config.schema.json
version: v1beta2
namespace: "team-analytics"

defaults:
  severity: ERROR

entities:
  - id: ch-prod.default.orders
    time_partitioning_column: created_at
    monitors:
      - id: orders_volume
        type: volume

      - id: orders_freshness
        type: freshness
        expression: created_at

    tests:
      - id: order_id_unique
        type: unique
        columns: [order_id]

      - id: required_order_fields
        type: not_null
        columns: [customer_id, order_date]

      - id: order_status_values
        type: accepted_values
        column: status
        values: [pending, processing, shipped, delivered, cancelled]
```

Give every monitor and test an `id:` of your own, as above. It is optional, but it is the address of the check. With an `id:`, an edit to any other field updates the check and keeps its history. Without one, many edits replace the check (§ 6).

Use `version: v1beta2`. `v1beta1` is legacy and exists only for compatibility. Do not write new files in it.

### Check types

Monitor types watch an asset over time and learn what is normal: `volume`, `freshness`, `field_stats`, `custom_numeric`, `category_distribution`, `table_stats`, `custom_table_stats`. A `field_stats` column can have its own metric selection, and each metric its own sensitivity.

SQL test types assert that something is true now: `not_null`, `empty`, `unique`, `accepted_values`, `rejected_values`, `min_max`, `min_value`, `max_value`, `freshness`, `relative_time`, `business_rule`, `business_query`, `relationships`.

- `business_rule` and `business_query` carry your own SQL. They accept `{{ table }}`, which is replaced with the fully qualified name of the anchored table.
- On a `sql_tests` deployment rule, `{{ table }}` lets one query cover every matched table. Any other `{{ name }}` token is rejected on save. See [SQL tests](README_SQL_TESTS.md).

### Deployment rules

Deployment rules (`deployment_rules`, and `deployment_exclusions` to remove assets again) apply checks by query, not by a list of assets. So a new matching asset is covered without a change to the file.

- `type` tells what a rule deploys. `table_stats` deploys monitors. `sql_tests` deploys the SQL tests in its own `tests:` onto every table or view that the selection matches.
- On a `sql_tests` rule, the schedule, timezone, severity and `save_failures` are set on the rule and apply to every test. A test can override only `severity`.
- A `sql_tests` rule deploys each test only onto matched tables that have every column the test reads. The plan lists the others as skipped tests. So a broad selection is safe: each table gets the tests that fit it, and the plan shows where they went.
- `export` writes `sql_tests` rules back into `deployment_rules`, so you can adopt a workspace that has them with the loop in § 1.
- `export` writes a `table_stats` rule as the single-asset `table_stats` monitors that it covers. Its query form is only for authoring. If a file must keep the query form, keep your hand-written version.

### Where to find fields

In order of authority:

1. [The published schema](https://schemas.synq.io/synq-monitors/v1/config.schema.json): every field and every accepted value.
2. [`examples/`](examples/): one file per monitor type, per test type and per pattern. You can deploy each file as written.
3. [SQL tests](README_SQL_TESTS.md) for the test types in prose, and [Monitors as code](https://docs.synq.io/monitors/monitors-as-code) for what the YAML declares and why.

An unknown key on an entity, a monitor or a test is an error, so a misspelled field there fails the run. Inside the options of a single test, this check is weaker, so a successful run does not prove that every key was applied. Check the plan: an accepted field shows in the diff of an existing check.

---

## 6. What re-deploying changes, and what it replaces

`deploy` matches a check by its id. So the source of the id decides if an edit is an update or a replacement. A replacement is not an edit: the old check is deleted with its issue history and error runs (and, for a monitor, its learned baseline), and a new check is created.

### The `id:` is the address

| `id:` | What identifies the check |
|---|---|
| a name of your own, for example `orders_nulls` | the namespace, the entity, the check type and that name, nothing else |
| a UUID | the UUID itself; this is what `export` writes |
| omitted | the content of the check, which is different per type (tables below) |

With an `id:` of your own, everything except the entity and the type is an attribute that you can edit in place. Examples: the `metric_aggregation` of a `custom_numeric`, the column of a `category_distribution`, the columns or SQL of a test, the mode, the segmentation, the partitioning column. Three edits still replace the check:

- A new `id` is a new address, so it is a new check.
- A move to another entity is a new check, because the API refuses to point a monitor at a different asset.
- A new `type` is a new check. A `volume` monitor that becomes a `freshness` monitor measures something else, so its baseline and history have no meaning.

Without an `id:`, the content is the identity. This lets two checks of the same type sit on one entity with no ids, but an edit to the content replaces the check. The content is:

- for a `custom_numeric`, its `metric_aggregation`;
- for a `category_distribution`, its column;
- for a test, its columns, and for `business_rule` or `business_query` also the SQL;
- for every monitor, also the mode, the time-partitioning expression and the segmentation expression.

Two `field_stats` monitors on one entity are an exception. Their content does not include `columns`, so two that differ only there get the same id, and the deploy fails on the duplicate. Give at least one of them an `id:`.

### Monitor edits

| Edit | With an `id:` | Without one |
|---|---|---|
| `severity`, `sensitivity`, thresholds, `timezone`, schedule, `name`, `description`, `filter`, `category`, the `never_negative` of a `custom_numeric` | update in place | update in place |
| the `metric_aggregation` of a `custom_numeric` | update in place | replace |
| the column of a `category_distribution` | update in place | replace |
| `time_partitioning_column`, segmentation | update in place | replace |
| switch `anomaly_engine` ⇄ `fixed_thresholds` | update in place | replace |
| the `columns` of a `field_stats` | update in place | update in place |
| the metrics of a `field_stats` column | update in place | update in place |
| move the monitor to a different entity | replace | replace |
| change the monitor `type` | replace | replace |
| rename the `id` | replace | n/a |

### Test edits

| Edit | With an `id:` | Without one |
|---|---|---|
| `severity`, recurrence, evaluators, `name`, `description`, `save_failures`, `category` | update in place | update in place |
| the `values` of an `accepted_values`, the bounds of a `min_max` | update in place | update in place |
| the `column`/`columns` of a test | update in place | replace |
| the SQL of a `business_rule` or `business_query` | update in place | replace |
| move the test to a different entity | replace | replace |
| change the test `type` | replace | replace |
| rename the `id` | replace | n/a |

### Deployment rule edits

Rule identity works the same way, and the risk is higher. If you delete a `sql_tests` rule, you delete every test that the rule deployed, with their check entities and history. Nothing can restore them.

- If a rule has no `id:`, a rename replaces it. The old rule is deleted with everything it deployed, and the next scheduled sync builds the tests again from the start under the new name.
- A monitor rule is safer: its monitors stay while some rule still covers the asset. But the id of the rule changes, so every link and every `id=` that someone kept stops working.

| `id:` on a rule | What identifies the rule |
|---|---|
| a name of your own | the namespace and that name, so you can edit the `name:` of the rule |
| a UUID | the UUID itself; `export` writes one |
| omitted | the namespace and the `name:` of the rule, so a rename replaces it |

This applies to both kinds of rule and to a `deployment_exclusions` entry. Everything else is an attribute: the `resolver_ql` selection and the fields that describe the deployed checks. The two kinds have different attributes. A field of the other kind is rejected, not ignored.

| Edit | With an `id:` | Without one |
|---|---|---|
| `resolver_ql`: broaden, narrow, or only reformat | update in place | update in place |
| `severity`, either kind | update in place | update in place |
| `metrics`, `sensitivity`, `delay_model`, `table_stats` only | update in place | update in place |
| the `tests:` of a rule, and `schedule`, `timezone`, `save_failures`, `keep_removed_tests`, `sql_tests` only | update in place | update in place |
| `name` | update in place | replace |

An update in place syncs the rule again. Checks appear on assets that the selection now matches and go away from assets that it no longer matches. Assets that match before and after keep their checks and history.

- An entity-anchored `table_stats` monitor becomes a rule on that one asset, and each asset has one such rule. So the asset is its address, and its `id:` accepts only the UUID that `export` writes for it, which adopts that rule. A name of your own there is rejected.
- If you change such a monitor into a query rule under `deployment_rules`, it is a different rule with a different selection, not an edit. Expect a create and a delete.
- A rule and an exclusion can share a name or an `id:`. They are different things, and each keeps its own identity.

### Reading a replacement in the plan

`--dry-run` shows a monitor replacement on one line: `Replaced, id changed: <old> → <new>`, with what it discards and the fields that differ. It counts as neither a create nor a delete.

- The plan pairs any one monitor that leaves and one that arrives on the same asset with the same type. This can be a renamed `id:`, an edit to a monitor without an `id:`, or two unrelated monitors that share the asset and the type.
- Read the diff, not the word "Replaced". The diff tells which of the three it is. The line above prints the name of the new monitor, so a `name:` in the diff means that the old monitor had a different name.
- An entity move and a `type:` change stay a delete plus a create, in the two sections of the plan.
- A replaced test or rule is also a delete plus a create. If you edited an `id:` or the `name:` of a rule, read those two sections.

"Update in place" means that the check keeps its id and its history. Some updates in place still discard what the check learned (§ 7).

---

## 7. Resets and re-runs: updates that are not free

Some updates in place keep the check but change what it has. The plan shows these. The cost is different for a monitor and for a test.

### Monitor reset

A monitor reset discards the learned baseline. The monitor has no anomaly baseline until it collects enough history again, and during that time it does not really watch anything. A monitor resets when one of these changes:

- its timezone;
- its mode (`anomaly_engine` ⇄ `fixed_thresholds`);
- its schedule (type, time of day, minute of hour, or delay);
- its `metric_aggregation`, or the column or `top_k_limit` of a `category_distribution`;
- its time-partitioning expression or interval;
- its segmentation.

The mode, the `metric_aggregation`, the `category_distribution` column, the partitioning expression and the segmentation cause a reset only when the monitor has an `id:`. Without one, they are part of its identity, so the same edit replaces the monitor, which is worse.

### Test re-run

A test re-run discards nothing. A test has no learned state. The plan only tells you that the test runs again: it is re-armed and runs at once, not at its next scheduled time.

- A test is re-armed when its recurrence, its severity, its evaluators or its template body change. The template body is any test-specific field, for example the columns, the `values` of an `accepted_values`, or the bounds of a `min_max`.
- A test with no `schedule:` runs from a trigger outside `synqcli`. It is not re-armed, and runs at its next external trigger.
- A test loses data only on a replacement (§ 6). A new id means a new check entity, so its issues and error runs are gone. An `id:` prevents this, not the re-run.

No flag suppresses a reset, and you cannot deploy the change without it. You can accept it or leave the field unchanged. So notice it in the plan and tell the person before you confirm.

---

## 8. Refusals, and what each one means

### "managed by other configs": the deploy is refused

```
🚫 2 monitors managed by other configs.
   - Monitor ID: 7f3b…, Managed by namespace: team-platform
```

This is the ownership gate, the only condition that refuses a deploy. Your file declares a check that exists and belongs to a different namespace. If two configs claimed one check, each deploy would undo the other, so the tool stops.

It stops the entire run, not only that namespace, and also `--dry-run`. No plan is printed for any namespace until you resolve it.

Try these steps in order:

1. Check if the namespace on your side is wrong. The message names the namespace that owns the check. If your file should declare that namespace, fix `namespace`, and the conflict goes away.
2. Stop declaring the check. If it belongs to the other config, remove it from yours. This is usually the correct answer.
3. Transfer it, if it must move. Remove it from the owning config and deploy that config first. Then add it to yours and deploy again. Between the two deploys the check does not exist: it is deleted and created again, so it loses its history. There is no handover in place.

Which config owns a check is a decision about how a team works. Ask the owner of the config. Do not edit the file of the other team.

### "Managed by API (No Config)" and monitors created in the app: not refused

Both show in the plan under their own headings, and neither blocks anything. They are adoptions. If your config declares a check that was created in the web app, or over the API without a namespace, your namespace takes it. After that it is yours, and a later deploy that stops declaring it deletes it.

This is usually what you want, but it is silent and one-way. If someone maintains a check in the UI on purpose, a config that declares it takes it from them.

### Everything else that stops a run

| Refusal | What it means |
|---|---|
| An entity id could not be resolved | § 2: the asset is unknown, or you are on the wrong workspace |
| An entity id resolved to multiple entities | § 2: use one of the full paths it lists |
| An entity id has no warehouse table or view to monitor | § 2: it names an asset that is not a table; use the table or view itself |
| Duplicate checks | Two entries in the file have the same identity. Give them different columns, expressions or ids |
| A validation error | The message names the field. Check it against the schema |
| A field is rejected as unknown | Unknown keys are errors, not warnings. Usually a misspelling, or a v1beta1 field in a v1beta2 file |
| A parse error in a file you did not mean to deploy | `deploy` with no arguments reads every `.yaml` and `.yml` in the working directory. Pass files explicitly |

These errors are collected over every file and namespace and reported together. Then the run exits and deploys nothing. Fix them all in one pass.

---

## 9. Running it non-interactively

```bash
synqcli deploy config.yaml --dry-run
synqcli deploy config.yaml --auto-confirm
```

Credentials, in order of precedence:

1. The `--client-id` / `--client-secret` flags.
2. `QUALITY_CLIENT_ID` / `QUALITY_CLIENT_SECRET`, or a pre-issued `QUALITY_TOKEN`.
3. A `.env` file in the working directory.
4. A cached browser login.

Client credentials come before a cached login, so the last person who logged in does not change CI. `synqcli auth login --coalesce [--organization <org-id>]` caches a login with your Coalesce account. That credential carries your Coalesce Quality roles in the workspace of the organization, not consented permissions.

- Select the deployment by name: `--region eu|us|au`, or `--endpoint` for a self-hosted one. `synqcli auth login` remembers the choice, and `synqcli auth use <region>` changes it, so later commands need no flag.
- A wrong region is the most expensive mistake here. It reconciles a workspace that does not declare your checks, and to reconcile means to delete.
- Confirm the target before you write. Every `deploy` and `export` prints the workspace that it connected to before it does anything. Read that line.
- Set `--log-file` or `$QUALITY_LOG_FILE` to get a JSONL execution log: one record per phase, per namespace result and per error. Parse this log, not the formatted output.
- Exit code `0` means the command did what it said. `1` means it did not: a failed deploy or export, and also a mistyped flag, an unknown command, a missing argument, or an unusable `--region` / `--endpoint`. So a CI step that checks the status also catches an invocation that an older `synqcli` does not understand.
- A cancelled namespace is not a failure: a declined prompt exits `0`. `--dry-run` also exits `0`, with or without changes. If a CI gate needs "nothing to deploy", read the log file, not the status.

---

## 10. `advisor`: propose tests for an asset

`advisor` asks a model to suggest tests for an entity, and can write them into a config:

```bash
synqcli advisor \
  --entity-id "ch-prod::default::orders" \
  --instructions "Suggest data quality tests for this table" \
  --output ./suggested
```

- `--output` is a directory. It gets one file per entity. An existing file is skipped unless you pass `--force`.
- `--entity-id` can be repeated.
- `--columns` narrows the scope. `--instructions-file` reads a long prompt from a file. `--severity` sets the default severity, and `--namespace` the group it writes into.
- `--connections` lets it profile the warehouse. With it, it proposes real `accepted_values` and real bounds, not guesses.

Review what it writes before you deploy it. The safe order is `--output`, then read, then `deploy --dry-run`. It only proposes, and it does not know what the workspace already has.

Specific instructions give better results. Some that work:

```
This is an orders table. Suggest tests for:
- Order ID uniqueness
- Required fields: customer_id, order_date, total_amount
- Amount validation (total_amount >= 0)
- Date consistency (order_date <= ship_date)
- Status values should be: pending, processing, shipped, delivered, cancelled
```

```
This is financial transaction data. Suggest tests for:
- Transaction ID must be unique
- Amount must be non-negative
- Currency code should be valid ISO 4217
- Created timestamp must not be in the future
- Account IDs must not be null
```

```
This is time series data. Suggest tests for:
- Timestamp freshness, within the expected arrival interval
- Volume monitoring, to detect missing periods
- Metric value ranges
- No duplicate timestamps per entity
```

---

## 11. What each command costs

- `schema`: free and offline. No credentials, no API call.
- `auth status`: reads local files only. No network.
- `export`: reads the workspace, and is cheap. It refuses to overwrite an existing output file, so give it a new path.
- `deploy --dry-run`: resolves every entity id and compares with the workspace. Cheap, and always worth it.
- `deploy`: writes. The cost is the blast radius, not the runtime: it creates, updates and deletes in one pass.
- `advisor`: one model call per entity, plus warehouse metadata reads, and with `--connections` profiling queries on the warehouse. It is the only command with a cost per run that matters. Limit it with `--entity-id` and `--columns`.
- `upgrade`: replaces this binary with the latest release, verified against the `checksums.txt` of the release. `upgrade --check` reports what it would do and changes nothing. Neither touches the workspace.

About once a day, `synqcli` mentions a newer release on stderr. It is silent when an agent drives it, when output is not a terminal, in CI and in a container, so the notice never gets into output that you parse. `QUALITY_NO_UPDATE_CHECK=1` turns it off.

---

## 12. Never do these

- Never run `deploy` on a workspace whose config you did not write without first reading a `--dry-run`. Deploy is a reconcile, so an incomplete file is a delete.
- Never guess a `namespace`. A typo does not narrow the run, it asks for a delete.
- Never export the whole workspace under a namespace and deploy it back. `export --namespace` labels, it does not filter (§ 3).
- Never invent a UUID for an `id:`. A name of your own is the supported address (§ 6). A UUID that you made up addresses nothing, and on an entity-anchored `table_stats` monitor it adopts a rule that does not exist. Copy a UUID only from `export` or from the app.
- Never assume that a rename is a rename. Check it against § 6, and read the replacement line and the delete list in the plan.
- Never confirm a monitor reset yourself. If the plan says that a monitor resets, say so and let the person decide, because the discarded baseline cannot be recovered. A test that "will run again" discards nothing and needs no pause.
- Never point a config at a workspace that you did not confirm. Run `auth status`, and read the workspace line that the command prints.
- Never edit the schema or the CLI reference by hand. Both are generated.

---

## 13. When something looks wrong

| Symptom | Cause |
|---|---|
| The plan deletes checks you did not expect | A wrong `namespace`, or a file that is missing what the namespace already owns. Export first |
| The command hangs and prints nothing more | It waits on the confirmation prompt. Use `--dry-run`, or `--auto-confirm` |
| A check was created again instead of updated | Its address changed (a renamed `id:`, another entity, another `type`), or it has no `id:` and a field of its content changed. § 6 |
| A monitor updated but lost its baseline | A reset, § 7 |
| "managed by other configs" refuses everything | The ownership gate, § 8 |
| An entity id does not resolve | § 2. Check the workspace before the spelling |
| A check created in the app is now in your plan | Adoption, which is expected and one-way, § 8 |
| `deploy` complains about YAML you never wrote | With no file arguments, it reads the working directory. Pass files explicitly |
| Nothing deploys and the status is still `0` | The prompt was declined, or `--dry-run` was set |

The full command and flag reference is generated from the CLI and published as the [CLI reference](https://docs.synq.io/monitors/cli). For what the YAML declares and why, start at [Monitors as code](https://docs.synq.io/monitors/monitors-as-code). For the test types in detail, see [SQL tests](https://docs.synq.io/monitors/sql-tests) and [Deployment rules](https://docs.synq.io/monitors/deployment-rules).
