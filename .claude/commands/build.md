# Build MvR Robot Workflows

Implements the `Processes/` stub workflows created by `/scaffold-robot` using the
deterministic Python generators from `uipath-ai-skills/uipath-core/`.

**Prerequisite**: `/scaffold-robot` must have been run first. The project must contain
`Processes/NNN_*.xaml` stubs and matching `Processes/NNN_*.spec.json` files.

> ⛔ **Safety rules (from uipath-ai-skills)**
> - Never hand-write XAML — always use `generate_workflow.py` (Rule G-1)
> - Write JSON specs to disk before running the generator (Rule G-2)
> - Playwright inspection is READ-ONLY. HALT at any login page and wait for the user (Rule I-2)
> - Never generate credentials, tokens, or passwords

**Generate one workflow at a time — validate before starting the next.**

---

## Phase 1 — Read standards and reference docs

Read `mvr-rpa-standards.json` fully. Then, based on the process type, read the relevant
reference files from `uipath-ai-skills/uipath-core/references/`:

| What is in the process | Read |
|---|---|
| Any workflow (always) | `xaml-foundations.md`, `xaml-invoke.md`, `xaml-error-handling.md` |
| Web UI automation | `xaml-ui-automation.md` (grep for activity name — large file) |
| Credentials, queue items, assets | `xaml-orchestrator.md` |
| Excel, Email, PDF | `xaml-integrations.md` |
| DataTable operations | `xaml-data.md` |
| If/ForEach/While | `xaml-control-flow.md` |
| JSON spec patterns and generators | `cheat-sheet.md` (full read — critical) |
| Naming and decomposition rules | `decomposition.md` (first 80 lines) |

Do not proceed without reading `cheat-sheet.md` — it contains exact generator syntax,
common crash patterns, and valid enum values that are not in any other reference.

---

## Phase 2 — Locate project and list stubs

1. Find the SDD matching `*/Documentation/SDD_*.md`. If multiple, ask which project to build.
2. Read the SDD. Extract per-stub data from **Section 4** (one entry per automated step):
   - `Workflowbestand` (filename)
   - `Doel` (purpose)
   - `Input argumenten` / `Output argumenten`
   - `Omschrijving handelingen` (the numbered steps — these drive implementation)
3. Scan `Processes/` and list every `.spec.json` file.
4. Cross-reference: every SDD Section 4 entry should have a matching `.spec.json`. If a stub
   has no `.spec.json` (scaffold was run before Step 6c), create the spec file manually from
   the SDD before continuing — do NOT skip it.
5. Print the build list as a table:

| # | File | Type | Handelingen | Selectors needed? |
|---|---|---|---|---|
| 001 | 001_Suite_OpenEnInloggen.xaml | Init | 6 steps | Yes — web login |
| 002 | 002_Suite_OpenWerkproces.xaml | Process | 4 steps | Yes — web UI |
| ... | | | | |

Ask: **"Confirm build order, or adjust."** Do not generate any XAML until confirmed.

---

## Phase 3 — Classify each stub

Before generating, classify every stub. This determines the generator strategy:

| Stub type | Classification criteria | Key generators |
|---|---|---|
| **Init** | Filename contains `OpenEnInloggen` or `Inloggen` | `napplicationcard_open` / `napplicationcard_desktop_open`, `getrobotcredential`, `ntypeinto`, `nclick`, `pick_login_validation` |
| **Web UI** | Steps mention browser, navigeren, klikken, typen | `napplicationcard_attach`, `ngotourl`, `ntypeinto`, `nclick`, `nselectitem`, `ncheck`, `ngettext`, `nextractdata` |
| **Desktop UI** | Steps mention desktopapplicatie, scherm, tab | `napplicationcard_desktop_open/attach`, `ntypeinto`, `nclick`, `ncheckstate` |
| **Data** | Steps mention Excel, DataTable, CSV, PDF | `read_range`, `foreach_row`, `filter_data_table`, `read_pdf_text` |
| **API** | Steps mention HTTP, REST, API call | `net_http_request`, `deserialize_json` (built-in retry — do NOT add RetryScope) |
| **Queue** | Steps mention queue item, transaction data | `get_queue_item`, `add_queue_item` (wrap in `retryscope`) |
| **Orchestrator** | Steps mention asset, credential | `get_robot_asset`, `getrobotcredential` (wrap in `retryscope`) |

A stub can have multiple types (e.g., Init stubs are always UI + Credential).

---

## Phase 4 — UI inspection (BEFORE writing any spec with selectors)

**Only run this phase for stubs classified as Web UI, Desktop UI, or Init (web/desktop).**
For Data/API/Queue stubs: skip to Phase 5.

### Web app inspection (Playwright MCP)

Follow the 5-step Playwright workflow from
`uipath-ai-skills/uipath-core/references/ui-inspection.md`:

1. Navigate to the app URL
2. ⛔ **LOGIN GATE**: If redirected to a login page — STOP. Do not inspect further.
   Write to the user:
   > "Ik heb de loginpagina van [AppName] bereikt. Ik kan niet inloggen.
   > Wil je zelf inloggen en daarna bevestigen? Dan ga ik verder met de inspectie."
   Wait for user confirmation before continuing.
3. After login (user confirmed): snapshot each relevant screen
4. Map Playwright element IDs to UiPath selectors — use `playwright-selectors.md` as mapping reference
5. Build `selectors.json` for this app (format from `cheat-sheet.md` § selectors.json)
   and save it to `ProjectName/selectors.json`

### Desktop app inspection

```bash
# Run from repo root (Windows only)
powershell -File uipath-ai-skills/uipath-core/scripts/inspect-ui-tree.ps1
```

Map properties to selectors per `ui-inspection-reference.md`.

### Generate Object Repository (for UI automation stubs only)

After `selectors.json` is complete:

```bash
cd uipath-ai-skills/uipath-core/scripts
python generate_object_repository.py \
  --from-selectors "../../../ProjectName/selectors.json" \
  --project-dir "../../../ProjectName"
```

This creates `ProjectName/.objects/` and `ProjectName/.objects/refs.json`.
Pass `--project-dir` to `generate_workflow.py` later so it auto-wires Object Repository
references into all UI activities.

---

## Phase 5 — Generate stubs (one at a time, sequentially)

For **each stub** in the confirmed build order:

### Step 5.1 — Build the JSON spec

Read the stub's existing `.spec.json`. Extend it — do not replace the `class_name` or
`arguments` already set. Build the full `activities` array from the SDD 4.N.2 handelingen.

**Handeling → generator mapping:**

| SDD handeling (Dutch) | Generator | Notes |
|---|---|---|
| Get Credential via `in_Config(...)` | `getrobotcredential` | Wrap in `retryscope` |
| Open browser / navigeer naar URL | `napplicationcard_open` | `OpenMode=Always` |
| Open desktopapplicatie | `napplicationcard_desktop_open` | Use `in_strAppPath` arg |
| Attach aan open browser | `napplicationcard_attach` | `OpenMode=Never` |
| Navigeer naar pagina / URL | `ngotourl` | URL from `in_Config(...)` |
| TypeInto tekstveld / gebruikersnaam | `ntypeinto` | |
| TypeInto wachtwoordveld | `ntypeinto` | `is_secure: true` |
| Klik knop / link | `nclick` | |
| Klik checkbox | `ncheck` | NOT `nclick` |
| Selecteer dropdown / keuzelijst | `nselectitem` | `item_variable` is a VB expression |
| Lees tekst / haal waarde op | `ngettext` | |
| Wacht op pagina / check app state | `pick_login_validation` (for login) or `ncheckstate` | |
| Extraheer tabel | `nextractdata` | `extract_metadata` is REQUIRED |
| Lees Excel bestand | `read_range` | |
| Schrijf naar Excel | `write_range` or `write_cell` | |
| Verwerk rijen DataTable | `foreach_row` | |
| Filter DataTable | `filter_data_table` | Use CAPS enum: `EQ`, `NE`, `GT`, etc. |
| Lees PDF | `read_pdf_text` | |
| HTTP API aanroep | `net_http_request` | Built-in retry — NO extra RetryScope |
| Voeg toe aan queue | `add_queue_item` | Wrap in `retryscope` |
| Haal queue item op | `get_queue_item` | Already wraps in RetryScope |
| Haal asset op | `get_robot_asset` | Wrap in `retryscope` |
| Als / conditie | `if` or `if_else_if` | |
| Gooi BusinessRuleException | `throw` | `New BusinessRuleException("msg")` (VB.NET) |
| Kopieer / verplaats bestand | `copy_file` / `move_file` | |

**Every spec must begin and end with `log_message`:**
```json
{"gen": "log_message", "args": {"message_expr": "\"In: NNN_AppName_Action\"", "level": "Info"}}
```

**MvR argument conventions:**
- Always include `in_Config` (type `Dictionary`) — it carries all Config.xlsx values
- Queue-based robots: include `in_TransactionItem` (type `QueueItem`) on process stubs
- Init stubs (OpenEnInloggen): do NOT include `in_TransactionItem` — these run once per robot run
- Output variables: use `out_` prefix + type prefix (e.g. `out_uiSuite`, `out_dt_Results`)
- All Config lookups: `in_Config("KeyName").ToString` (NOT `Config("KeyName")`)

**Init stub (OpenEnInloggen) — standard spec pattern:**
```json
{
  "class_name": "001_Suite_OpenEnInloggen",
  "arguments": [
    {"name": "in_Config", "direction": "In", "type": "Dictionary"},
    {"name": "out_uiSuite", "direction": "Out", "type": "UiElement"}
  ],
  "variables": [
    {"name": "strUsername", "type": "String"},
    {"name": "secstrPassword", "type": "SecureString"},
    {"name": "strErrorText", "type": "String"},
    {"name": "uiErrorElement", "type": "UiElement"}
  ],
  "activities": [
    {"gen": "log_message", "args": {"message_expr": "\"In: 001_Suite_OpenEnInloggen\"", "level": "Info"}},
    {"gen": "retryscope", "children": [
      {"gen": "getrobotcredential", "args": {
        "asset_name_variable": "in_Config(\"Suite_Credential\").ToString",
        "username_variable": "strUsername",
        "password_variable": "secstrPassword"
      }}
    ]},
    {"gen": "napplicationcard_open",
     "args": {
       "display_name": "Suite",
       "url_variable": "in_Config(\"Suite_URL\").ToString",
       "out_ui_element": "out_uiSuite",
       "target_app_selector": "<html app='msedge.exe' title='Suite*' />"
     },
     "children": [
       {"gen": "ntypeinto", "args": {"display_name": "Type Into 'Gebruikersnaam'", "selector": "<webctrl id='username' tag='INPUT' />", "text_variable": "strUsername"}},
       {"gen": "ntypeinto", "args": {"display_name": "Type Into 'Wachtwoord'", "selector": "<webctrl id='password' tag='INPUT' />", "text_variable": "secstrPassword", "is_secure": true}},
       {"gen": "nclick", "args": {"display_name": "Click 'Inloggen'", "selector": "<webctrl tag='BUTTON' aaname='Inloggen' />"}},
       {"gen": "pick_login_validation", "args": {
         "success_selector": "<webctrl tag='H1' aaname='Dashboard' />",
         "error_selector": "<webctrl tag='DIV' class='alert-danger' />"
       }}
     ]
    },
    {"gen": "log_message", "args": {"message_expr": "\"Out: 001_Suite_OpenEnInloggen\"", "level": "Info"}}
  ]
}
```

Replace selectors with real values from Phase 4 inspection.
`pick_login_validation` requires `strErrorText` and `uiErrorElement` variables to be declared.

**Process stub (web UI action) — standard pattern:**
```json
{
  "class_name": "002_Suite_OpenWerkproces",
  "arguments": [
    {"name": "in_Config", "direction": "In", "type": "Dictionary"},
    {"name": "in_TransactionItem", "direction": "In", "type": "QueueItem"},
    {"name": "io_uiSuite", "direction": "InOut", "type": "UiElement"}
  ],
  "variables": [],
  "activities": [
    {"gen": "log_message", "args": {"message_expr": "\"In: 002_Suite_OpenWerkproces\"", "level": "Info"}},
    {"gen": "napplicationcard_attach",
     "args": {"display_name": "Suite", "ui_element_variable": "io_uiSuite"},
     "children": [
       {"gen": "ngotourl", "args": {"url_variable": "String.Format(\"{0}/werkproces/{1}\", in_Config(\"Suite_URL\").ToString, in_TransactionItem.SpecificContent(\"WIID\").ToString)"}},
       {"gen": "nclick", "args": {"display_name": "Click 'Werkproces openen'", "selector": "<webctrl aaname='Werkproces openen' tag='A' />"}}
     ]
    },
    {"gen": "log_message", "args": {"message_expr": "\"Out: 002_Suite_OpenWerkproces\"", "level": "Info"}}
  ]
}
```

### Step 5.2 — Pre-validate the spec

```bash
cd uipath-ai-skills/uipath-core/scripts
python generate_workflow.py --validate-spec "../../../ProjectName/Processes/NNN_Stub.spec.json"
```

Fix any type errors (e.g. `Dictionary(String,Object)` → `Dictionary`) before generating.

### Step 5.3 — Generate the XAML

```bash
cd uipath-ai-skills/uipath-core/scripts
python generate_workflow.py \
  "../../../ProjectName/Processes/NNN_Stub.spec.json" \
  "../../../ProjectName/Processes/NNN_Stub.xaml" \
  --project-dir "../../../ProjectName"
```

`--project-dir` auto-wires Object Repository references if `.objects/refs.json` exists.

### Step 5.4 — Validate immediately

```bash
python -m validate_xaml "../../../ProjectName/Processes/NNN_Stub.xaml" --lint --errors-only
```

- **Errors**: fix before moving to the next stub. Consult
  `uipath-ai-skills/uipath-core/references/lint-reference.md` for each rule number.
- **Auto-fix** deterministic violations:
  ```bash
  python -m validate_xaml "../../../ProjectName/Processes/NNN_Stub.xaml" --lint --fix
  ```
- **Warnings**: note them — include in Phase 8 summary.

**Do not start the next stub until this file passes with zero errors.**

---

## Phase 6 — Wire framework files

### 6a — Wire InitAllApplications (Init stubs only)

For every stub classified as **Init** (OpenEnInloggen / Inloggen), insert an
`InvokeWorkflowFile` into `Framework/InitAllApplications.xaml`.

Generate the invoke snippet from the spec output, then insert:

```bash
cd uipath-ai-skills/uipath-core/scripts
python modify_framework.py insert-invoke \
  "../../../ProjectName/Framework/InitAllApplications.xaml" \
  "<XAML_SNIPPET_FROM_GENERATOR>"
```

> ⛔ Rule G-3: the XAML snippet passed to `insert-invoke` MUST come from generator output —
> never hand-write it, especially for InvokeWorkflowFile with argument blocks.

The invoke must wire:
- `in_Config` ← `in_Config` (pass-through from InitAllApplications)
- `out_uiAppName` ← the variable declared in InitAllApplications (e.g. `uiSuite`)

If the UiElement variable does not yet exist in InitAllApplications, add it:

```bash
python modify_framework.py add-variables \
  "../../../ProjectName/Framework/InitAllApplications.xaml" \
  "uiSuite:ui:UiElement"
```

### 6b — Update Process.xaml InvokeWorkflowFile arguments

The generated Process.xaml has InvokeWorkflowFile stubs with empty argument dictionaries
and a TODO annotation: `"TODO: Wire arguments"`. After the process stubs are generated you know
the exact argument names and types. Replace the empty Arguments dict in each InvokeWorkflowFile
with the real wiring using `modify_framework.py set-expression`:

```bash
python modify_framework.py set-expression \
  "../../../ProjectName/Process.xaml" \
  "PLACEHOLDER_EXPR" \
  "REAL_EXPRESSION"
```

If the Process.xaml arguments block is too complex for `set-expression`, open the file in
Studio and wire manually — note this as a remaining TODO in the Phase 8 summary.

### 6c — Wire CloseAllApplications (if App_Close stubs exist)

For any `AppName_Close` or `App_Close` style stubs, wire them into
`Framework/CloseAllApplications.xaml` the same way as 6a.

---

## Phase 7 — Final project validation

Run the full project validation after all stubs are generated and wired:

```bash
cd uipath-ai-skills/uipath-core/scripts
python -m validate_xaml "../../../ProjectName" --lint --errors-only
```

Fix all remaining errors. Then cross-reference Config.xlsx:

```bash
cd ../../../
python uipath-ai-skills/uipath-core/scripts/config_xlsx_manager.py validate "ProjectName"
```

Add any **MISSING** Config keys to the appropriate sheet before confirming.

For a dependency graph of the generated project (optional):

```bash
cd uipath-ai-skills/uipath-core/scripts
python -m validate_xaml "../../../ProjectName" --graph
```

---

## Phase 8 — Confirm

Print a summary:

**Generated workflows:**
- List each `Processes/NNN_*.xaml` with: generators used, arguments, validation status

**Framework wiring:**
- Which stubs were inserted into InitAllApplications
- Which stubs were inserted into CloseAllApplications

**Remaining TODOs (must be completed in Studio):**
- Process.xaml InvokeWorkflowFile arguments that need manual wiring
- Selectors that used placeholder values (inspect was halted at login gate)
- `<<FILL_BEFORE_FIRST_RUN>>` Config values that need real values before deployment

**Validation:**
- Error count after fixes
- Warnings to review (with lint rule numbers)
- Config.xlsx: N missing / N unused keys
