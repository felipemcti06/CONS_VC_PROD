# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is an **IBM Planning Analytics (TM1) FP&A model** — not a software project. There are no build commands, no package manager, no test runner. The repository contains the serialized artifacts of a TM1 server model: cube and dimension metadata (JSON), TurboIntegrator process scripts (`.ti` + `.json` pairs), MDX-style calculation rules (`.rules`), and PA Workspace Excel workbooks.

Changes are deployed by syncing to a TM1 Server via the TM1 project tooling (IBM Planning Analytics for Microsoft Excel or PA Workspace Admin). Version control tracks the model source; execution happens server-side.

## Object naming conventions

All objects follow a strict naming scheme: `PREFIX.SEQUENCE.Description`

| Prefix | Scope |
|--------|-------|
| `ALL`  | Global / shared across all modules |
| `DFS`  | Finance statements (P&L, Cash Flow, Working Capital, Debt, KPIs) |
| `REV`  | Revenue consolidation |
| `MAP`  | Master data mappings (Products, Accounts, Region) |
| `STG`  | Staging area for raw data ingestion |
| `SYS`  | System/infrastructure (control, logging, version copy) |
| `zCTI` | CTI Global framework utilities |

Dimension names append a type qualifier: `ALL.D.Year` (hierarchy dimension), `ALL.M.Economic_Assumptions` (measure dimension). Cube names omit the qualifier: `ALL.010.Economic_Assumptions`.

The sequence number encodes execution order within a module. Processes named `*.Chamador_*` are orchestrators that call child processes.

## Repository structure

```
cubes/          # Cube definitions: NAME.json (metadata) + NAME.rules (calculations)
                #   NAME.views/  – saved OLAP view definitions
dimensions/     # Dimension definitions: NAME.json + NAME.hierarchies/NAME.json
processes/      # TI scripts: NAME.ti (code) + NAME.json (datasource/params/variables)
chores/         # Scheduled chore definitions
files/
  .applications/  # PA Workspace Excel input workbooks (*.xlsx + *.json descriptors)
  .applications/}externals/  # Timestamped backup copies of workbooks
tm1project.json # Project config and ignore list (system/temp objects excluded from Git)
```

## TurboIntegrator process anatomy

Every `.ti` file is divided into four regions executed in order:

1. **Prolog** – variable declarations, parameter validation, datasource path resolution, view/subset creation for clearing the target cube, initial failure log write to `SYS.150.Controle_Cargas`
2. **Metadata** – dimension element inserts (adds new members discovered in the source file)
3. **Data** – row-level validation, reject routing to `SYS.250.Rejeitados`, `CellPutN`/`CellPutS` writes to the target cube
4. **Epilog** – final status log write, `CubeSaveData`, logging toggle reset

**Variable prefix conventions (enforced by CTI standard):**

| Prefix | Meaning |
|--------|---------|
| `v`    | Mapped from datasource column |
| `p`    | Process parameter (user-supplied at runtime) |
| `c`    | Constant (set once in Prolog, not changed) |
| `n`    | Numeric working variable |
| `s`    | String working variable |

**Process parameters always include** `pVersao` (target version element from `ALL.D.Version`) and `pAno` (year element from `ALL.D.Year`). Both are validated against their dimensions before the datasource is opened.

**CSV datasource format**: delimiter `;`, decimal separator `,`, thousand separator `.` (Brazilian pt-BR). Header row count = 1.

## Rules file patterns

Rules files open with:
```
SKIPCHECK;
FEEDSTRINGS;
```

`SKIPCHECK` disables consolidation checks for performance. `FEEDSTRINGS` enables string-valued rule calculations.

**Dynamic Version block** — present in all DFS cube rules. Routes calculations: versions under the `Dynamic Versions` consolidation run through rules; all others use stored values (`STET`). The `RE` (Rolling Estimate) version overlays actual data from `ALL.001.Actual_Control` onto forecast versions:

```
[]=IF(ELISANC('ALL.D.Version','Dynamic Versions',!ALL.D.Version)<>0
  % !ALL.D.Version@='RE'
  ,CONTINUE
  ,STET);
```

**Currency conversion** reads FX rates from `ALL.010.Economic_Assumptions` via `DB()` lookups. The `DFS.M.*.Adjustments` measure holds the converted result.

**Rule cell references** use the full dimension name as the hierarchy qualifier: `['DFS.D.Accounts_PnL':'DFS.D.Accounts_PnL':'Net Revenue']`.

## Data flow architecture

```
CSV files (model_upload/…)
  └─► STG.100.DataBase_Uploads       ← raw staging cube
        │
        ├─► ALL.010.Economic_Assumptions   ← macro/FX assumptions
        ├─► ALL.015.Region_Assumptions     ← regional assumptions
        ├─► MAP.* cubes                    ← account/product/region mappings
        │
        └─► DFS.100.Profit_and_Loss        ← P&L (core)
              ├─► DFS.120.Financial_Results
              ├─► DFS.140.Indirect_Cash_Flow
              ├─► DFS.160.Working_Capital
              ├─► DFS.180.Gross_Debt
              └─► DFS.200.KPIs
                    └─► REV.100.Revenue_Consolidated
                          └─► PA Workspace reports / Excel workbooks
```

Each load process clears the target slice (`ViewZeroOut`) before writing. The slice is always scoped to `(pAno, pVersao)` at minimum.

## Key system/control cubes

| Cube | Purpose |
|------|---------|
| `SYS.150.Controle_Cargas` | Process execution log — start time, end time, message, record count, user |
| `SYS.250.Rejeitados` | Row-level rejects with column values and error message |
| `ALL.001.Actual_Control` | Flags which (Year, Month, Version) intersections hold actual data |
| `ALL.002.Uploads_Status` | Upload health tracking |
| `SYS.900/910.Version_Copy_*` | Parameters and source data for version copy operations |
| `SYS.915.Version_Copy_Log` | Audit log for `zCTI.001.*` version copy processes |

## Shared dimensions across all fact cubes

`ALL.D.Year`, `ALL.D.Month`, `ALL.D.Version`, `ALL.D.Currency`, `ALL.D.Region`, `ALL.D.Products` — these are defined once and referenced by all modules. Modifications to these dimensions affect every cube that uses them.

`ALL.D.Version` contains a `Dynamic Versions` consolidation that gates rule calculations in every rules file. Adding or reclassifying version elements here has model-wide impact.

## tm1project.json

The `Ignore` array lists TM1 objects that should not be tracked in Git — primarily Plans/QUBEdocs system objects and user-held cubes (`}Hold_*`). The `Files` array registers Excel workbooks under version control along with their timestamped backups in `}externals/`.
