# Connections — template

This project expects a notebook named `Connections`, `%run` before
`RLS_Management_Functions`, in the two example notebooks
(`RLS_Sales.ipynb`, `RLS_SalesCloud.ipynb`). It only needs to define
Lakehouse path variables — nothing here is RLS-specific, it's the same kind
of setup notebook most Fabric projects already have.

Fill in the values for your own tenant/workspace. **Do not commit this file
with real workspace names, IDs, or paths to a public repo** — treat it the
same way you'd treat a `.env` file.

## Exactly what the example notebooks use

Checked directly against the two example notebooks — these four variables,
nothing more:

| Variable | Used by | For |
|---|---|---|
| `path_rls` | `RLS_Sales.ipynb` | Loading the Sales RLS source table |
| `path_silver` | `RLS_SalesCloud.ipynb` | Loading the Sales Cloud RLS source table |
| `path_bronze` | Both | Loading the AD Users table (UPN validation) |
| `path_control` | Both | `audit_path=` — where `run_with_audit()` writes the audit tables |

```python
# ── Workspace / lakehouse identifiers ──────────────────────────────────────
workspace_control = "<your control workspace name>"
lakehouse_control  = f"{workspace_control}_LH"

# ── Path variables — one per lakehouse layer your RLS source data lives in ─
# Adjust workspace/lakehouse names per layer to match your own architecture;
# this assumes a medallion setup where bronze/silver/rls are separate
# lakehouses, and a dedicated "control" lakehouse for audit output.
path_bronze = "abfss://<bronze_workspace>@onelake.dfs.fabric.microsoft.com/<bronze_lakehouse>.Lakehouse/Tables/"
path_silver = "abfss://<silver_workspace>@onelake.dfs.fabric.microsoft.com/<silver_lakehouse>.Lakehouse/Tables/"
path_rls    = "abfss://<rls_workspace>@onelake.dfs.fabric.microsoft.com/<rls_lakehouse>.Lakehouse/Tables/"

path_control       = f"abfss://{workspace_control}@onelake.dfs.fabric.microsoft.com/{lakehouse_control}.Lakehouse/Tables/"
path_control_files = f"abfss://{workspace_control}@onelake.dfs.fabric.microsoft.com/{lakehouse_control}.Lakehouse/Files/"
```

## What this notebook does NOT need to provide

`connect_semantic_model` (from `semantic-link-labs`), `pyspark.sql.functions`
(`F`), `pyspark.sql.types`, `re`, `math`, and the `env` variable are already
defined inside `RLS_Management_Functions.ipynb` itself.
