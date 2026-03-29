# Scaffold MvR REFramework Robot

Follow these steps exactly, in order:

## Step 1 — Read standards
Read `mvr-rpa-standards.json` completely. The canonical rules for XAML generation, naming, config
and error handling are all defined there. Pay special attention to `xaml_generation_rules`
and `sdd_generation` (defines the SDD structure and section-to-scaffold mapping).
Do not proceed without reading it.

**XAML reference documents** — when generating any XAML activity, consult the matching reference
file in `uipath-ai-skills/uipath-core/references/` rather than inventing syntax:

| Activity type | Reference file |
|---|---|
| LogMessage, Assign, Comment | `xaml-foundations.md` |
| InvokeWorkflowFile, argument wiring | `xaml-invoke.md` |
| TryCatch, Throw, Rethrow | `xaml-error-handling.md` |
| If, ForEach, While, Flowchart | `xaml-control-flow.md` |
| GetRobotCredential, GetRobotAsset, queue activities | `xaml-orchestrator.md` |
| Excel, Email, PDF | `xaml-integrations.md` |
| NClick, NTypeInto, NApplicationCard, selectors | `xaml-ui-automation.md` |
| DataTable operations | `xaml-data.md` |

Read only the sections relevant to the activities in the current process. For large files use
targeted reads — grep for the specific activity name rather than reading the full file.

## Step 2 — Locate SDD
Search for an SDD file matching `*/Documentation/SDD_*.md` (i.e. inside any project folder's
`Documentation/` subfolder at the repo root).
- If **no SDD** is found: stop immediately. Inform the developer that no SDD exists yet.
  Instruct them to run `/solutions` first (with the PDD in `Documentation/`) to generate the SDD,
  then return to `/scaffold-robot`.
- If **multiple SDDs** are found: list them (with their project folder) and ask which one to use.

## Step 3 — Read SDD and extract
Read the SDD. Extraction is **table lookups** from structured Markdown tables — do not infer
from free text. Every field below has a defined SDD section and column.
If any required field is `<<ONBEKEND>>`, surface it in the pre-flight summary (Step 5)
and ask for resolution before generating any files.

| What to extract | SDD section | Field / rule |
|---|---|---|
| Projectnaam (folder name) | 1.1 Samenvatting | `Projectnaam` row |
| Beschrijving (project.json) | 1.1 Samenvatting | `Beschrijving` row |
| isAttended flag | 1.1 Samenvatting | `Type robot` (`Attended` = true, others = false) |
| Transactietype | 1.1 Samenvatting | `Transactietype` row |
| Dispatcher/Performer | 1.1 Samenvatting | `Dispatcher/Performer` row |
| Opdrachtgever, Proceseigenaar | Stakeholders | `Naam (Klant)` column |
| Workflow stubs (filename + order) | 1.2 Procesflow | Rows where `Handmatig / Geautomatiseerd` = `Geautomatiseerd` → filename = zero-pad(Nr.,3) + `_` + Applicatie + `_` + Sub-proces (PascalCase) + `.xaml` |
| Process.xaml root annotation text | 1.3 Decompositie | Numbered list (incl. uitvalpad bullet) |
| Applications + browsers | 1.4 Rechten/applicaties | `Applicatie` and `Browser` columns |
| OrchestratorQueueName | 2.1 Algemeen | `Queue naam` row |
| ProjectWithDispatcher | 2.1 Algemeen | `Dispatcher aanwezig` row (Ja = True) |
| QueueRetry | 2.1 Algemeen | `QueueRetry` row |
| MaxRetryNumber_FrameworkRetry | 2.1 Algemeen | `MaxRetry (framework)` row |
| BusinessRuleException triggers | 2.2 Queue/retry | `BusinessRuleException` rows, `Trigger` column |
| Full asset list | 2.3 Init-fase | **Entire asset table** — this is the only source for Assets + CreateAssets sheets |
| Per-stub filename | 4.N.1 | `Workflowbestand` row (must match 1.2 derivation) |
| Per-stub Doel (annotation line 1) | 4.N.1 | `Doel` row |
| Per-stub input/output arguments | 4.N.1 | `Input argumenten` and `Output argumenten` rows |
| Per-stub TODO steps (annotation) | 4.N.2 | Numbered list of handelingen |
| logF_BusinessProcessName | 1.1 Samenvatting | `Procesnaam` row |
| LogMessageHtml var_subject/var_procesnaam | 1.1 Samenvatting | `Procesnaam` row |

## Step 4 — Map assets from SDD to Config sheets

Read the master asset table from SDD Section 2.3. Every row maps to **both** Config sheets:

**Assets sheet** (columns: Name | Asset | Folder | Description)
- Name = `Asset naam` column value
- Asset = `Asset naam` column value
- Folder = blank
- Description = `Omschrijving` column value

**CreateAssets sheet** (columns: Name | ValueType | Value | CreateAsset | Comments)
- Name = `Asset naam` column value
- ValueType = `Type` column value
- Value = `Waarde (indien bekend)` column value (empty string for Credential rows)
- CreateAsset = True (always)
- Comments = `Omschrijving` column value

The same asset must appear in BOTH sheets — Assets sheet for runtime lookup, CreateAssets sheet for Orchestrator provisioning.

## Step 5 — Pre-flight risk summary (output BEFORE generating any files)
Before writing any file, print a pre-flight summary:
- **Project name derived**: confirm Department_ProcessName from SDD 1.1
- **Transaction type**: queue-based or linear (from SDD 1.1)
- **`<<ONBEKEND>>` fields in SDD**: list each one with its section — these must be resolved before proceeding
- **Credentials / assets that need manual completion**: Credential-type rows with empty Waarde
- **Process steps requiring UI automation**: the Geautomatiseerd rows from SDD 1.2 (become TODO stubs)
- **Risks**: anything that could cause the generated scaffold to need rework

Then ask the user: "Confirm to proceed, or correct any of the above."
Do not generate files until the user confirms.

## Step 6 — Scaffold project structure
After user confirmation, generate the following folder and file structure inside the project folder.
The project folder (`ProjectName/`) may already exist — if so, add files into it without removing
anything that is already there. Check the `MvR_REFramework/` folder in this repository for the
current framework file list — use it as the authoritative source for framework stubs rather than
a hardcoded list.

```
ProjectName/
  ├── Documentation/                     (already exists — SDD files are here; do not overwrite)
  ├── Main.xaml                          (REF state machine stub — copy from MvR_REFramework/)
  ├── project.json                       (correct name, description from SDD — see GUID note below)
  ├── CHANGELOG.md                       (empty, ready for versioning)
  ├── README.md                          (process summary, app list, queue/asset names)
  ├── Data/
  │   └── Config.xlsx                    (Settings, Constants, Assets, CreateAssets, LogMessageHtml, LogmailAttachment — all pre-populated)
  ├── Documentation/
  │   └── Dictionaries.xlsx              (only if Dictionary variables are identified)
  ├── Framework/
  │   └── (stubs matching MvR_REFramework/ file list)
  └── Processes/
      └── NNN_AppName_Action.spec.json   (one spec per Geautomatiseerd step — XAML generated from spec in Step 6c)
```

**project.json GUIDs**: generate two fresh random GUIDs — one for `projectId` and one for
`entryPoints[0].uniqueId`. Never copy GUIDs from another project. Never leave them as empty strings.

**project.json NuGet versions — resolve from feed (Rule G-5: never guess versions):**
After writing project.json with placeholder versions, run `resolve_nuget.py` to update every
dependency to the real latest stable version.

Read the `NuGet packages` row from SDD Section 2.1 — `/solutions` pre-computed this list from
the process steps. Use exactly those packages (always add `UiPath.System.Activities` if not listed):

```bash
# Build the package list from SDD Section 2.1 "NuGet packages" row, then run:
python uipath-ai-skills/uipath-core/scripts/resolve_nuget.py \
  --add "ProjectName" \
  UiPath.System.Activities \
  UiPath.UIAutomation.Activities \
  [... other packages from SDD 2.1 ...]
```

The script updates `project.json` in-place and skips packages already at the latest version.
`UiPath.Testing.Activities` is always needed for REFramework test cases.

## Step 6a — Populate Config.xlsx

Copy `MvR_REFramework/Data/Config.xlsx` to `ProjectName/Data/Config.xlsx` (this template already
contains all 6 sheets with correct headers). Then use `config_xlsx_manager.py` to add rows for
every asset from SDD Section 2.3.

**Settings sheet** — use for runtime string values (queue name, process name, credential asset name references):
```bash
python uipath-ai-skills/uipath-core/scripts/config_xlsx_manager.py add "ProjectName" \
  --sheet Settings --key "logF_BusinessProcessName" --value "Procesnaam" --desc "Process name for log fields"
```

**Constants sheet** — use for framework tuning values (retry counts, flags, log message templates):
```bash
python uipath-ai-skills/uipath-core/scripts/config_xlsx_manager.py add "ProjectName" \
  --sheet Constants --key "MaxRetryNumber" --value "3" --desc "Max retries per transaction"
```

**Assets sheet** — use for Orchestrator-managed assets (credentials, URLs, paths):
```bash
python uipath-ai-skills/uipath-core/scripts/config_xlsx_manager.py add "ProjectName" \
  --sheet Assets --key "AppName_Credential" --asset "AppName_Credential" --folder "" \
  --desc "Login credential for AppName"
```

Run one command per row in the SDD 2.3 asset table. Apply all 4 default framework rows
(MaxTransactions, Folder_Temp, Folder_Log, LogMessageAddress) plus all process-specific rows.

> **Note**: The script only covers Settings, Constants, and Assets sheets. The **CreateAssets**,
> **LogMessageHtml**, and **LogmailAttachment** sheets must be populated manually in the copied
> Config.xlsx after the script runs — use the asset table from SDD 2.3 as the source.

## Step 6b — Generate Process.xaml (fully scaffolded)
Process.xaml must be generated as completely as possible from the SDD. See `mvr-rpa-standards.json` → `process_xaml_scaffolding` for the full rules. Key points:

**Init vs Process split (important):**
- Any workflow that **opens an application or logs in** (e.g. `NNN_AppName_OpenEnInloggen.xaml`) is invoked from `Framework/InitAllApplications.xaml` — NOT from Process.xaml. It runs once per robot run.
- Only **per-transaction business logic steps** go in Process.xaml. These are steps that act on a specific queue item.
- When a login/open step is moved to InitAllApplications, add a note to the Process.xaml root annotation: `"NB: [AppName] openen en inloggen ([filename]) wordt uitgevoerd vanuit Framework/InitAllApplications.xaml — niet hier."`

- **Root Sequence annotation**: process name, note about init steps in InitAllApplications (if any), numbered per-transaction steps from SDD, uitvalpad. `IsAnnotationDocked: True`.
- **Variables block**: After writing Process.xaml, declare all inter-workflow data variables using
  `modify_framework.py add-variables` rather than hand-writing the Variables XML block:
  ```bash
  python uipath-ai-skills/uipath-core/scripts/modify_framework.py add-variables \
    "ProjectName/Process.xaml" \
    "uiAppName:UiElement" \
    "dtResult:DataTable" \
    "boolSuccess:Boolean"
  ```
  Type shortcuts: `String` (default if omitted), `Boolean`, `Int32`, `DataTable`, `UiElement`,
  `Dictionary`, `QueueItem`. Derive variable names from the SDD 4.N.1 output arguments and the
  UiElement variables needed by each process stub.
- **LogMessage** first: `"In: Process - ProjectName"` at Info level.
- **TryCatch** wrapping the main flow:
  - **Try → "Try - Hoofdproces"**: one `ui:InvokeWorkflowFile` per Geautomatiseerd step from SDD 1.2, in order.
    - `DisplayName` = `"NNN - Description (AppName)"`
    - `WorkflowFileName` = correct relative path to the sub-workflow stub
    - `Arguments` = pre-wired from SDD 4.N.1 — do NOT leave empty. For each stub:
      - `in_Config` ← `in_Config` (always)
      - `in_TransactionItem` ← `io_TransactionData` (queue-based) or omit (linear)
      - `io_uiAppName` ← `uiAppName` local variable (for UI stubs — variable must be declared in Process.xaml Variables block)
      - Any `out_X` from SDD 4.N.1 ← matching local variable declared in Variables block
    - `sap2010:Annotation.AnnotationText` = list only arguments that still need Studio attention (e.g. complex expressions, dynamic values). If all arguments are pre-wired, annotation = `"Controleer argumenten na /build."` with `IsAnnotationDocked: True`
  - Conditional SDD logic → generate `If` activities with bool_ variable conditions and InvokeWorkflowFile calls in Then/Else branches.
  - Final **LogMessage** at Info level logging key result variables.
  - **Catch `ui:BusinessRuleException`**: LogMessage Warn (BRE trigger text from SDD 2.2) + InvokeWorkflowFile for error handler (if SDD specifies one, with wiring TODO annotation) + Rethrow.
- Sub-workflow stubs (`NNN_AppName_Action.xaml`) carry the implementation TODOs in their root Sequence annotation — Process.xaml does not.

## Step 6c — Generate Processes/ stubs via spec → generate_workflow.py

**Do NOT hand-write XAML for Processes/ stubs (Rule G-1).** Instead: write the spec, run the
generator, validate. This is the only permitted path for producing stub XAML files.

For each `Processes/NNN_AppName_Action.xaml` stub, write a
`Processes/NNN_AppName_Action.spec.json` file first, then generate the XAML from it. The
`/build` skill also reads this spec — the richer it is, the less build has to re-derive from the SDD.

**Spec structure — populate every field:**

```json
{
  "class_name": "NNN_AppName_Action",
  "_build": {
    "stub_type": "init",
    "invoked_from": "InitAllApplications",
    "app_name": "AppName",
    "app_type": "web",
    "browser": "Edge",
    "credential_key": "AppName_Credential",
    "url_key": "AppName_URL",
    "target_selector": "<<INSPECT_REQUIRED>>",
    "handelingen": [
      "1. Get Credential via in_Config(\"AppName_Credential\")",
      "2. Open AppName browser op in_Config(\"AppName_URL\")",
      "3. TypeInto gebruikersnaamveld",
      "4. TypeInto wachtwoordveld",
      "5. Click inlogknop",
      "6. Check App State: wacht tot hoofdpagina geladen is"
    ]
  },
  "arguments": [
    {"name": "in_Config", "direction": "In", "type": "Dictionary"},
    {"name": "out_uiAppName", "direction": "Out", "type": "UiElement"}
  ],
  "variables": [],
  "activities": [
    {"gen": "log_message", "args": {"message_expr": "\"In: NNN_AppName_Action\"", "level": "Info"}},
    {"gen": "comment", "args": {"text": "TODO: implement — see _build.handelingen"}}
  ]
}
```

**`_build` field rules** — populate from the SDD:

| `_build` field | Source | Values |
|---|---|---|
| `stub_type` | Filename contains `OpenEnInloggen`/`Inloggen` → `"init"`, else → `"process"` | `"init"` / `"process"` / `"close"` |
| `invoked_from` | Init stubs → `"InitAllApplications"`, process stubs → `"Process.xaml"` | |
| `app_name` | SDD 1.2 `Applicatie` column for this row | exact application name |
| `app_type` | SDD 1.4 `Type` column (`Web` / `Desktop`) | `"web"` / `"desktop"` |
| `browser` | SDD 1.4 `Browser` column (blank for desktop) | `"Edge"` / `"Chrome"` / `null` |
| `credential_key` | SDD 2.3 asset row where Type = `Credential` for this app | `"AppName_Credential"` |
| `url_key` | SDD 2.3 asset row where Type = `Text` and name ends in `_URL` for this app | `"AppName_URL"` / `null` for desktop |
| `target_selector` | Unknown until UI inspection — always set to `"<<INSPECT_REQUIRED>>"` | |
| `handelingen` | SDD 4.N.2 numbered list — **copy verbatim as a JSON array** | one string per step |

**`arguments` field rules:**

- Always include `in_Config` (type `"Dictionary"`)
- **Init stubs**: add `out_uiAppName` (type `"UiElement"`, direction `"Out"`) — variable name = `out_ui` + AppName (no spaces, PascalCase)
- **Process stubs (Web UI or Desktop UI)**: add `io_uiAppName` (type `"UiElement"`, direction `"InOut"`) — same naming as init output
- **Process stubs (queue-based)**: add `in_TransactionItem` (type `"QueueItem"`, direction `"In"`)
- **Linear robots**: omit `in_TransactionItem`; **attended robots**: omit both queue and UiElement args
- All other output arguments from SDD 4.N.1 `Output argumenten`

**Example — Init stub (web, queue-based):**
```json
{
  "class_name": "001_Suite_OpenEnInloggen",
  "_build": {
    "stub_type": "init",
    "invoked_from": "InitAllApplications",
    "app_name": "Suite",
    "app_type": "web",
    "browser": "Edge",
    "credential_key": "Suite_Credential",
    "url_key": "Suite_URL",
    "target_selector": "<<INSPECT_REQUIRED>>",
    "handelingen": [
      "1. Get Credential via in_Config(\"Suite_Credential\")",
      "2. Open Suite browser op in_Config(\"Suite_URL\")",
      "3. TypeInto gebruikersnaamveld (selector vereist)",
      "4. TypeInto wachtwoordveld (selector vereist)",
      "5. Click inlogknop (selector vereist)",
      "6. Check App State: wacht tot hoofdpagina geladen is"
    ]
  },
  "arguments": [
    {"name": "in_Config", "direction": "In", "type": "Dictionary"},
    {"name": "out_uiSuite", "direction": "Out", "type": "UiElement"}
  ],
  "variables": [],
  "activities": [
    {"gen": "log_message", "args": {"message_expr": "\"In: 001_Suite_OpenEnInloggen\"", "level": "Info"}},
    {"gen": "comment", "args": {"text": "TODO: implement — see _build.handelingen"}}
  ]
}
```

**Example — Process stub (web UI, queue-based):**
```json
{
  "class_name": "002_Suite_OpenWerkproces",
  "_build": {
    "stub_type": "process",
    "invoked_from": "Process.xaml",
    "app_name": "Suite",
    "app_type": "web",
    "browser": "Edge",
    "credential_key": null,
    "url_key": "Suite_URL",
    "target_selector": "<<INSPECT_REQUIRED>>",
    "handelingen": [
      "1. Attach aan open Suite browser",
      "2. Navigeer naar werkprocespagina via in_TransactionItem WIID",
      "3. Klik knop 'Werkproces openen'"
    ]
  },
  "arguments": [
    {"name": "in_Config", "direction": "In", "type": "Dictionary"},
    {"name": "in_TransactionItem", "direction": "In", "type": "QueueItem"},
    {"name": "io_uiSuite", "direction": "InOut", "type": "UiElement"}
  ],
  "variables": [],
  "activities": [
    {"gen": "log_message", "args": {"message_expr": "\"In: 002_Suite_OpenWerkproces\"", "level": "Info"}},
    {"gen": "comment", "args": {"text": "TODO: implement — see _build.handelingen"}}
  ]
}
```

**After writing each .spec.json, immediately run generate_workflow.py to produce the .xaml:**

```bash
# Run from repo root — repeat for each stub
python uipath-ai-skills/uipath-core/scripts/generate_workflow.py \
  "ProjectName/Processes/NNN_AppName_Action.spec.json" \
  "ProjectName/Processes/NNN_AppName_Action.xaml" \
  --project-dir "ProjectName"
```

Then validate each generated file before moving to the next stub:
```bash
python -m validate_xaml "ProjectName/Processes/NNN_AppName_Action.xaml" --lint --errors-only
```
Fix any errors before writing the next spec. This keeps each stub clean before proceeding.

All 94 supported generators and the full JSON spec format are documented in
`uipath-ai-skills/uipath-core/references/generation.md`.

## Step 7 — XAML generation rules
Follow `mvr-rpa-standards.json` → `xaml_generation_rules` for all generated XAML. Key points:
- Try-Catch in Process.xaml: ONLY catch BusinessRuleException
- Never generate Catch(Exception) or any generic base exception catch
- Never use `x:` prefix for .NET types in XAML type arguments
- Begin every workflow with a Log Message: `"In: WorkflowName"`
- All argument names: in_/out_/io_ prefix + PascalCase

**TODO patterns — see `mvr-rpa-standards.json` → `todo_in_xaml` for the full rules and XAML example:**
- NEVER use XML comments (`<!-- TODO -->`) for developer instructions — they are invisible in UiPath Studio's designer.
- Every stub activity (both the workflow root Sequence and individual placeholder Sequences inside it) gets a meaningful `DisplayName` describing what the activity does — never a `[TODO]` prefix in the name.
- Put TODO instructions in `sap2010:Annotation.AnnotationText` on the activity. Always set `IsAnnotationDocked: True` in the ViewState so the annotation opens automatically when the developer opens the file.
- Annotation text structure: activity purpose → numbered TODO steps derived from SDD 4.N.2. Use `&#xA;` for newlines and `&quot;` for quotes inside the attribute value.
- See `mvr-rpa-standards.json` → `todo_in_xaml` for the full rules and a XAML example.

## Step 7b — Validate generated XAML

After all XAML files are written, run the validator to catch structural and lint issues before
the developer opens Studio:

```bash
cd uipath-ai-skills/uipath-core/scripts
python -m validate_xaml "../../../ProjectName" --lint --errors-only
```

- If the validator reports **errors**: fix them before proceeding to Step 8.
  See `uipath-ai-skills/uipath-core/references/lint-reference.md` for each rule.
- **Warnings** are non-blocking — include them in the Step 8 summary.
- To auto-fix deterministic violations (lints 53, 54, 71, 83, 87, 89, 90, 93, 99):
  ```bash
  python -m validate_xaml "../../../ProjectName" --lint --fix
  ```

Then cross-reference Config.xlsx against all generated XAML to ensure every `Config("key")`
reference has a matching entry:

```bash
cd ../../../
python uipath-ai-skills/uipath-core/scripts/config_xlsx_manager.py validate "ProjectName"
```

Add any keys flagged as **MISSING** to the appropriate Config.xlsx sheet before confirming.

## Step 8 — Confirm
After generating all files, print a summary:
- Project name used
- List of generated workflow stubs (`.xaml`) and their companion spec files (`.spec.json`)
- List of Config.xlsx entries added per sheet (Settings, Constants, Assets) via script
- List of assets added to CreateAssets sheet (manually)
- Validation result: errors fixed, warnings to review
- List of TODOs the developer must complete in UiPath Studio
