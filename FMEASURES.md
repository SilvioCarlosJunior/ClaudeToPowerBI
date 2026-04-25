# Table `fmeasures` — Granderson Analytics model

Fact table. Each row = one property (`PropertyID`) for one month (`StartAt`).

## ⚠️ Directive for Claude

When answering any user question about data from this table, **use only the columns and metrics listed in this document**. Any other column or measure that exists in `fmeasures` (even if it appears referenced inside the DAX formulas below) is **not considered valid** and must not be referenced in answers, in suggested DAX queries, in calculation explanations, or in diagnostics.

If a user's question can only be answered with items outside this list, **say so explicitly** instead of improvising with undocumented columns.

---

## ✅ Production-validated columns

### Keys and context

| Column | Type | Origin | Description |
|---|---|---|---|
| `PropertyID` | String | physical | Unique property identifier. Relationship key to `dimProperties`, `Profit&Loss`, `physical_occ`, etc. |
| `Name` | String | physical | Property name. |
| `StartAt` | DateTime | physical | Start date of the reference month. Always use it as the time filter for this fact table. |
| `Properties` | String | **calculated** | `RELATED(dimProperties[Properties])` — pulls the property name from the dimension. |
| `GrupoName` | String | **calculated** | Name of the group/fund the property belongs to (DST III, Midwest, FUND I, etc.). Filters `dimPropertiesGroup` by `PropertyID` and `GroupName_ofc = TRUE`, returning the name with the 2-char prefix stripped. **This is the discriminator used by the `_*_final` and `*_RemovedAccounts` columns to choose between "full value" and "value with removed accounts".** |

### Monthly P&L (values for the month at `StartAt`)

| Column | Type | Origin | Description |
|---|---|---|---|
| `TotalIncome` | Double | physical | Total revenue for the month. |
| `TotalNonIncome` | Double | physical | Non-operating income for the month. |
| `TotalNonExpense` | Double | physical | Non-operating expenses for the month. |
| `TotalExpense_RemovedAccounts` | Double | **calculated** | Total expense **adjusted** by removing accounts flagged with `Paginated_Report_GLAccount_filter = 0`. The adjustment is applied **only for the `DST III`, `Midwest`, and `FUND I` groups**; for all other groups it returns `TotalExpense` unchanged. **Use this column instead of `TotalExpense` whenever the calculation must respect the removed-accounts rule.** |
| `_NOI_final` | Double | **calculated** | Final monthly NOI. For `DST III`, `Midwest`, and `FUND I` it uses `NOI_RemovedAccounts`; for all other groups it uses `NOI`. **This is the canonical NOI column for monthly analysis.** |

### T12 (trailing 12 months)

| Column | Type | Origin | Description |
|---|---|---|---|
| `_T12NOI_final` | Double | **calculated** | Final trailing-12-months NOI. Same rule as `_NOI_final`: `T12NOI_RemovedAccounts` for `DST III`/`Midwest`/`FUND I`, `T12NOI` otherwise. USD currency format. **Use this as the canonical T12 NOI column.** |

### Operational

| Column | Type | Origin | Description |
|---|---|---|---|
| `TotalSites` | Int64 | physical | Total sites/units of the property in the month. |
| `PhysicalOccupancy` | Int64 | physical | Physical occupancy (occupied sites). |
| `EconomicOccupancy` | Int64 | physical | Economic occupancy. |
| `MoveIns` | Int64 | physical | Move-ins in the month. |
| `MoveOuts` | Int64 | physical | Move-outs in the month. |
| `EmptySites` | Int64 | **calculated** | Empty sites. Sums `physical_occ[SubUnitsNumberOfActiveUnits]` filtering by `PropertyID`, `DateD = StartAt`, and `UnitTypesName = "Empty Site"`. |

---

## 📐 DAX formulas (calculated columns)

### `Properties`
```DAX
RELATED(dimProperties[Properties])
```

### `GrupoName`
```DAX
VAR NameG =
CALCULATE(
    MAX(dimPropertiesGroup[NameGroup]),
    FILTER(
        dimPropertiesGroup,
        dimPropertiesGroup[PropertyID] = EARLIER(fmeasures[PropertyID])
        && dimPropertiesGroup[GroupName_ofc] = TRUE()
    )
)
RETURN
MID(NameG, 3, LEN(NameG)-2)
```

### `TotalExpense_RemovedAccounts`
```DAX
// Final column for the Expense of all properties
VAR AccountsRemoved =
CALCULATE(
    SUM('Profit&Loss'[AmountCalc]),
    FILTER(
        'Profit&Loss',
        'Profit&Loss'[PropertyID] = EARLIER(fmeasures[PropertyID])
        && 'Profit&Loss'[startdate] = EARLIER(fmeasures[StartAt])
        && 'Profit&Loss'[Paginated_Report_GLAccount_filter] = 0
    )
)
RETURN
IF(
    fmeasures[GrupoName] = "DST III" || fmeasures[GrupoName] = "Midwest" || fmeasures[GrupoName] = "FUND I",
    fmeasures[TotalExpense] - AccountsRemoved,
    fmeasures[TotalExpense]
)
```

### `_NOI_final`
```DAX
// Final column for the NOI of all properties
IF(
    fmeasures[GrupoName] = "DST III" || fmeasures[GrupoName] = "Midwest" || fmeasures[GrupoName] = "FUND I",
    fmeasures[NOI_RemovedAccounts],
    fmeasures[NOI]
)
```

### `_T12NOI_final`
```DAX
// Final column for the T12NOI of all properties
IF(
    fmeasures[GrupoName] = "DST III" || fmeasures[GrupoName] = "Midwest" || fmeasures[GrupoName] = "FUND I",
    fmeasures[T12NOI_RemovedAccounts],
    fmeasures[T12NOI]
)
```

### `EmptySites`
```DAX
CALCULATE(
    SUM(physical_occ[SubUnitsNumberOfActiveUnits]),
    FILTER(
        physical_occ,
        physical_occ[PropertyID] = EARLIER(fmeasures[PropertyID])
        && physical_occ[DateD] = EARLIER(fmeasures[StartAt])
        && physical_occ[UnitTypesName] = "Empty Site"
    )
)
```

---

## 🧠 Key patterns

1. **"RemovedAccounts" rule (DST III / Midwest / FUND I):**
   For these three groups, certain P&L accounts (`Paginated_Report_GLAccount_filter = 0`) must be excluded. The columns `_NOI_final`, `_T12NOI_final`, and `TotalExpense_RemovedAccounts` already encapsulate this rule. **Never recompute NOI manually without applying this logic for these groups.**

2. **Columns with the `_*_final` prefix are the official ones.**
   Several auxiliary columns (`_NOI`, `NOI_RemovedAccounts`, `_T12NetIncome_final`, etc.) exist as intermediate steps. In production, always use the `_final` columns.

3. **Granularity:** one row per `PropertyID` × `StartAt` (month).
