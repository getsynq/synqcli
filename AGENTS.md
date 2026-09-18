# Driving `synqcli`

This is the operating guide for a coding agent using `synqcli` to declare data
quality monitors, SQL tests and deployment rules as code: the loop to follow, how
to read what it prints, the refusals and what each one actually means, and the
mistakes that cost real time. It is written to be acted on top-down.

**`synqcli deploy` reconciles a workspace with a YAML file.** You declare which
monitors and tests belong on which assets; `deploy` makes the workspace match.
That means it creates, it updates, and **it deletes** — anything the file's
namespace owns that the file no longer declares is removed. Understanding that one
sentence prevents most of the damage this tool can do.

Two more facts decide almost everything else, and both are covered in full below.
A check is addressed by its `id:` — write one and the rest of the check is
editable, leave it out and its content is its identity, so some edits replace the
check instead of updating it. And a check belongs to exactly one namespace, so
pointing a second config at it is refused rather than merged.

---

## 1. The loop

```bash
synqcli auth status                        # 1. know which workspace you are about to change
synqcli export --source api existing.yaml  # 2. see what is already declared
synqcli deploy config.yaml --dry-run       # 3. read the plan, especially the deletes
synqcli deploy config.yaml                 # 4. apply, after the same plan is shown again
synqcli deploy config.yaml --dry-run       # 5. confirm the plan is now empty
```

Step 5 is not ceremony. An empty plan is the only proof that the file and the
workspace agree, and therefore that the file is now the source of truth. Anything
else means the next person to run `deploy` gets changes they did not write.

**Get the schema into the editor first.** It is the field-level reference, and it
is authoritative in a way prose is not:

```bash
synqcli schema > schema.json
```

Then start every config file with the line that binds it, which gives completion
and validation as you type:

```yaml
# yaml-language-server: $schema=https://schemas.synq.io/synq-monitors/v1/config.schema.json
```

The same schema is served at that URL, so the local copy is optional.

---

## 2. Entity ids: resolve them, never guess them

This is the most common failure, and the one the CLI can help with least. A wrong
id is not a typo it can correct — the asset simply is not there.

An entity id in the YAML is written in one of three forms, and all three are
resolved against the workspace before anything is deployed:

| Form | Example | Resolved by |
|---|---|---|
| Full path, `::` separated | `ch-prod::default::runs` | direct lookup |
| Full path, dot separated | `ch-prod.default.runs` | dots are converted to `::`, then looked up |
| Warehouse coordinates (SQL FQN) | `analytics.public.orders` | matched against known warehouse objects |

The first two are the same thing written two ways. The third is different: it is
what the table is called in the warehouse, and it is matched only for ids the
first lookup did not find.

**Resolution can only succeed for an asset the platform already knows about**, and
only for a table, a view or an equivalent model — a dashboard or a job is not a
monitorable entity, so an id that names one fails the same way a nonexistent one
does.

**Get real ids rather than constructing them.** Two reliable sources:

- `synqcli export --source all out.yaml` and read the `entities[].id` values it
  writes. They are already in the simplified form the config accepts.
- The asset's page in the web app, which shows the same path.

### Reading the failure

```
The following entity IDs could not be resolved:
  - ch-prod.default.runz
```

The id names nothing in this workspace. Either it is misspelled, the asset has not
been ingested yet, or — much more often than it looks — you are pointed at the
wrong workspace. Check `auth status` before you check the spelling.

```
The following entity IDs resolved to multiple entities:
  - orders
      - ch-prod::default::orders
      - bq-prod::analytics::orders
  Please specify more specific entity IDs to resolve to a single entity.
```

A short or coordinate-shaped id matched more than one asset. The fix is to replace
it with one of the full paths listed underneath — they are printed precisely so
they can be pasted back into the file.

**An unresolved id fails the whole run**, including `--dry-run`, and no plan is
printed. Fix every id before expecting to see a plan at all.

---

## 3. The namespace is the reconcile boundary

```yaml
namespace: "team-analytics"
```

`namespace` groups checks so that different teams reconcile independently. It is
the single most consequential field in the file:

- **Deleting is scoped to it.** `deploy` removes checks that this namespace owns
  and the file no longer declares. It never touches another namespace's checks.
- **A missing `namespace` is itself a namespace** — the default one — not an
  absence of grouping.
- **Getting it wrong is not a no-op.** A typo means "this namespace declares
  nothing that it used to own", which is a request to delete, not a request to
  skip.
- Several files may share one namespace. They are parsed together and reconciled
  as one unit, so a plan for a namespace reflects every file that declares it.

`deploy --namespace <name>` filters which namespaces in the parsed files are acted
on; it is repeatable. It does not change what a namespace owns, only which of them
this run processes.

**`export --namespace` is a different thing entirely.** On `export`, `--namespace`
is the value *written into* the generated YAML — a label, not a filter. `export`
has no namespace filter, so it returns everything matching the other scopes and
stamps whichever namespace you named onto all of it. Exporting the whole workspace
under a namespace and deploying that file back is how one config comes to claim
checks that belonged to five others.

---

## 4. `--dry-run` and the confirmation prompt are not the same thing

They read as one thing and are not:

- `--dry-run` computes and prints the plan, then **exits without deploying**. It
  never prompts. Nothing is written.
- Without `--dry-run`, the same plan is printed and then the run **asks**, per
  namespace, whether to apply it. Answering anything other than `y` cancels that
  namespace and moves to the next.
- `--auto-confirm` answers `y` for you. The plan is still printed; nothing is
  reviewing it.

For an agent this matters twice over. **Without `--auto-confirm` a non-interactive
run blocks on the prompt** — it does not proceed, and it does not fail. And
`--auto-confirm` does not skip the review step so much as remove it, which is only
safe when a `--dry-run` of the same file was read first.

The order that is both non-blocking and reviewed:

```bash
synqcli deploy config.yaml --dry-run     # read this output — it is the review
synqcli deploy config.yaml --auto-confirm
```

`--dry-run` and `--auto-confirm` together are accepted and mean `--dry-run`:
nothing is deployed.

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

Give every monitor and test an `id:` of your own, as above. It is optional, but it
is the check's address: with one, editing any other field updates the check and
keeps its history, and without one a good many edits replace it instead. Section 6
is the whole story.

`version: v1beta2` is the current format; `v1beta1` is legacy and exists for
compatibility only. Do not author new files in it.

**Monitor types** watch an asset over time and learn what normal looks like:
`volume`, `freshness`, `field_stats`, `custom_numeric`, `category_distribution`,
`table_stats`, `custom_table_stats`.

**SQL test types** assert something is true right now: `not_null`, `empty`,
`unique`, `accepted_values`, `rejected_values`, `min_max`, `min_value`,
`max_value`, `freshness`, `relative_time`, `business_rule`, `business_query`,
`relationships`. The two that carry SQL you write yourself — `business_rule` and
`business_query` — accept `{{ table }}`, replaced with the anchored table's fully
qualified name. On a `sql_tests` deployment rule that is what makes one query
cover every matched table instead of being copied onto each of them; any other
`{{ name }}` token is rejected on save. See
[SQL tests](README_SQL_TESTS.md).

**Deployment rules** (`deployment_rules`, and `deployment_exclusions` to carve
assets back out) apply checks by *query* rather than by listing assets, so a new
matching asset is covered without editing the file. `type` says what a rule
deploys: `table_stats` deploys monitors, `sql_tests` deploys the SQL tests listed
under its own `tests:`, onto every table or view the selection matches. On a
`sql_tests` rule the schedule, timezone, severity and `save_failures` are
rule-level and govern every test it deploys; a test may override only `severity`.

A `sql_tests` rule deploys each test only onto the matched tables that have every
column it reads, and the deploy plan lists the rest as skipped tests. So a broad
selection is safe to write: a rule carrying three tests covers each table with
the ones that fit it, and the plan says where they landed.

`export` writes `sql_tests` rules back into the `deployment_rules` list, so a
workspace that has them can be adopted with the loop in section 1. A
`table_stats` rule is written as the single-asset `table_stats` monitors it
covers; its query form is authoring-only, so a file that has to keep one must
keep the hand-written version.

Where to look for the fields of any one of them, in order of authority:

1. [the published schema](https://schemas.synq.io/synq-monitors/v1/config.schema.json)
   — every field, every accepted value.
2. [`examples/`](examples/) — one file per monitor type, per test type and per
   pattern, all of them deployable as written.
3. [SQL tests](README_SQL_TESTS.md) for the test types in prose, and
   [Monitors as code](https://docs.synq.io/monitors/monitors-as-code) for what the
   YAML declares and why.

An unknown key on an entity, a monitor or a test is an error, not something
ignored, so a misspelled field there fails the run rather than silently doing
nothing. Deeper inside a single test's own options that check thins out, so a
run that succeeds is not by itself proof that every key landed. Check the plan:
a field that was accepted shows up in the diff for an existing check.

---

## 6. What re-deploying changes, and what it replaces

`deploy` matches a check by its **id**, so what the id is derived from decides
whether an edit is an update or a replacement — and a replacement is not an edit.
The old check is deleted, with its issue history and its error runs, and for a
monitor with the baseline it has learned; a new one is created in its place.

**Write an `id:` of your own and it becomes the check's address:**

| `id:` | What identifies the check |
|---|---|
| a name of your own — `orders_nulls` | the namespace, the entity, the check type, and that name. Nothing else |
| a UUID | itself, verbatim — this is what `export` writes |
| omitted | the check's *content*, which differs per type — see the tables below |

So under an `id:` of your own, everything except the entity and the type is an
attribute you can edit in place: the `metric_aggregation` of a `custom_numeric`,
the column of a `category_distribution`, a test's columns or its SQL, the mode,
the segmentation, the partitioning column. Three things still replace the check, and each has a reason
that will not go away:

- **renaming the `id`** — it is the address, so a new one is a new check;
- **moving the check to another entity** — the API refuses to point an existing
  monitor at a different asset;
- **changing its `type`** — a `volume` monitor that becomes a `freshness` monitor
  measures something else, so neither its baseline nor its history means anything
  under the new one.

`id:` is optional, and **without one the content is the identity** — which is what
lets two checks of the same type sit on one entity without either of them naming
an id. The cost is that editing a field in that content replaces the check. The
per-type content is: for a `custom_numeric` its `metric_aggregation`, for a
`category_distribution` its column, for a test its columns and, for a
`business_rule` or `business_query`, the SQL; plus, for every monitor, the mode,
the time-partitioning expression and the segmentation expression.

Two `field_stats` monitors on one entity are the exception: the content
derivation does not include their `columns`, so a pair that differs only there
gets the same id and the deploy fails on the duplicate. Give at least one of them
an `id:`.

So, for monitors:

| Edit | With an `id:` | Without one |
|---|---|---|
| `severity`, `sensitivity`, thresholds, `timezone`, schedule, `name`, `description`, `filter`, `category` | update in place | update in place |
| the `metric_aggregation` of a `custom_numeric` | update in place | replace |
| the column of a `category_distribution` | update in place | replace |
| `time_partitioning_column`, segmentation | update in place | replace |
| switch `anomaly_engine` ⇄ `fixed_thresholds` | update in place | replace |
| the `columns` of a `field_stats` | update in place | update in place |
| move the monitor to a different entity | replace | replace |
| change the monitor `type` | replace | replace |
| rename the `id` | replace | — |

And for tests:

| Edit | With an `id:` | Without one |
|---|---|---|
| `severity`, recurrence, evaluators, `name`, `description`, `save_failures`, `category` | update in place | update in place |
| the `values` of an `accepted_values`, the bounds of a `min_max` | update in place | update in place |
| the `column`/`columns` a test names | update in place | replace |
| the SQL of a `business_rule` or `business_query` | update in place | replace |
| move the test to a different entity | replace | replace |
| change the test `type` | replace | replace |
| rename the `id` | replace | — |

A deployment rule works the same way, and it matters more there than anywhere
else. **Deleting a `sql_tests` rule deletes every test the rule deployed**,
together with those tests' check entities and their history. Nothing brings
them back — not the tests, not their history.

Renaming a rule that carries no `id:` replaces it: the old rule is deleted
with everything it deployed, and the next scheduled sync builds the tests
again from scratch under the new name. A monitor rule is gentler — its
monitors survive as long as some rule still covers the asset — but the
rule's own id changes, so every link and every `id=` someone held goes
stale.

| `id:` on a rule | What identifies the rule |
|---|---|
| a name of your own | the namespace and that name, so the rule's `name:` becomes editable |
| a UUID | itself, verbatim — `export` writes one |
| omitted | the namespace and the rule's `name:`, so a rename replaces it |

That holds for both kinds of rule and for a `deployment_exclusions` entry.
Everything else — the `resolver_ql` it selects with, and whatever the kind carries
to describe the checks it deploys — is an attribute. The two kinds carry different
attributes, so the table names which fields belong to which; a field from the
other kind is rejected, not ignored.

| Edit | With an `id:` | Without one |
|---|---|---|
| `resolver_ql` — broaden, narrow, or just reformat | update in place | update in place |
| `severity` — either kind | update in place | update in place |
| `metrics`, `sensitivity`, `delay_model` — `table_stats` only | update in place | update in place |
| the `tests:` a rule deploys, plus `schedule`, `timezone`, `save_failures`, `keep_removed_tests` — `sql_tests` only | update in place | update in place |
| `name` | update in place | replace |

Updating a rule in place resyncs it: checks appear on assets the selection now
matches, go away from assets it no longer matches, and assets that match both
before and after keep the checks they already had, with their history.

One rule kind narrows the field: an entity-anchored `table_stats` monitor becomes
a rule on that one asset, and there is one such rule per asset — so it is
addressed by the asset, and its `id:` accepts only the UUID `export` writes for
it, adopting that rule. A name of your own there is rejected rather than
reinterpreted. Converting such a monitor into a query rule under
`deployment_rules` is a different rule selecting differently, not an edit: expect
a create and a delete.

A rule and an exclusion may share a name, or an `id:`: they are two different
things and each keeps its own identity.

**`--dry-run` names a monitor replacement**, on one line — `Replaced, id changed:
<old> → <new>`, with what it discards and the fields that differ — and counts it as
neither a create nor a delete. Anything on one asset and of one type that leaves as
one monitor and arrives as one monitor is paired that way: a renamed `id:`, an edit
to a monitor carrying no `id:`, or two unrelated monitors that happen to share the
asset and the type. **Read the diff, not the word "Replaced"** — it is what says
which of the three you are looking at, and the line above prints the new monitor's
name, so a `name:` in the diff means the one going away had a different one. An
entity move and a `type:` change stay a delete plus a create in the two sections of
the plan, because both really are a different check. A replaced **test** or **rule**
is a delete plus a create too, so read those two sections when you have edited an
`id:` or a rule's `name:`.

"Update in place" above means the check keeps its id and its history. A few of
those updates keep the check but still discard what it has learned, which is the
next section.

---

## 7. Resets and re-runs: updates that are not free

Some in-place updates keep the check but disturb what it has. The plan flags
these. What they cost is not the same for a monitor and for a test, and the two
used to share one word.

**A monitor reset discards the learned baseline.** It has no anomaly baseline
until it has collected enough history again, and during that window it is not
really watching anything. A monitor resets when its timezone changes, its mode
changes (`anomaly_engine` ⇄ `fixed_thresholds`), its schedule changes (type, or
the time of day / minute of hour, or the delay), its `metric_aggregation` or a
`category_distribution`'s column or `top_k_limit` changes, its time-partitioning
expression or interval changes, or its segmentation changes.

The mode, the `metric_aggregation`, a `category_distribution`'s column, the
partitioning expression and the segmentation reach a reset only when the monitor
carries an `id:`. Without one they are part of its identity, so the same edit
replaces the monitor outright, which is strictly worse.

**A test re-run discards nothing.** A test has no learned state, so what the plan
calls out is that the test will run again: it is re-armed and runs immediately
rather than at its next scheduled slot. A test is re-armed when its recurrence,
its severity, its evaluators, or its template body change — the last one meaning
any test-specific field, including the columns it names, the `values` of an
`accepted_values` or the bounds of a `min_max`. A test with no `schedule:`, run by
something outside `synqcli`, is not re-armed at all; it runs at its next external
trigger.

What a test *does* lose is a replacement, from § 6: a new id means a new check
entity, so its issues and its error runs are gone. That is the thing an `id:`
buys, not the re-run.

There is no flag to suppress a reset and no way to deploy the change without it.
The choice is to accept it or to leave the field alone, so the useful thing an
agent can do is notice it in the plan and say so before confirming, rather than
discover it afterwards.

---

## 8. Refusals, and what each one means

### "managed by other configs" — the deploy is refused

```
🚫 2 monitors managed by other configs.
   - Monitor ID: 7f3b…, Managed by namespace: team-platform
```

This is the ownership gate, and it is the only condition that refuses a deploy
outright. It means your file declares a check that already exists and belongs to a
different namespace. Two configs claiming one check would make every deploy fight
the other, so the tool stops instead of picking a winner.

**It aborts the entire run, not just that namespace**, and it aborts `--dry-run`
too — so no plan is printed for any namespace until it is resolved.

The recovery, in the order to try it:

1. **Confirm it is not just the wrong namespace on your side.** The message names
   the namespace that owns the check. If that is the namespace your file should
   have declared, fix `namespace` and the conflict disappears.
2. **Stop declaring it.** If it genuinely belongs to the other config, remove it
   from yours. This is the right answer far more often than transferring it.
3. **Transfer it deliberately**, if it really should move: remove it from the
   owning config and deploy that config first, then add it to yours and deploy
   again. Between the two deploys the check does not exist — it is deleted and
   recreated, so it loses its history. There is no in-place handover.

Deciding which config should own a check is a judgement about how a team works, so
it is one to put to whoever owns the config, not one to resolve by editing the
other team's file.

### "Managed by API (No Config)" and monitors created in the app — not refused

Both appear in the plan under their own headings, and neither blocks anything.
**They are adoptions:** a check created in the web app, or created over the API
without a namespace, is taken over by your namespace when your config declares it.
From then on it is yours, and a later deploy that stops declaring it will delete
it.

That is usually what was wanted, but it is silent and it is one-way. If a check was
deliberately maintained in the UI, declaring it in a config takes it away from
whoever maintains it there.

### Everything else that stops a run

| Refusal | What it means |
|---|---|
| An entity id could not be resolved | § 2 — the asset is unknown, or you are on the wrong workspace |
| An entity id resolved to multiple entities | § 2 — use one of the full paths it lists |
| Duplicate checks | Two entries in the file reduce to the same identity. Give them different columns, expressions or ids |
| A validation error | The message names the field. Check it against the schema |
| A field is rejected as unknown | Unknown keys are errors, not warnings. Usually a misspelling or a v1beta1 field in a v1beta2 file |
| A parse error in a file you did not mean to deploy | `deploy` with no arguments walks the working directory for every `.yaml` and `.yml`. Pass files explicitly |

All of these are collected across every file and namespace and reported together,
then the run exits without deploying anything. That is deliberate: fix them in one
pass rather than one at a time.

---

## 9. Running it non-interactively

```bash
synqcli deploy config.yaml --dry-run
synqcli deploy config.yaml --auto-confirm
```

- **Credentials**, in precedence order: `--client-id` / `--client-secret` flags;
  `QUALITY_CLIENT_ID` / `QUALITY_CLIENT_SECRET` (or a pre-issued `QUALITY_TOKEN`);
  a `.env` file in the working directory; a cached browser login. Client
  credentials beat a cached login, so CI is unaffected by whoever logged in last.
- **The deployment** is named, not spelled out: `--region eu|us|au`, or
  `--endpoint` for a self-hosted one. `synqcli auth login` remembers the choice and
  `synqcli auth use <region>` switches it, so later commands need no flag. **A
  wrong region is the most expensive mistake available here** — it reconciles a
  workspace that does not have your checks declared, and reconciling means
  deleting.
- **Confirm the target before writing.** Every `deploy` and `export` prints the
  workspace it connected to, before it does anything. Read that line.
- **An execution log** in JSONL is written when `--log-file` or `$QUALITY_LOG_FILE`
  is set: one record per phase, per namespace result and per error. This is the
  machine-readable version of the run, and it is much better to parse than the
  formatted output.
- **Exit codes:** `0` means the command did what it said, `1` means it did not — a
  failed deploy or export, and also a mistyped flag, an unknown command, a missing
  argument or an unusable `--region` / `--endpoint`. So a CI step that checks the
  status also catches an invocation an older `synqcli` does not understand, rather
  than passing having deployed nothing.
- **A cancelled namespace is not a failure.** Declining the prompt exits `0`.
  `--dry-run` also exits `0` whether or not there were changes. If a CI gate needs
  "nothing to deploy", read the log file rather than the status.

---

## 10. `advisor` — propose tests for an asset

`advisor` asks a model to suggest tests for an entity and can write them straight
into a config:

```bash
synqcli advisor \
  --entity-id "ch-prod::default::orders" \
  --instructions "Suggest data quality tests for this table" \
  --output ./suggested
```

`--output` is a **directory**; one file is written per entity, and an existing file
is skipped unless `--force` is passed. `--entity-id` is repeatable. Other flags
worth knowing: `--columns` to narrow the scope, `--instructions-file` for a long
prompt, `--severity` for the default it assigns, `--namespace` for the group it
writes into, and `--connections` to let it profile the warehouse — which is what
lets it propose real `accepted_values` and real bounds instead of guesses.

**Review what it writes before deploying it.** `--output`, then read, then `deploy
--dry-run` is the safe order. It proposes; it does not know what the workspace
already has.

Being specific in the instructions pays. Some that work:

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

- `schema` — free and offline. No credentials, no API call.
- `auth status` — reads local files only. No network.
- `export` — reads the workspace. Cheap. Note that it **refuses to overwrite an
  existing output file**, so give it a fresh path.
- `deploy --dry-run` — resolves every entity id and diffs against the workspace.
  Cheap, and always worth it.
- `deploy` — writes. The cost is the blast radius, not the runtime: it creates,
  updates and **deletes** in one pass.
- `advisor` — a model call per entity, plus warehouse metadata reads and, with
  `--connections`, profiling queries against the warehouse itself. The only command
  here with a per-run cost worth thinking about; scope it with `--entity-id` and
  `--columns` rather than pointing it at everything.
- `upgrade` — replaces this binary with the latest release, verified against the
  release's `checksums.txt`; `upgrade --check` reports what it would do and changes
  nothing. Neither touches the workspace. `synqcli` also mentions a newer release
  on stderr about once a day, and stays silent when an agent is driving it, when
  output is not a terminal, in CI, and in a container — so it never lands in output
  you are parsing. `QUALITY_NO_UPDATE_CHECK=1` turns it off outright.

---

## 12. Never do these

- **Never run `deploy` without reading a `--dry-run` first** on a workspace whose
  config you did not author. Deploy is a reconcile, so an incomplete file is a
  delete.
- **Never guess a `namespace`.** A typo does not scope the run down, it asks for a
  deletion.
- **Never export the whole workspace under a namespace and deploy it back.**
  `export --namespace` labels, it does not filter — see § 3.
- **Never invent a UUID for an `id:`.** A name of your own is the supported way to
  address a check (§ 6); a UUID you made up addresses nothing and, on an
  entity-anchored `table_stats` monitor, adopts a rule that does not exist. Copy a
  UUID only from `export` or from the app.
- **Never assume a rename is a rename.** Check it against § 6 first, and read the
  replacement line and the delete list in the plan.
- **Never self-confirm a monitor reset.** If the plan says a monitor resets, say so
  and let the person decide — the baseline it discards is not recoverable. A test
  that "will run again" discards nothing and needs no such pause.
- **Never point a config at a workspace you have not confirmed.** `auth status`,
  and read the workspace line the command prints.
- **Never edit the schema or the CLI reference by hand.** Both are generated.

---

## 13. When something looks wrong

| Symptom | Cause |
|---|---|
| The plan deletes checks you did not expect | Wrong `namespace`, or a file missing what the namespace already owns — export first |
| The command hangs and prints nothing further | It is waiting on the confirmation prompt. Use `--dry-run`, or `--auto-confirm` |
| A check was recreated instead of updated | Its address changed — a renamed `id:`, another entity, another `type` — or it carries no `id:` and a field of its content changed. § 6 |
| A monitor updated but lost its baseline | A reset — § 7 |
| "managed by other configs" refuses everything | The ownership gate — § 8 |
| An entity id does not resolve | § 2, and check the workspace before the spelling |
| A check created in the app is now in your plan | Adoption, which is expected and one-way — § 8 |
| `deploy` complains about YAML you never wrote | With no file arguments it walks the working directory. Pass files explicitly |
| Nothing deploys and the status is still `0` | The prompt was declined, or `--dry-run` was set |

The full command and flag reference is generated from the CLI itself and published
as the [CLI reference](https://docs.synq.io/monitors/cli). For what the YAML
declares and why, start at
[Monitors as code](https://docs.synq.io/monitors/monitors-as-code); for the test
types in detail, [SQL tests](https://docs.synq.io/monitors/sql-tests) and
[Deployment rules](https://docs.synq.io/monitors/deployment-rules).
