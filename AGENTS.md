# Driving `synqcli`

This is the operating guide for a coding agent using `synqcli` to declare data
quality monitors, SQL tests and deployment rules as code: the loop to follow, the
mistakes that cost real time, and what each command does to a workspace. It is
written to be acted on top-down.

**`synqcli` reconciles a workspace with a YAML file.** You declare which monitors
and tests belong on which assets; `deploy` makes the workspace match. That means it
creates, it updates, and **it deletes** — anything in the file's namespace that is
no longer declared goes away. Understanding that one sentence prevents most of the
damage this tool can do.

---

## 1. The loop

**1. See what already exists — `export`**

Never start from a blank file against a workspace that already has monitors.
`export` writes the current state back out as YAML:

```bash
synqcli export --namespace my_namespace current.yaml
```

That gives you the real shape of what is deployed, in the format `deploy` accepts.
Edit *that* rather than authoring from scratch, and you will not silently delete
someone's monitors.

**2. Get the schema into your editor**

```bash
synqcli schema > schema.json
```

Then put this at the top of every config file, which gives completion and
validation as you type:

```yaml
# yaml-language-server: $schema=https://schemas.synq.io/synq-monitors/v1/config.schema.json
```

**3. Preview — `deploy --dry-run`**

```bash
synqcli deploy config.yaml --dry-run
```

Prints the create / update / delete plan and touches nothing. **Read the delete
list every time.** This is the step that catches a wrong `namespace`, a mistyped
asset path, or a file that is missing half of what the workspace has.

**4. Deploy**

```bash
synqcli deploy config.yaml
```

Interactive by default: it shows the plan and asks. In CI, add `--auto-confirm`,
but only once the same config has been through `--dry-run` at least once.

**5. Iterate on the diff, not the file**

Re-run `--dry-run` after every edit. An empty plan is the goal — it means the
workspace matches your file, so the config is now the source of truth.

---

## 2. Never do these

- **Never run `deploy` without `--dry-run` first** on a workspace you did not
  author the config for. Deploy is a reconcile, so an incomplete file is a delete.
- **Never guess a `namespace`.** It is the grouping key: everything under one
  namespace is reconciled together, and a typo means "this namespace has nothing
  declared, so delete everything in it". Get it from `export`.
- **Never hand-assign monitor ids.** They are derived deterministically from the
  monitor's identity, which is what makes a re-deploy an update instead of a
  duplicate. Let the tool compute them.
- **Never edit the published docs or the schema by hand.** Both are generated.
- **Never assume a rename is a rename.** Changing something in a monitor's
  identity produces a *new* monitor and deletes the old one — with its history.
  `--dry-run` shows that as a delete plus a create; look for it.
- **Never point a config at a workspace you have not confirmed.** See below.

---

## 3. Confirm the target

Credentials resolve in this order:

1. `--client-id` / `--client-secret` flags
2. `QUALITY_CLIENT_ID` / `QUALITY_CLIENT_SECRET`
3. A `.env` file in the working directory
4. A browser login, cached and shared with the other Coalesce Quality CLIs

```bash
synqcli auth login          # browser flow
synqcli auth status         # every stored credential, every region
```

Deployments are named, not spelled out: `--region eu|us|au`, or `--endpoint` for a
self-hosted deployment. `auth login` remembers the choice, so later commands need
no flag. **A wrong region is the most expensive mistake available here** — it
reconciles the wrong workspace, deleting what it does not find declared.

---

## 4. What goes in the YAML

One file declares monitors and tests per asset:

- **Monitors** watch an asset over time: `volume`, `freshness`, `field_stats`,
  `custom_numeric`.
- **SQL tests** assert something is true right now: `not_null`, `unique`, `empty`,
  `accepted_values`, `rejected_values`, `min_max`, `min_value`, `max_value`,
  `freshness`, `relative_time`, `business_rule`, `business_query`.
- **Deployment rules** apply monitors by *query* rather than by listing assets, so
  new matching assets are covered without editing the file.

Asset paths can be written in a simplified form and are resolved against the
workspace, so you do not need to spell out full warehouse coordinates.

The field-by-field reference is
[the published schema](https://schemas.synq.io/synq-monitors/v1/config.schema.json),
and it is authoritative in a way prose is not. Read it rather than inferring field
names.

---

## 5. `advisor` — propose tests for an asset

`advisor` asks the model to suggest tests for an entity, and can write them
straight into a config:

```bash
synqcli advisor \
  --entity-id "<integration>::<database>::<schema>::<table>" \
  --instructions "Suggest data quality tests for this table" \
  --output suggested.yaml
```

Useful flags: `--columns` to narrow the scope, `--instructions-file` for a long
prompt, `--severity` for the default it assigns, `--namespace` for the group it
writes into, and `--deploy` to deploy immediately.

**Review what it writes before deploying it.** `--output` then read then `deploy
--dry-run` is the safe order; `--deploy` skips your review, not the reconcile.

Instructions steer it, and being specific pays. Some that work:

```
Suggest comprehensive data quality tests for this table. Focus on:
- Primary key uniqueness
- Required fields (NOT NULL constraints)
- Data freshness monitoring
- Volume anomaly detection
```

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

## 6. What each command costs

- `schema` — free, offline. No credentials, no API call.
- `export` — reads the workspace. Cheap.
- `deploy --dry-run` — resolves asset paths and diffs against the workspace.
  Cheap, and always worth it.
- `deploy` — writes. The cost is the blast radius, not the runtime: creates,
  updates and **deletes** in one pass.
- `advisor` — a model call per entity, plus warehouse metadata reads. The only
  command here with a per-run cost worth thinking about; scope it with
  `--entity-id` and `--columns` rather than pointing it at everything.

---

## 7. When something fails

| Symptom | Cause |
|---|---|
| The plan deletes monitors you did not expect | Wrong `namespace`, or a config missing what the workspace has — `export` first |
| A monitor was recreated instead of updated | Something in its identity changed, so it is a different monitor |
| "Breaking change" refuses to deploy | Deliberate: the change would reset a monitor's baseline or a test's history |
| An asset path does not resolve | It is resolved against the workspace, so the asset must be known to Coalesce Quality first |
| Deploying to the wrong workspace | `auth status`, and check `--region` |
| A field is rejected | Check it against the published schema; strict decoding rejects unknown fields rather than ignoring them |

The full command and flag reference is generated from the CLI itself and published
as the [CLI reference](https://docs.synq.io/monitors/cli). For what the YAML
declares and why, start at
[Monitors as code](https://docs.synq.io/monitors/monitors-as-code).
