# Scaffold MvR REFramework Robot

Follow these steps exactly, in order:

## Step 1 — Read standards
Read `mvr-rpa-standards.json` completely. The canonical rules for XAML generation, naming, config
and error handling are all defined there. Pay special attention to `xaml_generation_rules`.
Do not proceed without reading it.

## Step 2 — Locate PDD
Find the PDD file in this folder (PDD_*.docx or PDD_*.pdf).
If no PDD file is found: stop immediately and ask the user to provide or point to the PDD before continuing.

## Step 3 — Read PDD and extract
Read the PDD and extract:
- Process name and department → determines project/folder naming
- Applications involved → determines workflow file names (NNN_AppName_Action)
- Transaction type → queue-based or linear
- Business rules → what triggers a BusinessRuleException
- Exception scenarios → what the PDD says about error handling
- Config values needed → pre-populate Config.xlsx Settings/Assets/Constants
- Credential names → add as Asset entries in Config.xlsx (names only, no values)
- Start and end triggers

## Step 4 — Map assets from PDD to Config sheets

For every asset, credential, URL or configurable value found in the PDD:

**Assets sheet** (columns: Name | Asset | Folder | Description)
- Add one row per asset using AppName_Purpose naming
- Credential assets: Name = AppName_Credential, Asset = AppName_Credential, Folder = blank
- URL/text assets: Name = AppName_URL, Asset = AppName_URL, Folder = blank
- Always include the 4 default framework rows: MaxTransactions, Folder_Temp, Folder_Log, LogMessageAddress

**CreateAssets sheet** (columns: Name | ValueType | Value | CreateAsset | Comments)
- Add one row per asset — Name must exactly match the Asset column in the Assets sheet
- Credential assets: ValueType = Credential, Value = (empty), CreateAsset = True
- Text assets with known default: fill Value directly
- Text assets without known value: Value = <<FILL_BEFORE_FIRST_RUN>>
- Always include the 4 default framework rows with their default values
- Always set CreateAsset = True for all rows

The same asset must appear in BOTH sheets — Assets sheet for runtime lookup, CreateAssets sheet for Orchestrator provisioning.

## Step 5 — Pre-flight risk summary (output BEFORE generating any files)
Before writing any file, print a pre-flight summary:
- **Project name derived**: confirm Department_ProcessName and why
- **Transaction type assumption**: queue-based or linear, and why
- **Ambiguities / missing PDD info**: anything that could not be determined from the PDD
- **Credentials / assets that need manual completion**: names identified but values unknown
- **Process steps requiring UI automation**: these become TODO placeholders in XAML
- **Risks**: anything that could cause the generated scaffold to need rework

Then ask the user: "Confirm to proceed, or correct any of the above."
Do not generate files until the user confirms.

## Step 6 — Scaffold project structure
After user confirmation, generate the following folder and file structure.
Check the `MvR_REFramework/` folder in this repository for the current framework file list —
use it as the authoritative source for framework stubs rather than a hardcoded list.

```
ProjectName/
  ├── Main.xaml                          (REF state machine stub — copy from MvR_REFramework/)
  ├── project.json                       (correct name, description from PDD — see GUID note below)
  ├── CHANGELOG.md                       (empty, ready for versioning)
  ├── README.md                          (process summary, app list, queue/asset names)
  ├── Data/
  │   └── Config.xlsx                    (Settings, Constants, Assets, CreateAssets, LogMessageHtml, LogmailAttachment — all pre-populated)
  ├── Documentation/
  │   └── Dictionaries.xlsx              (only if Dictionary variables are identified)
  ├── Framework/
  │   └── (stubs matching MvR_REFramework/ file list)
  └── Processes/
      └── NNN_AppName_Action.xaml        (one stub per process step from PDD)
```

**project.json GUIDs**: generate two fresh random GUIDs — one for `projectId` and one for
`entryPoints[0].uniqueId`. Never copy GUIDs from another project. Never leave them as empty strings.

## Step 6b — Generate Process.xaml (fully scaffolded)
Process.xaml must be generated as completely as possible from the PDD. See `mvr-rpa-standards.json` → `process_xaml_scaffolding` for the full rules. Key points:

- **Root Sequence annotation**: process name, numbered flow steps from PDD, uitvalpad if any. `IsAnnotationDocked: True`.
- **Variables block**: declare all inter-workflow data variables derived from the PDD (names, types, type prefixes).
- **LogMessage** first: `"In: Process - ProjectName"` at Info level.
- **TryCatch** wrapping the main flow:
  - **Try → "Try - Hoofdproces"**: one `ui:InvokeWorkflowFile` per PDD step, in order.
    - `DisplayName` = `"NNN - Description (AppName)"`
    - `WorkflowFileName` = correct relative path to the sub-workflow stub
    - `Arguments` = empty Dictionary stub (wired in Studio)
    - `sap2010:Annotation.AnnotationText` = `"TODO: Wire arguments\n  in_X → localVar\n  out_Y → localVar"` with `IsAnnotationDocked: True`
  - Conditional PDD logic → generate `If` activities with bool_ variable conditions and InvokeWorkflowFile calls in Then/Else branches.
  - Final **LogMessage** at Info level logging key result variables.
  - **Catch `ui:BusinessRuleException`**: LogMessage Warn + InvokeWorkflowFile for error handler (if PDD specifies one, with wiring TODO annotation) + Rethrow.
- Sub-workflow stubs (`NNN_AppName_Action.xaml`) carry the implementation TODOs in their root Sequence annotation — Process.xaml does not.

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
- Annotation text structure: activity purpose → numbered TODO steps derived from the PDD. Use `&#xA;` for newlines and `&quot;` for quotes inside the attribute value.
- See `mvr-rpa-standards.json` → `todo_in_xaml` for the full rules and a XAML example.

## Step 8 — Confirm
After generating all files, print a summary:
- Project name used
- List of generated workflow stubs
- List of Config.xlsx entries pre-populated (all 6 sheets)
- List of assets added to both Assets sheet and CreateAssets sheet
- List of TODOs the developer must complete in UiPath Studio
