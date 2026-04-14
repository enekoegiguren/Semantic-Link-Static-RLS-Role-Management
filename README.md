# Semantic Link — Static RLS Role Manager

**Fabric Semantic Link Developer Experience Challenge 2026**  
**Author:** eguigurene

Automate the full lifecycle of static Row Level Security (RLS) roles in a Power BI Semantic Model — creation, replacement, member assignment, and cleanup of unused roles — using a single Microsoft Fabric notebook.

Built on [`semantic-link-labs`](https://github.com/microsoft/semantic-link-labs) and the Tabular Object Model (TOM), this notebook replaces the need for manual role management or Dynamic RLS security tables — both of which become painful at scale.

---

## The problem

When a Power BI model has complex, multi-dimensional security requirements — multiple business units, territories, countries, brands — the typical solution is **Dynamic RLS**: a security table in the model filtered by `USERPRINCIPALNAME()` at query time.

This works. Until it doesn't.

On a fact table with 10M+ rows and a security table with hundreds of combinations, Dynamic RLS evaluates on **every query, for every user**. Reports slow down. Users complain. And asking your Azure AD admin to manually create 500 security groups is not a conversation anyone wants to have.

**Static RLS roles are evaluated once at connection time** — VertiPaq resolves a fixed DAX expression instantly and propagates it through relationships. The bottleneck disappears.

The problem has always been: who creates 500 static roles? And who cleans them up when the business changes? This notebook does both.

---

## How it works

```
RLS table (Lakehouse)
        │
        ▼
Spark DataFrame (rls)
        │
        ├─ build_dax_filters()       → DAX expression per role per table
        ├─ create_or_replace_roles() → push roles + filters to semantic model
        ├─ add_members_to_roles()    → assign UPN emails to each role
        └─ drop_unused_roles()       → remove roles no longer in the RLS table
                                            │
                                            ▼
                               Published Semantic Model
```

1. You define a `config` — one entry per dimension to secure (e.g. `Country`, `Brand`, `Company`)
2. You define `global_filters` — filters applied to every role on their own fixed table (e.g. `Is_Consolidated = "Consolidated"` on `DT_Customer`)
3. The notebook reads distinct values from your RLS DataFrame, generates a role per value, applies the DAX filters, assigns members from the `Username` column, and optionally cleans up any roles that no longer have matching data

---

## Design principles

### Filter on dimension tables (DT\_\*), not fact tables (FT\_\*)

Filtering directly on a fact table with millions of rows means the engine scans it on every query for every user. Filtering on a dimension table lets the relationship engine propagate the filter automatically — which is exactly what VertiPaq is optimized for.

```
DT_Customer [Country = "Australia"]  →  propagates through relationship  →  FT_Sales
```

### Static over dynamic

| | Dynamic RLS | Static RLS (this notebook) |
|---|---|---|
| Evaluated | Every query, every user | Once at connection time |
| Security table in model | Yes — adds model complexity | No |
| Maintenance | Manual or scripted | Scripted, re-runnable |
| Performance on 10M+ rows | Slow | Fast |
| 500 combinations | Very slow | Fine |

### Global filters are always applied to every role

`global_filters` are applied to their own fixed table regardless of what the role-specific filter uses. If both land on the same table, they are merged with `&&` into a single `set_rls` call — because `set_rls` replaces, not appends.

### Members are saved one at a time

A single invalid UPN causes `SaveChanges()` to reject the entire batch. By opening a fresh TOM connection per member, a bad UPN never blocks a valid one. Failed members are collected and exported as a JSON report to your Lakehouse `Files` folder.

### Unused roles are cleaned up safely

`drop_unused_roles()` only ever removes roles whose name matches a prefix defined in `config` — roles created outside this notebook (e.g. `Admin`, `ReadOnly`) are never touched.

---

## Requirements

- Microsoft Fabric workspace with a Lakehouse attached to the notebook
- Power BI Semantic Model published to the workspace
- XMLA Read/Write enabled on the Fabric capacity
- `semantic-link-labs` (installed in the notebook via `%pip install`)
- RLS source table with at minimum: `Username` (UPN email) + dimension columns

---

## Configuration

### `global_filters`

Filters applied to **every** role, each on their own fixed table. Multiple filters supported.

```python
global_filters = [
    {
        "table":  "DT_Customer",    # Table in the semantic model
        "column": "Is_Consolidated",
        "value":  "Consolidated"
    },
    # {
    #     "table":  "DT_Product",
    #     "column": "Is_Active",
    #     "value":  "True"
    # },
]
```

### `config`

One entry per dimension. The key must match a column name in your RLS DataFrame.

```python
config = {
    "Country": {
        "table":  "DT_Customer",     # Dimension table — not the fact table
        "prefix": "RLS_Country",     # Roles will be named RLS_Country_AU, etc.
        "column": "Country"          # Column in the semantic model table
    },
    "Brand": {
        "table":  "DT_Product",
        "prefix": "RLS_Brand",
        "column": "Brand"
    },
    "Company": {
        "table":  "DT_Company",
        "prefix": "RLS_Company",
        "column": "Company"
    },
}
```

### RLS DataFrame (`rls`)

Must contain:

| Column | Description |
|--------|-------------|
| `Username` | UPN email (`user@domain.com`) |
| `Country` | One column per `config` key |
| `Brand` | ... |
| `Company` | ... |

---

## Usage

```python
# Step 1 — always safe to re-run (idempotent)
create_or_replace_roles(
    config         = config,
    dataset        = dataset,
    workspace      = workspace,
    global_filters = global_filters,
    rls            = rls,
    config_keys    = None,   # None = all; or ["Country"] to run a subset
)

# Step 2 — optional, run separately once roles are confirmed
failed = add_members_to_roles(
    config       = config,
    dataset      = dataset,
    workspace    = workspace,
    rls          = rls,
    username_col = "Username",
    config_keys  = None,
)

# Step 3 — optional, run when values have been removed from the RLS table
dropped = drop_unused_roles(
    config      = config,
    dataset     = dataset,
    workspace   = workspace,
    rls         = rls,
    config_keys = None,
)
```

All three functions accept a `config_keys` parameter to process only a subset of the config — useful when iterating or debugging a specific dimension.

---

## Function reference

| Function | Purpose | Safe to re-run |
|----------|---------|----------------|
| `build_dax_filters()` | Builds DAX filter expressions per role | — |
| `create_or_replace_roles()` | Creates or replaces roles + DAX filters in the model | ✅ Yes |
| `add_members_to_roles()` | Assigns UPN emails to roles, one at a time | ✅ Yes |
| `drop_unused_roles()` | Removes roles whose value no longer exists in the RLS table | ✅ Yes |

---

## Output

- Roles created in the semantic model: `RLS_Country_AU`, `RLS_Brand_Nike`, `RLS_Company_Acme`, ...
- Each role has DAX filters applied per table — role-specific and global
- Members assigned from the `Username` column in the RLS DataFrame
- Failed members exported to `Files/rls_failed_members/` in the Lakehouse as JSON
- Unused roles removed from the model when `drop_unused_roles()` is run

---

## File structure

```
2026_SemanticLink_eguigurene_StaticRLSRoleManager.ipynb   ← main notebook
README.md
```

---

## Related

- [semantic-link-labs](https://github.com/microsoft/semantic-link-labs) — the library powering the TOM connection
- [TMDL view in Power BI Desktop](https://learn.microsoft.com/en-us/power-bi/transform-model/desktop-tmdl-view) — the broader TMDL ecosystem this fits into
- [Semantic model connectivity with XMLA endpoint](https://learn.microsoft.com/en-us/power-bi/enterprise/service-premium-connect-tools)
- [Fabric Semantic Link Developer Experience Challenge](https://community.fabric.microsoft.com/t5/Power-BI-Community-Blog/Announcing-the-Fabric-Semantic-Link-Developer-Experience/ba-p/5139639)

---

## Contributing

Issues and PRs welcome. If your security model has patterns not covered by the current `config` structure — multiple fact table filters, OLS, cross-workspace models — open an issue and let's discuss.

---

## License

MIT
