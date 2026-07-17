# Configuration reference

This is a reference for what a config entry can do, not a walkthrough of one
model or the other — every key below is available in both Sales and Sales
Cloud, and every difference between the two lives entirely in which keys get
used and what the dataframe looks like when they're evaluated.

A config is a Python dict of dicts. Each top-level key is an arbitrary name
you choose; each value is a settings object that tells `create_or_replace_roles_v3`
how to turn rows of your RLS source table into semantic-model roles.

## The anatomy of one entry

![JSON to DAX](./assets/rls_json_to_dax.png)

The important thing this diagram shows: **the config doesn't contain DAX.**
Nothing in the JSON is a filter string you wrote by hand — every filter is
*generated* from a small number of structural keys. That's what makes the
same engine reusable: you're never writing DAX, you're describing shape, and
the engine writes the DAX for you, consistently, every time.

## From one entry to many roles

![One config, many roles](./assets/rls_role_explosion.png)

Neither entry above declares its roles anywhere — the role count comes
entirely from what's actually in the source data on a given run. Five
countries in the data today means five roles; a sixth tomorrow means a
sixth role next run, with no config change required.

## Standard roles — one role per unique value

```python
"Country": {
    "table": "DT_Country",
    "prefix": "RLS_Country",
    "column": "CountryCode",
    "source_column": "CountryCode"
}
```

| Key | What it controls |
|---|---|
| `table` | Which table in the semantic model gets the DAX filter |
| `prefix` | The role name prefix — final name is `{prefix}_{sanitized value}` |
| `column` | The column name *in the model* being compared |
| `source_column` | The column name *in the RLS source table* to read unique values from — only needed if it differs from the config key |

**Generated DAX**, one per distinct value found in `source_column`:
```dax
[CountryCode] = "ES"
```

**Impact:** if the source table has 238 distinct countries, this produces
238 roles, each with a single equality filter. Cardinality of the *values*
determines the number of *roles* — not the number of users. A user with
access to 200 of those 238 countries costs nothing extra at query time,
because the filter is resolved via role membership, not evaluated per row.

## Concatenated roles — combining two or more columns into one condition

```python
"CompanyCode_Orders": {
    "prefix": "RLS_DocType7_CompanyCode",
    "concatenated_from": {
        "CompanyCode": {
            "table": "FT_Sales",
            "column": "CompanyCode"
        },
        "DocType_CompanyCode": {
            "table": "FT_Sales",
            "column": "DocType",
            "is_numeric": True
        }
    }
}
```

| Key | What it controls |
|---|---|
| `concatenated_from` | A dict of named sources, each with its own `table` + `column` |
| `is_numeric` (per source) | Whether the value is compared unquoted. Required whenever the target column is Integer/Whole Number in the model — quoting a number produces a DAX type-comparison error |

Every source in `concatenated_from` is required — a role only gets created
for combinations where **all** sources have a non-blank value in the same
row. Two sources pointing at *different tables* is valid: each becomes its
own condition, and they're AND-ed together in the final filter regardless of
which table they're on.

**Generated DAX**, one per distinct combination of values:
```dax
([CompanyCode] = "SALTO HQ") && ([DocType] = 7)
```

**Impact:** cardinality multiplies. 100 company codes × 2 applicable
DocTypes (if a code applied to two document types) would be up to 200 roles
— in practice fewer, because most values only ever co-occur with one
DocType in the real data. This is why the dataframe-explosion step exists:
it's what decides *which* combinations are even possible before the role
generator ever runs, rather than generating every mathematically possible
pairing.

## `extra_filter` — a fixed condition, without concatenation

```python
"IndustryCode_OppsPipeline": {
    "table": "FT_Sales",
    "prefix": "RLS_DocType478_IndustryCode",
    "column": "IndustryCode",
    "source_column": "IndustryCode",
    "extra_filter": "[DocType] IN {4, 8}"
}
```

Where `concatenated_from` builds a condition out of two *columns whose
values vary*, `extra_filter` is for a condition that's the *same for every
role in this config entry* — a static string, AND-ed onto every generated
filter.

**Generated DAX**, one per distinct `IndustryCode` value:
```dax
([DocType] IN {4, 8}) && ([IndustryCode] = "Retail")
```

**When to use which:** if the fixed part never changes across the whole
config entry, `extra_filter` is simpler and doesn't need a second dataframe
column. If the fixed part itself varies by row (as with `CompanyCode` only
applying to Orders while `IndustryCode` applies to Opportunities/Pipeline),
that's the `concatenated_from` + exploded-dataframe pattern instead.

## `filter_template` — a custom DAX shape, last resort only

```python
"filter_template": 'CONTAINSSTRING(";" & [Fk_Brand] & ";", ";{value};")'
```

⚠️ **Avoid this pattern if at all possible.** `CONTAINSSTRING` forces a
row-by-row, non-indexed string scan instead of VertiPaq's normal indexed
filtering — meaningfully slower at scale. Worse, an unguarded
`CONTAINSSTRING` risks partial-string matches (a role scoped to `"5"` can
also match `"15"`, `"105"`, anything containing that substring), which is a
real access-control bug, not just a performance one. The delimiter-wrapped
version shown above (`";" & column & ";"`) avoids the substring-leak
specifically, but the performance cost remains.

**The preferred fix, almost always available:** if a column is
multi-valued because one fact row can belong to several dimension values,
normalize it — split it into a bridge table with **one row per
fact-dimension pair**, in Power Query or upstream ETL, and go back to a
plain `standard` role (`= "value"`, resolved by relationship, indexed). See
the `ProductLine` entry in [CONFIG_TO_ACCESS_EXAMPLE.md](./CONFIG_TO_ACCESS_EXAMPLE.md)
for the worked version of this — the "multi-valued" problem and the
"CONTAINSSTRING" fix are two separate questions, and normalizing the data
solves the first without needing the second at all.

`filter_template` still exists for the rare case where no bridge table is
feasible and the DAX shape genuinely can't be expressed as a plain equality
— treat it as an explicit trade-off, not a default choice.

## `special: "full_access"` — no filter at all

```python
"FullAccess": {
    "table": None, "prefix": "RLS_FullAccess", "column": None,
    "special": "full_access",
    "source_column": "FullAccess",
    "source_value": "ALL"
}
```

One role, no DAX filter on any table. Membership: every user whose
`source_column` equals `source_value`. Because it's a single unconditional
role, its members see everything on every table it isn't itself filtered on
— by design, it's meant to override everything else for the people who have it.

## `special: "consolidated"` — opt-in, flag plus blank-check

```python
"Consolidated": {
    "table": "Customers",
    "prefix": "RLS_Consolidated",
    "special": "consolidated",
    "fixed_filter": '[Is_Consolidated] = "Consolidated"',
    "source_column": "Is_Consolidated",
    "source_value": "Consolidated",
    "ignore_cols": ["PK", "Username", "FullAccess", "model_upn"]
}
```

| Key | What it controls |
|---|---|
| `fixed_filter` | The single DAX filter every member of this role gets — same for everyone |
| `source_column` / `source_value` | The flag a user must match to be considered for this role |
| `ignore_cols` | Columns excluded from the "must also be blank" check |

Membership requires **both**: `source_column == source_value`, **and** every
column not listed in `ignore_cols` is blank for that user. This is a
two-factor eligibility check — the flag alone isn't enough, the framework
independently verifies the user has no other restrictions either.

## `special: "consolidated_default"` — everyone, except named exceptions

```python
"Unrestricted_OppsPipeline": {
    "table": "FT_Sales",
    "prefix": "RLS_DocType478_Unrestricted",
    "special": "consolidated",
    "fixed_filter": "[DocType] IN {4, 8}",
    "source_column": "Is_Unrestricted_478",
    "source_value": "True",
    "ignore_cols": list(df.columns)
}
```

The inverse eligibility model from plain `consolidated`: instead of "flag AND
blank-check", setting `ignore_cols` to **every column in the dataframe**
neutralizes the blank-check entirely (there's nothing left to check), so
membership depends on the flag alone. This is the pattern used when the flag
was already computed correctly upstream — aggregated across every row a
user has, with blanks handled deliberately — and re-running a generic
blank-check on top of it would only risk contradicting a calculation that
was already right.

A separate, distinct `consolidated_default` type also exists for the
opposite shape of problem — "grant to everyone except members of another
named role" — using `exclude_special_keys` to reference another config
entry's `source_column`/`source_value` directly, without needing to check
real role membership in the model.

## What ties it all together: OR across roles

None of the primitives above know about each other. A user typically ends
up a member of several roles simultaneously — one from a standard role, one
from a concatenated role, one from a consolidated role — and Power BI unions
all of them per table. The final access pattern is never written down
anywhere in the config; it's an emergent property of *which* roles a given
user's data happens to put them in.

![OR composition](./assets/rls_or_composition.png)

Practically, this means extending the security model almost never means
editing an existing config entry — it means adding one more entry, one more
thing a user can independently qualify for. The primitives stay exactly as
generic as they started; only the *set of roles in play* grows.
