# Implementation guide

From zero to a running static-RLS pipeline. Each step links to where the
detail lives if you need it — this page stays short on purpose.

## 1. Prerequisites

- A Fabric workspace with a Lakehouse attached, and a Power BI semantic model already published to it.
- XMLA Read/Write enabled on the capacity.
- `semantic-link-labs` available in your Fabric environment.
- Two setup notebooks of your own — templates (contract only, no real credentials) in [`setup notebooks/`](../setup%20notebooks):
  - `Connections` — Lakehouse path variables. Required.
  - `Email notifications` — Microsoft Graph email sending. Only needed if you want alert emails.

## 2. Load the library

```python
%run Connections            # your own notebook
%run Email notifications    # your own notebook, optional
%run RLS_Management_Functions
```

## 3. Prepare your RLS source table

One Spark DataFrame, at minimum:

| Column | Required |
|---|---|
| `Username` | Always — UPN email |
| one column per dimension you're securing | e.g. `CountryCode`, `Brand` |

```python
df = spark.read.format("delta").load(f"{path_rls}rls_source_table")
```

If any restriction only applies to certain kinds of records (like `RLS_SalesCloud`'s example), do that prep here too — see [`CONFIGURATION_REFERENCE.md`](./CONFIGURATION_REFERENCE.md#concatenated-roles) for the pattern.

## 4. Write your config

Start with `standard` roles — one per dimension:

```python
config = {
    "Country": {
        "table": "DimCountry",
        "prefix": "RLS_Country",
        "column": "CountryCode",
        "source_column": "CountryCode"
    },
}
```

Add other types only where the business rule needs them — full reference
in [`CONFIGURATION_REFERENCE.md`](./CONFIGURATION_REFERENCE.md).

## 5. If you used `concatenated_from`, preprocess first

```python
df, config = preprocess_concatenated_roles(df, config)
```

Skip this step entirely if your config has no `concatenated_from` entries.

## 6. Create the roles

```python
create_or_replace_roles_v3(config, dataset, workspace, global_filters=[], rls=df)
```

Safe to re-run — creates new roles, refreshes the DAX filter on existing ones.

## 7. Sync membership, through the audit wrapper

```python
result, summary = run_with_audit(
    update_members_delta,
    config=config,
    dataset=dataset,
    workspace=workspace,
    env="prod",
    rls=df,
    chunk_size=200,
    audit_path=path_control,              # from Connections
    alert_to="your-team@yourcompany.com",  # omit to skip email alerts
)
```

Never call `update_members_delta(...)` directly in a scheduled run — only
`run_with_audit()` writes the snapshot, change log, and run summary.

## 8. Clean up unused roles

```python
drop_unused_roles(config, dataset, workspace, rls=df)
```

Run whenever values disappear from your source data — not required every
execution.

## 9. Schedule it

A daily Fabric pipeline: sync Active Directory → refresh the RLS source
table → invoke one notebook per secured model, in parallel. See the
diagram in the main [README](../README.md#running-in-production).

## 10. Check it worked

```sql
SELECT * FROM rls_run_summary ORDER BY run_timestamp DESC LIMIT 5
```

A near-zero `delta` most days is normal. A large `delta_pct` — up or down —
is worth a manual look even if it wasn't your first run.
