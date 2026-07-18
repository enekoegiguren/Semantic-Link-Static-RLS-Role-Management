# From one config to real user access

This uses a generic "order management" domain — nothing here is tied to any
specific business. The point is to show every config type in one place, and
then trace exactly how different rows of source data turn into different
role memberships for different users.

## The full config, every type at once

```python
config = {

    # ── STANDARD: one role per unique value, no extra condition ─────────────
    "Region": {
        "table": "DimRegion",
        "prefix": "RLS_Region",
        "column": "RegionCode",
        "source_column": "RegionCode"
    },

    # ── CONCATENATED: value column + numeric record-type column, AND-ed ─────
    "Department_RecordTypeA": {
        "prefix": "RLS_RecordTypeA_Department",
        "concatenated_from": {
            "Department": {
                "table": "FactRecords",
                "column": "DepartmentCode"
            },
            "RecordType_Department": {
                "table": "FactRecords",
                "column": "RecordType",
                "is_numeric": True
            }
        }
    },

    # ── extra_filter: a fixed condition shared by every role in this entry ──
    "Category_RecordTypeBC": {
        "table": "FactRecords",
        "prefix": "RLS_RecordTypeBC_Category",
        "column": "CategoryCode",
        "source_column": "CategoryCode",
        "extra_filter": "[RecordType] IN {2, 3}"
    },

    # ── STANDARD, against a bridge/dimension table ───────────────────────────
    # ProductLine can be multi-valued per record (one order can belong to
    # several product lines). Rather than pattern-matching a delimited string
    # at query time (CONTAINSSTRING) — slower, and risks partial-string
    # matches like "east" matching inside "eastern" — DimProductLine is a
    # pre-exploded bridge table: one row per record-productline pair. That
    # turns this back into a plain equality check, resolved by an indexed
    # relationship instead of a row-by-row string scan.
    "ProductLine": {
        "table": "DimProductLine",
        "prefix": "RLS_ProductLine",
        "column": "ProductLineCode",
        "source_column": "ProductLine"
    },

    # ── full_access: no filter, one role, flag-driven ────────────────────────
    "FullAccess": {
        "table": None, "prefix": "RLS_FullAccess", "column": None,
        "special": "full_access",
        "source_column": "AccessLevel",
        "source_value": "ALL"
    },

    # ── consolidated: opt-in flag AND every other listed column blank ───────
    "Consolidated": {
        "table": "DimAccount",
        "prefix": "RLS_Consolidated",
        "special": "consolidated",
        "fixed_filter": '[IsConsolidated] = "Yes"',
        "source_column": "ConsolidatedFlag",
        "source_value": "Yes",
        "ignore_cols": ["Username", "AccessLevel", "UserId"]
    },

    # ── consolidated_default: flag alone decides, no blank-check at all ─────
    "Unrestricted_RecordTypeBC": {
        "table": "FactRecords",
        "prefix": "RLS_RecordTypeBC_Unrestricted",
        "special": "consolidated",
        "fixed_filter": "[RecordType] IN {2, 3}",
        "source_column": "IsUnrestricted_BC",
        "source_value": "True",
        "ignore_cols": list(df.columns)   # every column — flag is the only check
    },
}
```

Seven entries, seven distinct mechanisms, one function (`create_or_replace_roles`)
reading all of them the same way.

## The source data — five users, five different situations

| Username | RegionCode | DepartmentCode | RecordType_Department | CategoryCode | ProductLine | AccessLevel | ConsolidatedFlag | IsUnrestricted_BC |
|---|---|---|---|---|---|---|---|---|
| user.a | NA | | | | | | | True |
| user.b | | Sales | 1 | | | | | False |
| user.c | | | | Support | | | | False |
| user.d | | | | | east | | | False |
| user.d | | | | | west | | | False |
| user.e | | | | | | ALL | | False |

Each row is realistic: most columns are blank for any given user, because
most users only qualify for a handful of the seven mechanisms above.
`user.d` spans two rows on purpose — standard roles read one value per row,
so a user who needs access to several `ProductLine` values simply has
several rows, one per value, rather than one row holding a delimited list.

## What each user actually becomes a member of

| User | Roles assigned | Why |
|---|---|---|
| **user.a** | `RLS_RecordTypeBC_Unrestricted` | `IsUnrestricted_BC = True` is the only populated flag — matches the `consolidated_default`-style entry directly, no other column matters because `ignore_cols` excludes all of them from the blank-check |
| **user.b** | `RLS_RecordTypeA_Department_Sales_1` | `DepartmentCode` and `RecordType_Department` are both populated on the same row — the concatenated role's required combination is met, so one role is generated for that pairing |
| **user.c** | `RLS_RecordTypeBC_Category_Support` | `CategoryCode` is populated — the role gets `[RecordType] IN {2, 3}` from `extra_filter` AND-ed with `[CategoryCode] = "Support"` |
| **user.d** | `RLS_ProductLine_east`, `RLS_ProductLine_west` | Two rows, two values — this user matches **both** the east and west standard roles, one role per row. Each role is a plain equality filter against `DimProductLine`, resolved by relationship, not by parsing a string at query time |
| **user.e** | `RLS_FullAccess` | `AccessLevel = "ALL"` — matches `full_access` directly, no DAX filter on any table |

## What this means when two mechanisms overlap on the same user

None of the five rows above happen to overlap, but it's worth being explicit
about what happens when they do — because that's where the real flexibility
shows up. Imagine a sixth user with **both** `RegionCode = "NA"` *and*
`DepartmentCode = "Sales"` / `RecordType_Department = 1` populated:

- They become a member of `RLS_Region_NA` **and**
  `RLS_RecordTypeA_Department_Sales_1` — two independent roles, from two
  independent mechanisms, neither aware the other exists.
- Power BI unions both, per table. On `FactRecords`, their final visibility
  is `(RegionCode = "NA") OR (DepartmentCode = "Sales" AND RecordType = 1)`
  — a two-term OR built entirely out of role membership, never written as a
  single expression anywhere.

That's the whole mechanism, end to end: **the config never describes a
user's final access — it only describes what one role allows.** A user's
actual access is just the union of every role their row(s) happen to
qualify them for.
