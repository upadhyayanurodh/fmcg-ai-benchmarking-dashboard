# Architecture

## System Overview

The dashboard is a single static HTML file. All computation happens at page load in the browser. There is no server-side logic, no database, and no API calls at runtime.

```mermaid
flowchart LR
    subgraph DATA["Data Layer"]
        CSV["18 CSV Files\n93 processes · 465 activities\n4 companies · 10 business functions"]
    end

    subgraph APP["Application — Single HTML File"]
        V1["Viz 1 · State of AI Scorecard\nRanked leaderboard · stacked bars"]
        V2["Viz 2 · Use of AI Scorecard\nRanked leaderboard · stacked bars"]
        V3["Viz 3 · BF × State of AI\nSpectrum grid · 10 BFs × 4 companies"]
        V4["Viz 4 · Process × Use of AI\nSpectrum grid · aligned BF rows"]
    end

    subgraph AWS["AWS"]
        S3["S3\nStatic website hosting\nap-south-1"]
        CF["CloudFront\nGlobal CDN · HTTPS"]
    end

    subgraph DEV["Local Dev"]
        PY["Python http.server\nPort 7654"]
    end

    CSV -->|"embedded in JS"| APP
    PY -->|"serves"| APP
    APP -->|"aws s3 sync"| S3
    S3 --> CF
    CF -->|"HTTPS"| USERS["CXO / Strategy Teams\nAny browser · No login"]
```

## Components

| Component | Role | Technology |
|---|---|---|
| `Build/state_of_ai_scorecard.html` | All four visualisations — JS data, rendering logic, CSS | Vanilla HTML/JS/CSS |
| `Build/assets/logos/` | Company logo SVGs — generic letter avatars | SVG |
| AWS S3 (`fmcg-ai-benchmarking-dashboard`) | Static file hosting | AWS S3 — ap-south-1 |
| AWS CloudFront (`E3EF4S9G53A1LL`) | HTTPS + global CDN | AWS CloudFront |
| Python `http.server` | Local development server | Python 3 stdlib |

## Data Flow

1. **Source data**: 18 CSV files define the raw dataset — 93 processes, 465 activities, 10 business functions, 4 companies
2. **Pre-processing**: Aggregate counts (processes per tier per company per BF) computed from the CSVs and embedded directly as JS arrays in the HTML file
3. **Page load**: Browser parses the HTML, JS IIFEs execute and render all four visualisations via DOM `innerHTML`
4. **No network calls**: Once the HTML file is loaded, all rendering is local — no fetch, no XHR, no CDN fonts

---

## Data Model

The dashboard renders four GROUP BY COUNT aggregations over 18 CSV source files. If you understand the data, the visuals follow naturally — Viz 1 and 2 are stacked bars whose widths come from tier counts; Viz 3 and 4 are spectrum grids whose cells come from per-BF lookups and process groupings. This section documents the schema, relationships, and exact aggregation pipeline so that updating the dashboard with new data requires no reverse-engineering of the HTML.

### Data Flow — CSV to Dashboard

```mermaid
flowchart TD
    subgraph T1["Tier 1 — Reference & Taxonomy (6 files — never change)"]
        direction LR
        BFM["BusinessFunctionMaster\n10 BFs"]
        PROC["ProcessBusinessFunctionMap\n93 processes"]
        ACT["ActivitiesProcessesMap\n465 activities"]
        SAI["StateOfAIMaster · 4 tiers"]
        UAI["UseOfAIMaster · 4 tiers"]
        FM["FMCGCompaniesMaster · 4 companies"]
    end

    subgraph T2["Tier 2 — Company Scoring (12 files — update per data refresh)"]
        direction LR
        CBF["×4 CompanyBFDetails\n10 rows · StateOfAI + UseOfAI + headcount per BF"]
        CPR["×4 CompanyProcessMap\n93 rows · StateOfAI + UseOfAI per process"]
        CAC["×4 CompanyActivityMap\n465 rows · UseOfAI per activity"]
    end

    subgraph VIZ["Dashboard — 4 JS data blocks (hardcoded aggregates)"]
        V1["Viz 1\nCOUNT processes by StateOfAI\n→ stacked bar leaderboard"]
        V2["Viz 2\nCOUNT activities by UseOfAI\n→ stacked bar leaderboard"]
        V3["Viz 3\nSELECT StateOfAI per BF\n→ BF spectrum grid"]
        V4["Viz 4\nCOUNT processes by BF × UseOfAI\n→ aligned process spectrum grid"]
    end

    BFM --> PROC --> ACT
    CPR -->|"93 rows · GROUP BY StateOfAIId"| V1
    CAC -->|"465 rows · GROUP BY UseOfAIId"| V2
    CBF -->|"10 rows · SELECT StateOfAIId"| V3
    CPR -->|"93 rows · JOIN PROC for BF\nGROUP BY BF + UseOfAIId"| V4
    PROC -.->|"join for BF lookup"| V4
```

---

### End-to-End Trace

Before reading schemas, walk one data point from raw CSV to its rendered position in the dashboard. This is what the entire pipeline does — 93× for processes, 465× for activities, per company.

**Subject: Process 3 — IMC Planning & Execution (HUL)**

**Step 1 — Find the process in the taxonomy.**

`ProcessBusinessFunctionMap.csv`, row 3:

| ProcessId | ProcessName | BusinessFunctionId |
|---|---|---|
| 3 | Integrated Marketing Communications (IMC) Planning & Execution | 1 (Marketing & Brand Management) |

**Step 2 — Look up HUL's classification for this process.**

`HULProcessBusinessFunctionMap.csv`, row 3 (`Id=3` maps to `ProcessId=3` by row position — see Implicit FK below):

| Id | HULCompanyId | HULStateOfAIId | HULUseOfAIId |
|---|---|---|---|
| 3 | 1 (HUL) | 1 (AI Application) | 2 (AI Enabled) |

**Step 3 — See where this process appears in the dashboard.**

- **Viz 1**: `HULStateOfAIId = 1` (AI Application) → this row is counted in HUL's `aiApp = 6`
- **Viz 4**: `BusinessFunctionId = 1`, `HULUseOfAIId = 2` (AI Enabled) → this process appears in the BF01 · AI Enabled row in HUL's column (one of 2 AI Enabled processes in Marketing)

**Step 4 — Trace one of its five activities into Viz 2.**

`ActivitiesProcessesMap.csv`, activity 11:

| ActivityId | ActivityName | ProcessId |
|---|---|---|
| 11 | Develop annual communication strategy & campaign calendar | 3 |

`HULActivitiesProcessesMap.csv`, row 11 (`Id=11` maps to `ActivityId=11`):

| Id | HULCompanyId | HULUseOfAIId |
|---|---|---|
| 11 | 1 (HUL) | 4 (No AI use at all) |

**Viz 2**: this activity is counted in HUL's `noAI = 281`.

> The dashboard is exactly this trace, repeated for every process and every activity, for all 4 companies. The resulting counts are pre-computed and hardcoded into the JS.

---

### File Catalogue

| Tier | File | Rows | Grain |
|---|---|---|---|
| Reference | `FMCGCompaniesMaster.csv` | 4 | One row per company |
| Reference | `BusinessFunctionMaster.csv` | 10 | One row per business function |
| Reference | `StateOfAIMaster.csv` | 4 | One row per State of AI tier |
| Reference | `UseOfAIMaster.csv` | 4 | One row per Use of AI tier |
| Taxonomy | `ProcessBusinessFunctionMap.csv` | 93 | One row per process |
| Taxonomy | `ActivitiesProcessesMap.csv` | 465 | One row per activity (5 per process) |
| Company · BF | `HULBusinessFunctionsDetails.csv` | 10 | HUL State of AI + Use of AI + headcount per BF |
| Company · BF | `PandGBusinessFunctionsDetails.csv` | 10 | P&G — same structure |
| Company · BF | `LorealBusinessFunctionsDetails.csv` | 10 | L'Oréal — same structure |
| Company · BF | `NestleBusinessFunctionsDetails.csv` | 10 | Nestlé — same structure |
| Company · Process | `HULProcessBusinessFunctionMap.csv` | 93 | HUL State of AI + Use of AI per process |
| Company · Process | `PandGProcessBusinessFunctionMap.csv` | 93 | P&G — same structure |
| Company · Process | `LorealProcessBusinessFunctionMap.csv` | 93 | L'Oréal — same structure |
| Company · Process | `NestleProcessBusinessFunctionMap.csv` | 93 | Nestlé — same structure |
| Company · Activity | `HULActivitiesProcessesMap.csv` | 465 | HUL Use of AI per activity |
| Company · Activity | `PandGActivitiesProcessesMap.csv` | 465 | P&G — same structure |
| Company · Activity | `LorealActivitiesProcessesMap.csv` | 465 | L'Oréal — same structure |
| Company · Activity | `NestleActivitiesProcessesMap.csv` | 465 | Nestlé — same structure |

---

### Schemas

#### Reference Tables

**`FMCGCompaniesMaster.csv`** — 4 rows

| Column | Type | Role |
|---|---|---|
| `CompanyId` | INT | PK |
| `CompanyName` | STRING | Display name |
| `TotalNumberOfEmployees` | INT | Total headcount (India entity) |

`1` = HUL (18,240) · `2` = P&G (3,744) · `3` = L'Oréal (2,300) · `4` = Nestlé (7,980)

---

**`BusinessFunctionMaster.csv`** — 10 rows

| Column | Type | Role |
|---|---|---|
| `BusinessFunctionId` | INT | PK — 1–10 |
| `BusinessFunctionName` | STRING | Display name |

| Id | Business Function |
|---|---|
| 1 | Marketing & Brand Management |
| 2 | Sales & Distribution |
| 3 | Supply Chain |
| 4 | R&D / Product Development |
| 5 | Finance |
| 6 | HR |
| 7 | Legal & Regulatory / Compliance |
| 8 | IT / Digital |
| 9 | Manufacturing / Operations |
| 10 | Corporate Affairs / External Affairs |

---

**`StateOfAIMaster.csv`** — 4 rows

| Column | Type | Role |
|---|---|---|
| `StateOfAIComponentId` | INT | PK — 1–4 |
| `StateOfAIComponentName` | STRING | Tier label |

| Id | Label | Advancement rank |
|---|---|---|
| 1 | AI Application | 2nd (most processes sit here) |
| 2 | AI Adoption | 3rd |
| 3 | AI Nativeness | 1st — north star |
| 4 | Without AI | 4th — least advanced |

> ⚠️ **IDs are not ordinal.** `Id=3` is the most advanced tier, not the third. Correct advancement sequence: **3 → 1 → 2 → 4**. Never sort or compare these IDs numerically — always resolve to the label first.

---

**`UseOfAIMaster.csv`** — 4 rows

| Column | Type | Role |
|---|---|---|
| `UseOfAIComponentId` | INT | PK — 1–4 |
| `UseOfAIComponentName` | STRING | Tier label |

| Id | Label | Advancement rank |
|---|---|---|
| 1 | AI Assisted | 3rd |
| 2 | AI Enabled | 2nd |
| 3 | Autonomous AI | 1st — north star |
| 4 | No AI use at all | 4th — least advanced |

> ⚠️ **Same non-ordinal pattern.** `Id=3` is the most advanced. Correct advancement sequence: **3 → 2 → 1 → 4**. This directly determines the column order of the `bfCounts` inner array in Viz 4 — the columns run `[Id=3, Id=2, Id=1, Id=4]`, not `[Id=1, Id=2, Id=3, Id=4]`. Getting this wrong silently swaps AI Enabled and AI Assisted rows in Viz 4.

---

#### Taxonomy Tables

**`ProcessBusinessFunctionMap.csv`** — 93 rows

| Column | Type | Role |
|---|---|---|
| `ProcessId` | INT | PK — 1–93 |
| `ProcessName` | STRING | Process display name |
| `BusinessFunctionId` | INT | FK → `BusinessFunctionMaster.BusinessFunctionId` |

Process count per BF: Marketing (10) · Sales & Distribution (9) · Supply Chain (9) · R&D / Product Dev (9) · Finance (10) · HR (10) · Legal & Regulatory (9) · IT / Digital (9) · Manufacturing (10) · Corporate Affairs (8) = **93 total**

ProcessIds are sequential within each BF block: BF 1 = rows 1–10 · BF 2 = 11–19 · BF 3 = 20–28 · BF 4 = 29–37 · BF 5 = 38–47 · BF 6 = 48–57 · BF 7 = 58–66 · BF 8 = 67–75 · BF 9 = 76–85 · BF 10 = 86–93

---

**`ActivitiesProcessesMap.csv`** — 465 rows

| Column | Type | Role |
|---|---|---|
| `ActivityId` | INT | PK — 1–465 |
| `ActivityName` | STRING | Granular activity description |
| `ProcessId` | INT | FK → `ProcessBusinessFunctionMap.ProcessId` |
| `BusinessFunctionId` | INT | FK → `BusinessFunctionMaster.BusinessFunctionId` (denormalised) |

5 activities per process × 93 processes = 465 activities. `BusinessFunctionId` is derivable via `ProcessId` but retained here to allow direct BF-level activity counts without a join.

---

#### Company Scoring Tables

**The implicit FK pattern — read this once.**
None of the 12 company scoring files declares a `ProcessId`, `ActivityId`, or `BusinessFunctionId` column. Instead, each file has an `Id` column that runs 1 → N. **Row N in a company scoring file corresponds to ProcessId / ActivityId / BusinessFunctionId = N in the matching base table — by row position, not by named FK.** The files are compact but they depend entirely on the base taxonomy files maintaining a stable, sequential, 1-based row order. If you add processes or activities to the base tables, always append at the end and regenerate all 12 company scoring files in sync.

From the end-to-end trace: `HULProcessBusinessFunctionMap.csv` row 3 (`Id=3`) corresponds to `ProcessBusinessFunctionMap.csv` row 3 (`ProcessId=3` = IMC Planning & Execution, BF 1).

---

**`{Company}BusinessFunctionsDetails.csv`** — 10 rows · 4 files

Column prefix varies: `HUL`, `PandG`, `Loreal`, `Nestle`.

| Column | Type | Role |
|---|---|---|
| `Id` | INT | Implicit FK → `BusinessFunctionMaster.BusinessFunctionId` (row 1 = BF 1 … row 10 = BF 10) |
| `{Co}CompanyId` | INT | FK → `FMCGCompaniesMaster.CompanyId` (constant within each file) |
| `{Co}StateOfAIId` | INT | FK → `StateOfAIMaster.StateOfAIComponentId` — BF-level State of AI |
| `{Co}UseOfAIId` | INT | FK → `UseOfAIMaster.UseOfAIComponentId` — BF-level Use of AI |
| `{Co}NumberOfEmployees` | INT | Headcount in this BF for this company |

Headcount by BF × company. Column headers abbreviate BF names; column sums must equal `TotalNumberOfEmployees` in `FMCGCompaniesMaster`:

| BF | M&B | S&D | SC | R&D | Fin | HR | Leg | IT | Mfg | CorpAff | **Total** |
|---|---|---|---|---|---|---|---|---|---|---|---|
| HUL | 950 | 5,700 | 1,330 | 760 | 950 | 570 | 380 | 760 | 6,650 | 190 | **18,240** |
| P&G | 195 | 780 | 312 | 273 | 195 | 117 | 78 | 195 | 1,560 | 39 | **3,744** |
| L'Oréal | 250 | 750 | 175 | 125 | 125 | 75 | 50 | 125 | 600 | 25 | **2,300** |
| Nestlé | 336 | 1,680 | 672 | 168 | 420 | 252 | 168 | 420 | 3,780 | 84 | **7,980** |

---

**`{Company}ProcessBusinessFunctionMap.csv`** — 93 rows · 4 files

| Column | Type | Role |
|---|---|---|
| `Id` | INT | Implicit FK → `ProcessBusinessFunctionMap.ProcessId` (row 1 = ProcessId 1 … row 93 = ProcessId 93) |
| `{Co}CompanyId` | INT | FK → `FMCGCompaniesMaster.CompanyId` |
| `{Co}StateOfAIId` | INT | FK → `StateOfAIMaster.StateOfAIComponentId` — process-level State of AI |
| `{Co}UseOfAIId` | INT | FK → `UseOfAIMaster.UseOfAIComponentId` — **process-level** Use of AI |

To get `ProcessName` or `BusinessFunctionId` for any row, join `Id = ProcessId` in `ProcessBusinessFunctionMap.csv`.

> **Note on the two Use of AI classifications.** This file has `{Co}UseOfAIId` at the process level. The activity scoring files below have `{Co}UseOfAIId` at the activity level. These are **independently assessed** — a process rated AI Enabled overall may contain a mix of AI Assisted and No AI activities. **Viz 4 uses process-level UseOfAI (this file). Viz 2 uses activity-level UseOfAI (the file below).** Do not assume one can be derived from the other.

---

**`{Company}ActivitiesProcessesMap.csv`** — 465 rows · 4 files

| Column | Type | Role |
|---|---|---|
| `Id` | INT | Implicit FK → `ActivitiesProcessesMap.ActivityId` (row 1 = ActivityId 1 … row 465 = ActivityId 465) |
| `{Co}CompanyId` | INT | FK → `FMCGCompaniesMaster.CompanyId` |
| `{Co}UseOfAIId` | INT | FK → `UseOfAIMaster.UseOfAIComponentId` — **activity-level** Use of AI |

No `StateOfAIId` column. State of AI is a holistic maturity classification of a whole process — it cannot be applied to an individual activity within it. Only Use of AI (the intensity of AI utilisation in a specific task) is granular enough to classify at activity level. This is an intentional design constraint, not an omission.

---

### Entity Relationships

```mermaid
erDiagram
    BusinessFunctionMaster ||--o{ ProcessBusinessFunctionMap : "1 BF has N processes"
    ProcessBusinessFunctionMap ||--o{ ActivitiesProcessesMap : "1 process has 5 activities"
    BusinessFunctionMaster ||--o{ ActivitiesProcessesMap : "denormalised ref"
    BusinessFunctionMaster ||--o{ Co_BF_Scores : "BF scored x4 companies"
    ProcessBusinessFunctionMap ||--o{ Co_Process_Scores : "process scored x4 companies"
    ActivitiesProcessesMap ||--o{ Co_Activity_Scores : "activity scored x4 companies"
    FMCGCompaniesMaster ||--o{ Co_BF_Scores : "company"
    FMCGCompaniesMaster ||--o{ Co_Process_Scores : "company"
    FMCGCompaniesMaster ||--o{ Co_Activity_Scores : "company"
    StateOfAIMaster ||--o{ Co_BF_Scores : "State of AI tier"
    StateOfAIMaster ||--o{ Co_Process_Scores : "State of AI tier"
    UseOfAIMaster ||--o{ Co_BF_Scores : "Use of AI tier"
    UseOfAIMaster ||--o{ Co_Process_Scores : "Use of AI tier"
    UseOfAIMaster ||--o{ Co_Activity_Scores : "Use of AI tier"

    BusinessFunctionMaster { int BusinessFunctionId PK; string BusinessFunctionName }
    ProcessBusinessFunctionMap { int ProcessId PK; string ProcessName; int BusinessFunctionId FK }
    ActivitiesProcessesMap { int ActivityId PK; string ActivityName; int ProcessId FK; int BusinessFunctionId FK }
    StateOfAIMaster { int StateOfAIComponentId PK; string StateOfAIComponentName }
    UseOfAIMaster { int UseOfAIComponentId PK; string UseOfAIComponentName }
    FMCGCompaniesMaster { int CompanyId PK; string CompanyName; int TotalNumberOfEmployees }
    Co_BF_Scores { int Id FK; int CompanyId FK; int StateOfAIId FK; int UseOfAIId FK; int NumberOfEmployees }
    Co_Process_Scores { int Id FK; int CompanyId FK; int StateOfAIId FK; int UseOfAIId FK }
    Co_Activity_Scores { int Id FK; int CompanyId FK; int UseOfAIId FK }
```

`Co_BF_Scores`, `Co_Process_Scores`, `Co_Activity_Scores` each represent 4 physical files (one per company). `Id` is an implicit FK — row `n` maps to `BusinessFunctionId` / `ProcessId` / `ActivityId` = `n` in the corresponding base table.

---

### Aggregation Pipeline

> **The dashboard never reads CSVs at runtime.** These aggregations run offline; results are hardcoded into the four JS data blocks in `Build/state_of_ai_scorecard.html`. To update the dashboard, re-run these scripts and patch the arrays.
>
> ⚠️ Read all CSV files with `encoding='utf-8'` — L'Oréal and Nestlé contain non-ASCII characters that corrupt silently with default system encoding.

---

#### Viz 1 — State of AI Scorecard

**What it produces:** For each company, a count of its 93 processes at each State of AI tier.

```python
import pandas as pd

DATA = "path/to/Data/"

files = {
    "HUL":     "HULProcessBusinessFunctionMap.csv",
    "P&G":     "PandGProcessBusinessFunctionMap.csv",
    "L'Oréal": "LorealProcessBusinessFunctionMap.csv",
    "Nestlé":  "NestleProcessBusinessFunctionMap.csv",
}
for co, fname in files.items():
    df = pd.read_csv(f"{DATA}{fname}", encoding="utf-8")
    col = [c for c in df.columns if "StateOfAIId" in c][0]
    cts = df[col].value_counts()
    # StateOfAI IDs: 3=AI Nativeness, 1=AI Application, 2=AI Adoption, 4=Without AI
    print(f"{co}: aiNative={cts.get(3,0)}, aiApp={cts.get(1,0)}, "
          f"aiAdop={cts.get(2,0)}, withoutAI={cts.get(4,0)}")
```

**JS data block:** `companies[i] = { name, logo, isSubject, aiNative, aiApp, aiAdop, withoutAI }`
Pre-sort descending by `aiApp` before embedding — rank order is derived from array index, not computed at render time.
`aiNative + aiApp + aiAdop + withoutAI` **must equal `TOTAL_PROCESSES` (93).**

**Current values (pre-sorted by aiApp descending):**

| Rank | Company | AI Nativeness | AI Application | AI Adoption | Without AI |
|---|---|---|---|---|---|
| 1 | P&G | 0 | **27** | 62 | 4 |
| 2 | L'Oréal | 0 | **13** | 75 | 5 |
| 3 | Nestlé | 0 | **9** | 80 | 4 |
| 4 | HUL | 0 | **6** | 79 | 8 |

---

#### Viz 2 — Use of AI Scorecard

**What it produces:** For each company, a count of its 465 activities at each Use of AI tier.

```python
files = {
    "HUL":     "HULActivitiesProcessesMap.csv",
    "P&G":     "PandGActivitiesProcessesMap.csv",
    "L'Oréal": "LorealActivitiesProcessesMap.csv",
    "Nestlé":  "NestleActivitiesProcessesMap.csv",
}
for co, fname in files.items():
    df = pd.read_csv(f"{DATA}{fname}", encoding="utf-8")
    col = [c for c in df.columns if "UseOfAIId" in c][0]
    cts = df[col].value_counts()
    # UseOfAI IDs: 3=Autonomous AI, 1=AI Assisted, 2=AI Enabled, 4=No AI
    print(f"{co}: autoAI={cts.get(3,0)}, aiAssist={cts.get(1,0)}, "
          f"aiEnable={cts.get(2,0)}, noAI={cts.get(4,0)}")
```

**JS data block:** `companies[i] = { ..., autoAI, aiAssist, aiEnable, noAI }`
Pre-sort descending by `aiAssist`.
`autoAI + aiAssist + aiEnable + noAI` **must equal `TOTAL_ACTIVITIES` (465).**

**Current values (pre-sorted by aiAssist descending):**

| Rank | Company | Autonomous AI | AI Assisted | AI Enabled | No AI |
|---|---|---|---|---|---|
| 1 | Nestlé | 0 | **145** | 57 | 263 |
| 2 | HUL | 0 | **143** | 41 | 281 |
| 3 | L'Oréal | 0 | **142** | 57 | 266 |
| 4 | P&G | 0 | **126** | 144 | 195 |

---

#### Viz 3 — Business Function × State of AI

**What it produces:** For each company, the State of AI tier for each of the 10 BFs — a list of 10 integers.

```python
files = {
    "HUL":     "HULBusinessFunctionsDetails.csv",
    "P&G":     "PandGBusinessFunctionsDetails.csv",
    "L'Oréal": "LorealBusinessFunctionsDetails.csv",
    "Nestlé":  "NestleBusinessFunctionsDetails.csv",
}
for co, fname in files.items():
    df = pd.read_csv(f"{DATA}{fname}", encoding="utf-8")
    col = [c for c in df.columns if "StateOfAIId" in c][0]
    # Rows are in BF order 1–10; extract the column as a list directly
    bf_states = df[col].tolist()
    print(f"{co}: bfStates = {bf_states}")
```

**JS data block:** `companies[i].bfStates = [int × 10]`
Index 0 = BF 1 (Marketing) … index 9 = BF 10 (Corporate Affairs). Each integer is a `StateOfAIComponentId`.

**Current values:** Every cell = `2` (AI Adoption) for all 10 BFs across all 4 companies. The AI Nativeness, AI Application, and Without AI rows in Viz 3 are intentionally empty — this reflects the actual dataset, not a rendering bug.

---

#### Viz 4 — Process × Use of AI

**What it produces:** For each company, for each of the 10 BFs, a count of processes at each Use of AI tier — a 10×4 matrix per company.

```python
import collections

proc_bf_map = pd.read_csv(f"{DATA}ProcessBusinessFunctionMap.csv", encoding="utf-8")
proc_to_bf = dict(zip(proc_bf_map.ProcessId, proc_bf_map.BusinessFunctionId))

files = {
    "HUL":     "HULProcessBusinessFunctionMap.csv",
    "P&G":     "PandGProcessBusinessFunctionMap.csv",
    "L'Oréal": "LorealProcessBusinessFunctionMap.csv",
    "Nestlé":  "NestleProcessBusinessFunctionMap.csv",
}
# Inner array column order must follow Viz 4's row order top-to-bottom:
# [Autonomous AI (Id=3), AI Enabled (Id=2), AI Assisted (Id=1), No AI (Id=4)]
VIZ4_ORDER = [3, 2, 1, 4]

for co, fname in files.items():
    df = pd.read_csv(f"{DATA}{fname}", encoding="utf-8")
    use_col = [c for c in df.columns if "UseOfAIId" in c][0]
    df["BF"] = df["Id"].map(proc_to_bf)
    grp = df.groupby(["BF", use_col]).size().unstack(fill_value=0)
    bf_counts = [
        [int(grp.at[bf, tid]) if tid in grp.columns else 0 for tid in VIZ4_ORDER]
        for bf in range(1, 11)
    ]
    print(f"{co}: bfCounts = {bf_counts}")
```

**JS data block:** `companies[i].bfCounts = [[autoAI, aiEnabled, aiAssisted, noAI] × 10]`
Column order `[Id=3, Id=2, Id=1, Id=4]` matches Viz 4's top-to-bottom row display. Supplying `[Id=1, Id=2, Id=3, Id=4]` (numeric order) silently swaps AI Enabled and AI Assisted rows.

> ⚠️ For each BF, the four-value sum must be **identical across all 4 companies.** Viz 4 uses fixed 30px row heights — a BF with 10 processes for HUL and 11 for P&G will push every BF below it out of alignment with no error thrown.

**Current values** — `[AutoAI, AIEnabled, AIAssisted, NoAI]` per BF:

| Business Function | HUL | P&G | L'Oréal | Nestlé | BF total |
|---|---|---|---|---|---|
| Marketing & Brand Mgmt | [0, 2, 8, 0] | [0, 5, 5, 0] | [0, 5, 5, 0] | [0, 2, 8, 0] | **10** |
| Sales & Distribution | [0, 1, 8, 0] | [0, 0, 9, 0] | [0, 1, 8, 0] | [0, 1, 8, 0] | **9** |
| Supply Chain | [0, 1, 8, 0] | [0, 1, 8, 0] | [0, 1, 8, 0] | [0, 1, 8, 0] | **9** |
| R&D / Product Dev | [0, 1, 7, 1] | [0, 3, 6, 0] | [0, 4, 5, 0] | [0, 2, 7, 0] | **9** |
| Finance | [0, 0, 10, 0] | [0, 5, 5, 0] | [0, 0, 10, 0] | [0, 0, 10, 0] | **10** |
| HR | [0, 0, 10, 0] | [0, 3, 7, 0] | [0, 0, 10, 0] | [0, 0, 10, 0] | **10** |
| Legal & Regulatory | [0, 0, 8, 1] | [0, 1, 8, 0] | [0, 0, 8, 1] | [0, 0, 8, 1] | **9** |
| IT / Digital | [0, 0, 7, 2] | [0, 4, 5, 0] | [0, 1, 6, 2] | [0, 1, 7, 1] | **9** |
| Manufacturing / Ops | [0, 1, 6, 3] | [0, 2, 5, 3] | [0, 0, 7, 3] | [0, 1, 6, 3] | **10** |
| Corporate Affairs | [0, 0, 7, 1] | [0, 3, 4, 1] | [0, 1, 6, 1] | [0, 1, 6, 1] | **8** |

---

### Critical Gotchas

These are the four failure modes that produce wrong output silently — no error, no warning.

**1 · UseOfAI and StateOfAI IDs are not ordinal.**
`StateOfAIId=3` is the most advanced tier (AI Nativeness), not the third. `UseOfAIId=3` is the most advanced (Autonomous AI), not the third. Never sort or compare these IDs numerically. Always resolve to the label, then apply your own rank. Getting this wrong in a filter or sort produces miscounted tiers with bars still summing to 100%.

**2 · Viz 4 `bfCounts` inner array column order is semantic, not numeric.**
Correct: `[Id=3, Id=2, Id=1, Id=4]` = `[Autonomous AI, AI Enabled, AI Assisted, No AI]` — top to bottom in Viz 4.
Wrong: `[Id=1, Id=2, Id=3, Id=4]` — puts AI Assisted before AI Enabled, swapping two rows across all 4 company columns.

**3 · Viz 4 BF process totals must match across all companies.**
The JS does not validate this. If BF 9 (Manufacturing) has 10 processes for HUL but 11 for P&G, P&G's Manufacturing row is taller, pushing Corporate Affairs out of alignment. The gap accumulates silently. Always verify `bfCounts[i].reduce((a,b)=>a+b)` is identical for all 4 companies per BF before embedding.

**4 · The implicit FK depends on base table row order being stable.**
If you insert a new process between existing rows in `ProcessBusinessFunctionMap.csv`, every company scoring file's row-to-process mapping shifts below the insertion point. Always **append** new processes or activities at the end of their respective base files, and regenerate all 12 company scoring files in sync.

---

### Data Quality

All 18 source files are confirmed clean: 0 duplicate PKs, 0 FK orphans, 0 missing values. No validation code is needed for the current dataset.

**UTF-8 encoding:** `L'Oréal` contains a right single quotation mark (`'`, U+2019) and `é` (U+00E9). `Nestlé` contains `é`. Read all CSVs with `pd.read_csv(..., encoding='utf-8')` or `encoding='utf-8-sig'` if a BOM is present. Default system encoding will either raise an error or silently replace the characters with `?`.

**Pre-sort requirement:** Company arrays for Viz 1 and Viz 2 must be sorted by their ranking metric descending before embedding (`aiApp` for Viz 1, `aiAssist` for Viz 2). The JS derives rank from array index — it does not sort at render time.

---

### File Catalogue

| Tier | File | Rows | Grain |
|---|---|---|---|
| Reference | `FMCGCompaniesMaster.csv` | 4 | One row per company |
| Reference | `BusinessFunctionMaster.csv` | 10 | One row per business function |
| Reference | `StateOfAIMaster.csv` | 4 | One row per State of AI tier |
| Reference | `UseOfAIMaster.csv` | 4 | One row per Use of AI tier |
| Taxonomy | `ProcessBusinessFunctionMap.csv` | 93 | One row per process |
| Taxonomy | `ActivitiesProcessesMap.csv` | 465 | One row per activity |
| Company · BF | `HULBusinessFunctionsDetails.csv` | 10 | HUL AI score + headcount per BF |
| Company · BF | `PandGBusinessFunctionsDetails.csv` | 10 | P&G AI score + headcount per BF |
| Company · BF | `LorealBusinessFunctionsDetails.csv` | 10 | L'Oréal AI score + headcount per BF |
| Company · BF | `NestleBusinessFunctionsDetails.csv` | 10 | Nestlé AI score + headcount per BF |
| Company · Process | `HULProcessBusinessFunctionMap.csv` | 93 | HUL AI score per process |
| Company · Process | `PandGProcessBusinessFunctionMap.csv` | 93 | P&G AI score per process |
| Company · Process | `LorealProcessBusinessFunctionMap.csv` | 93 | L'Oréal AI score per process |
| Company · Process | `NestleProcessBusinessFunctionMap.csv` | 93 | Nestlé AI score per process |
| Company · Activity | `HULActivitiesProcessesMap.csv` | 465 | HUL Use of AI score per activity |
| Company · Activity | `PandGActivitiesProcessesMap.csv` | 465 | P&G Use of AI score per activity |
| Company · Activity | `LorealActivitiesProcessesMap.csv` | 465 | L'Oréal Use of AI score per activity |
| Company · Activity | `NestleActivitiesProcessesMap.csv` | 465 | Nestlé Use of AI score per activity |

---

### Schemas

#### Reference Tables

**`FMCGCompaniesMaster.csv`** — 4 rows

| Column | Type | Role |
|---|---|---|
| `CompanyId` | INT | PK |
| `CompanyName` | STRING | Company display name |
| `TotalNumberOfEmployees` | INT | Total headcount (India entity) |

Values: `1` = HUL (18,240) · `2` = P&G (3,744) · `3` = L'Oréal (2,300) · `4` = Nestlé (7,980)

---

**`BusinessFunctionMaster.csv`** — 10 rows

| Column | Type | Role |
|---|---|---|
| `BusinessFunctionId` | INT | PK — 1–10 |
| `BusinessFunctionName` | STRING | Business function display name |

Functions in order: `1` Marketing & Brand Management · `2` Sales & Distribution · `3` Supply Chain · `4` R&D / Product Development · `5` Finance · `6` HR · `7` Legal & Regulatory / Compliance · `8` IT / Digital · `9` Manufacturing / Operations · `10` Corporate Affairs / External Affairs

---

**`StateOfAIMaster.csv`** — 4 rows

| Column | Type | Role |
|---|---|---|
| `StateOfAIComponentId` | INT | PK — 1–4 |
| `StateOfAIComponentName` | STRING | Tier label |

| Id | Label | Semantic rank |
|---|---|---|
| `1` | AI Application | 2nd most advanced |
| `2` | AI Adoption | 3rd most advanced |
| `3` | AI Nativeness | Most advanced (north star) |
| `4` | Without AI | Least advanced |

> ⚠️ IDs do **not** reflect advancement order. Semantic order (advanced → lagging): AI Nativeness (3) → AI Application (1) → AI Adoption (2) → Without AI (4).

---

**`UseOfAIMaster.csv`** — 4 rows

| Column | Type | Role |
|---|---|---|
| `UseOfAIComponentId` | INT | PK — 1–4 |
| `UseOfAIComponentName` | STRING | Tier label |

| Id | Label | Semantic rank |
|---|---|---|
| `1` | AI Assisted | 3rd most advanced |
| `2` | AI Enabled | 2nd most advanced |
| `3` | Autonomous AI | Most advanced (north star) |
| `4` | No AI use at all | Least advanced |

> ⚠️ IDs do **not** reflect advancement order. Semantic order (advanced → lagging): Autonomous AI (3) → AI Enabled (2) → AI Assisted (1) → No AI use at all (4). When building the `bfCounts` array for Viz 4, column order must follow semantic rank — not ascending numeric Id.

---

#### Taxonomy Tables

**`ProcessBusinessFunctionMap.csv`** — 93 rows

| Column | Type | Role |
|---|---|---|
| `ProcessId` | INT | PK — 1–93 |
| `ProcessName` | STRING | Process display name |
| `BusinessFunctionId` | INT | FK → `BusinessFunctionMaster.BusinessFunctionId` |

Process count per BF: Marketing (10) · Sales & Distribution (9) · Supply Chain (9) · R&D / Product Dev (9) · Finance (10) · HR (10) · Legal & Regulatory (9) · IT / Digital (9) · Manufacturing (10) · Corporate Affairs (8) = **93 total**

---

**`ActivitiesProcessesMap.csv`** — 465 rows

| Column | Type | Role |
|---|---|---|
| `ActivityId` | INT | PK — 1–465 |
| `ActivityName` | STRING | Granular activity description |
| `ProcessId` | INT | FK → `ProcessBusinessFunctionMap.ProcessId` |
| `BusinessFunctionId` | INT | FK → `BusinessFunctionMaster.BusinessFunctionId` |

5 activities per process × 93 processes = **465 activities**. `BusinessFunctionId` is denormalised — derivable via `ProcessId → ProcessBusinessFunctionMap`, but retained for direct BF-level aggregations without a join.

---

#### Company-Specific Business Function Tables — 4 files × 10 rows

Schema is identical across all four files; only the column name prefix changes (`HUL`, `PandG`, `Loreal`, `Nestle`).

**`{Company}BusinessFunctionsDetails.csv`**

| Column | Type | Role |
|---|---|---|
| `Id` | INT | Implicit FK → `BusinessFunctionMaster.BusinessFunctionId` (row position = BF Id, 1–10) |
| `{Co}CompanyId` | INT | FK → `FMCGCompaniesMaster.CompanyId` (constant within each file) |
| `{Co}StateOfAIId` | INT | FK → `StateOfAIMaster.StateOfAIComponentId` — BF-level State of AI |
| `{Co}UseOfAIId` | INT | FK → `UseOfAIMaster.UseOfAIComponentId` — BF-level Use of AI |
| `{Co}NumberOfEmployees` | INT | Headcount allocated to this BF for this company |

The sum of `NumberOfEmployees` across all 10 rows equals `TotalNumberOfEmployees` in `FMCGCompaniesMaster`.

Headcount per BF (BF order 1–10):
- **HUL**: 950 · 5,700 · 1,330 · 760 · 950 · 570 · 380 · 760 · 6,650 · 190 = 18,240
- **P&G**: 195 · 780 · 312 · 273 · 195 · 117 · 78 · 195 · 1,560 · 39 = 3,744
- **L'Oréal**: 250 · 750 · 175 · 125 · 125 · 75 · 50 · 125 · 600 · 25 = 2,300
- **Nestlé**: 336 · 1,680 · 672 · 168 · 420 · 252 · 168 · 420 · 3,780 · 84 = 7,980

---

#### Company-Specific Process Tables — 4 files × 93 rows

**`{Company}ProcessBusinessFunctionMap.csv`**

| Column | Type | Role |
|---|---|---|
| `Id` | INT | Implicit FK → `ProcessBusinessFunctionMap.ProcessId` (row position = ProcessId, 1–93) |
| `{Co}CompanyId` | INT | FK → `FMCGCompaniesMaster.CompanyId` |
| `{Co}StateOfAIId` | INT | FK → `StateOfAIMaster.StateOfAIComponentId` — process-level State of AI |
| `{Co}UseOfAIId` | INT | FK → `UseOfAIMaster.UseOfAIComponentId` — process-level Use of AI |

`ProcessName` and `BusinessFunctionId` are not repeated — join to `ProcessBusinessFunctionMap` via `Id = ProcessId` to retrieve them.

---

#### Company-Specific Activity Tables — 4 files × 465 rows

**`{Company}ActivitiesProcessesMap.csv`**

| Column | Type | Role |
|---|---|---|
| `Id` | INT | Implicit FK → `ActivitiesProcessesMap.ActivityId` (row position = ActivityId, 1–465) |
| `{Co}CompanyId` | INT | FK → `FMCGCompaniesMaster.CompanyId` |
| `{Co}UseOfAIId` | INT | FK → `UseOfAIMaster.UseOfAIComponentId` — activity-level Use of AI |

**State of AI is not scored at activity level** — only at process and BF level. State of AI is a holistic maturity classification of a whole process; it cannot be meaningfully applied to individual activities within it.

---

### Entity Relationships

```mermaid
erDiagram
    FMCGCompaniesMaster {
        int CompanyId PK
        string CompanyName
        int TotalNumberOfEmployees
    }
    BusinessFunctionMaster {
        int BusinessFunctionId PK
        string BusinessFunctionName
    }
    StateOfAIMaster {
        int StateOfAIComponentId PK
        string StateOfAIComponentName
    }
    UseOfAIMaster {
        int UseOfAIComponentId PK
        string UseOfAIComponentName
    }
    ProcessBusinessFunctionMap {
        int ProcessId PK
        string ProcessName
        int BusinessFunctionId FK
    }
    ActivitiesProcessesMap {
        int ActivityId PK
        string ActivityName
        int ProcessId FK
        int BusinessFunctionId FK
    }
    CompanyBFDetails_x4 {
        int Id FK
        int CompanyId FK
        int StateOfAIId FK
        int UseOfAIId FK
        int NumberOfEmployees
    }
    CompanyProcessScores_x4 {
        int Id FK
        int CompanyId FK
        int StateOfAIId FK
        int UseOfAIId FK
    }
    CompanyActivityScores_x4 {
        int Id FK
        int CompanyId FK
        int UseOfAIId FK
    }

    BusinessFunctionMaster ||--o{ ProcessBusinessFunctionMap : "groups"
    ProcessBusinessFunctionMap ||--o{ ActivitiesProcessesMap : "contains"
    BusinessFunctionMaster ||--o{ ActivitiesProcessesMap : "denormalised ref"
    BusinessFunctionMaster ||--o{ CompanyBFDetails_x4 : "scored at BF level"
    ProcessBusinessFunctionMap ||--o{ CompanyProcessScores_x4 : "scored at process level"
    ActivitiesProcessesMap ||--o{ CompanyActivityScores_x4 : "scored at activity level"
    FMCGCompaniesMaster ||--o{ CompanyBFDetails_x4 : "company"
    FMCGCompaniesMaster ||--o{ CompanyProcessScores_x4 : "company"
    FMCGCompaniesMaster ||--o{ CompanyActivityScores_x4 : "company"
    StateOfAIMaster ||--o{ CompanyBFDetails_x4 : "tier"
    StateOfAIMaster ||--o{ CompanyProcessScores_x4 : "tier"
    UseOfAIMaster ||--o{ CompanyBFDetails_x4 : "tier"
    UseOfAIMaster ||--o{ CompanyProcessScores_x4 : "tier"
    UseOfAIMaster ||--o{ CompanyActivityScores_x4 : "tier"
```

`CompanyBFDetails_x4`, `CompanyProcessScores_x4`, and `CompanyActivityScores_x4` each represent 4 physical files (one per company). The `Id` column in each is an implicit FK — row `n` maps to `BusinessFunctionId` / `ProcessId` / `ActivityId` = `n` in the corresponding base table.

---

### Aggregation Pipeline (CSV → JavaScript)

The dashboard embeds pre-aggregated values — the raw CSVs are never fetched at runtime. To update the dashboard with new data, re-run the aggregations below and patch the four JS data blocks in `Build/state_of_ai_scorecard.html`.

**Viz 1 — State of AI Scorecard** · source: company process tables

```
For each company:
  COUNT rows in {Co}ProcessBusinessFunctionMap WHERE {Co}StateOfAIId = X
  → one count per tier (AI Nativeness, AI Application, AI Adoption, Without AI)

Maps to JS: companies[i] = { aiNative, aiApp, aiAdop, withoutAI }
Constraint: aiNative + aiApp + aiAdop + withoutAI must equal TOTAL_PROCESSES (93)
```

| Company | AI Nativeness | AI Application | AI Adoption | Without AI |
|---|---|---|---|---|
| HUL | 0 | 6 | 79 | 8 |
| P&G | 0 | 27 | 62 | 4 |
| L'Oréal | 0 | 13 | 75 | 5 |
| Nestlé | 0 | 9 | 80 | 4 |

---

**Viz 2 — Use of AI Scorecard** · source: company activity tables

```
For each company:
  COUNT rows in {Co}ActivitiesProcessesMap WHERE {Co}UseOfAIId = X
  → one count per tier (Autonomous AI, AI Assisted, AI Enabled, No AI use at all)

Maps to JS: companies[i] = { autoAI, aiAssist, aiEnable, noAI }
Constraint: autoAI + aiAssist + aiEnable + noAI must equal TOTAL_ACTIVITIES (465)
```

| Company | Autonomous AI | AI Assisted | AI Enabled | No AI use at all |
|---|---|---|---|---|
| HUL | 0 | 143 | 41 | 281 |
| P&G | 0 | 126 | 144 | 195 |
| L'Oréal | 0 | 142 | 57 | 266 |
| Nestlé | 0 | 145 | 57 | 263 |

---

**Viz 3 — Business Function × State of AI** · source: company BF tables

```
For each company:
  SELECT {Co}StateOfAIId for rows Id = 1–10 (one per BF, in BF order)

Maps to JS: companies[i].bfStates = [int × 10]
  where each int is a StateOfAIMaster.StateOfAIComponentId (1–4)
  and array index 0 = BF 1 (Marketing), ..., index 9 = BF 10 (Corporate Affairs)
```

Current value for all companies, all BFs: `2` (AI Adoption) — every cell. The AI Nativeness, AI Application, and Without AI rows in Viz 3 are intentionally empty; this reflects the actual dataset, not a rendering bug.

---

**Viz 4 — Process × Use of AI** · source: company process tables + base process table

```
For each company:
  JOIN {Co}ProcessBusinessFunctionMap (Id) WITH ProcessBusinessFunctionMap (ProcessId)
    → get BusinessFunctionId for each of the 93 process rows
  GROUP BY BusinessFunctionId, {Co}UseOfAIId → COUNT(ProcessId)
  → 10 BFs × 4 Use of AI tiers = 40 counts per company

Maps to JS: companies[i].bfCounts = [
  [autonomousAI_count, aiEnabled_count, aiAssisted_count, noAI_count],  // BF 1
  [autonomousAI_count, aiEnabled_count, aiAssisted_count, noAI_count],  // BF 2
  ...                                                                    // BF 3–10
]
```

> ⚠️ Inner array column order is `[Autonomous AI (Id=3), AI Enabled (Id=2), AI Assisted (Id=1), No AI (Id=4)]` — matching Viz 4's top-to-bottom row order. This is **not** ascending numeric Id order.

> ⚠️ BF row alignment: for each BF, the sum across all 4 tier counts must be identical for all 4 companies. If BF 1 has 10 processes for HUL, it must have 10 for P&G, L'Oréal, and Nestlé. Mismatches cause fixed-height rows to misalign across company columns with no error thrown.

---

### Data Constraints

| Constraint | Rule | Consequence if violated |
|---|---|---|
| Process totals (Viz 1) | Each company's 4 tier counts sum to `TOTAL_PROCESSES = 93` | Bar percentages don't reach 100% |
| Activity totals (Viz 2) | Each company's 4 tier counts sum to `TOTAL_ACTIVITIES = 465` | Bar percentages don't reach 100% |
| BF process alignment (Viz 4) | For each BF, total process count across tiers is identical across all 4 companies | Same BF lands on different row heights across columns — silent misalignment |
| BF headcount totals | Sum of `NumberOfEmployees` across all 10 BF rows = `TotalNumberOfEmployees` in `FMCGCompaniesMaster` | Headcount figures are internally inconsistent |
| UseOfAI sort order | `bfCounts` inner array follows semantic rank order `[Id=3, Id=2, Id=1, Id=4]`, not ascending numeric | Wrong processes appear in wrong Viz 4 rows |

---

## The Four Visualisations

### Viz 1 — State of AI Scorecard
- **Type**: Rank leaderboard (horizontal stacked bars)
- **Data**: Process counts per State of AI tier, per company (total: 93 processes)
- **Ranked by**: Processes at AI Application state
- **Tiers**: AI Nativeness (purple) → AI Application (blue) → AI Adoption (green) → Without AI (red)

### Viz 2 — Use of AI Scorecard
- **Type**: Rank leaderboard (horizontal stacked bars)
- **Data**: Activity counts per Use of AI tier, per company (total: 465 activities)
- **Ranked by**: Activities at AI Assisted level
- **Tiers**: Autonomous AI (purple) → AI Assisted (green) → AI Enabled (blue) → No AI (red)
- **Note**: Bar segments render AI Assisted before AI Enabled (most-volume tier leads); semantic hierarchy is Autonomous AI > AI Enabled > AI Assisted > No AI

### Viz 3 — Business Function × State of AI
- **Type**: Spectrum grid (5-column: label + 4 company columns)
- **Data**: State of AI assignment per business function per company
- **Rows**: AI Nativeness ⭐ → AI Application → AI Adoption → Without AI
- **Cells**: BF name chips coloured by tier
- **Data note**: In the current dataset all 10 business functions for all 4 companies sit at AI Adoption level; the AI Nativeness, AI Application, and Without AI rows are intentionally empty (shown as dashes)

### Viz 4 — Process × Use of AI
- **Type**: Spectrum grid with fixed-height aligned rows
- **Data**: Process counts by Use of AI tier, per business function, per company
- **Rows**: Autonomous AI ⭐ → AI Enabled → AI Assisted → No AI Use at All
- **Key feature**: Fixed 30px rows with gap-rows ensure the same BF always sits on the same visual row across all company columns

## CSS Design System

```css
--ai-native:  #7B3FCC   /* purple  — AI Nativeness / Autonomous AI */
--ai-app:     #2E7FD4   /* blue    — AI Application / AI Enabled */
--ai-adop:    #1A9968   /* green   — AI Adoption / AI Assisted */
--no-ai:      #D13B4B   /* red     — Without AI / No AI Use at All */
--subject-accent: #E85D20   /* orange  — subject company highlight */
```

Bar track: `height: 52px; border-radius: 8px; overflow: hidden` — `overflow:hidden` clips segments cleanly without per-segment border-radius.

Spectrum grid: `grid-template-columns: 150px 1fr 1fr 1fr 1fr`

## Infrastructure

| Resource | Value |
|---|---|
| S3 bucket | `fmcg-ai-benchmarking-dashboard` |
| S3 region | `ap-south-1` (Mumbai) |
| CloudFront distribution | `E3EF4S9G53A1LL` |
| CloudFront domain | `d2hzpx71woh3es.cloudfront.net` |
| Live URL | `https://d2hzpx71woh3es.cloudfront.net/state_of_ai_scorecard.html` |

## AWS Setup (from scratch)

```bash
# 1. Create S3 bucket
aws s3api create-bucket \
  --bucket YOUR-BUCKET-NAME \
  --region ap-south-1 \
  --create-bucket-configuration LocationConstraint=ap-south-1

# 2. Disable block public access
aws s3api delete-public-access-block --bucket YOUR-BUCKET-NAME

# 3. Set public read policy
aws s3api put-bucket-policy --bucket YOUR-BUCKET-NAME --policy '{
  "Version":"2012-10-17",
  "Statement":[{"Effect":"Allow","Principal":"*",
    "Action":"s3:GetObject","Resource":"arn:aws:s3:::YOUR-BUCKET-NAME/*"}]
}'

# 4. Enable static website hosting
aws s3 website s3://YOUR-BUCKET-NAME \
  --index-document state_of_ai_scorecard.html

# 5. Upload files
aws s3 sync Build s3://YOUR-BUCKET-NAME \
  --cache-control "no-cache, no-store, must-revalidate"

# 6. Create CloudFront distribution
# Write the distribution config to a temp file, then create
cat > /tmp/cf-dist.json << 'EOF'
{
  "CallerReference": "fmcg-dashboard-$(date +%s)",
  "Comment": "FMCG AI Benchmarking Dashboard",
  "Enabled": true,
  "DefaultRootObject": "state_of_ai_scorecard.html",
  "Origins": {
    "Quantity": 1,
    "Items": [{
      "Id": "s3-website",
      "DomainName": "YOUR-BUCKET.s3-website.ap-south-1.amazonaws.com",
      // Note: endpoint format varies by region.
      // ap-south-1, eu-west-1, etc. → bucket.s3-website.REGION.amazonaws.com (dot-separated)
      // us-east-1                   → bucket.s3-website-us-east-1.amazonaws.com (dash after "s3-website")
      "CustomOriginConfig": {
        "HTTPPort": 80,
        "HTTPSPort": 443,
        "OriginProtocolPolicy": "http-only"
      }
    }]
  },
  "DefaultCacheBehavior": {
    "TargetOriginId": "s3-website",
    "ViewerProtocolPolicy": "redirect-to-https",
    "AllowedMethods": {
      "Quantity": 2,
      "Items": ["GET", "HEAD"],
      "CachedMethods": { "Quantity": 2, "Items": ["GET", "HEAD"] }
    },
    "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
    "Compress": true
  }
}
EOF
aws cloudfront create-distribution --distribution-config file:///tmp/cf-dist.json
# Note the "DomainName" in the output (e.g. d1234abc.cloudfront.net) — that is your live URL.
# Status will show "InProgress" for ~15 minutes; the domain is usable immediately.

# 7. Invalidate on update
aws cloudfront create-invalidation \
  --distribution-id YOUR-DISTRIBUTION-ID --paths "/*"
```

## Key Design Decisions

**Single file over separate JS/CSS bundles** — Eliminates all module resolution, bundling, and build tooling. The file is large but self-contained. Any engineer can read the source without a build environment.

**Data embedded in JS over a runtime API** — Pre-embedding removes all latency, CORS configuration, and server dependency. Acceptable because the underlying benchmarking data changes on a quarterly cycle, not in real time.

**CloudFront over direct S3 URL** — S3 static website endpoints are HTTP-only. CloudFront provides HTTPS, global edge caching, and professional-grade URLs suitable for executive sharing.

**Fixed 30px BF rows (Viz 4)** — The spectrum grid computes a global list of active business functions per tier across all companies before rendering. Each company column iterates this global list in order, emitting a chip row for BFs with data or a gap row where count=0. This ensures cross-column visual alignment without CSS Grid subgrid (not universally supported) or JavaScript layout measurement.
