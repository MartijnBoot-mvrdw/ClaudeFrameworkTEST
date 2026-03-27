# Scaffold MvR REFramework Robot

Follow these steps exactly, in order:

## Step 1 — Read standards
Read `mvr-rpa-standards.json` completely. The canonical rules for XAML generation, naming, config
and error handling are all defined there. Pay special attention to `xaml_generation_rules`
and `sdd_generation` (defines the SDD structure and section-to-scaffold mapping).
Do not proceed without reading it.

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
      └── NNN_AppName_Action.xaml        (one stub per Geautomatiseerd step from SDD 1.2)
```

**project.json GUIDs**: generate two fresh random GUIDs — one for `projectId` and one for
`entryPoints[0].uniqueId`. Never copy GUIDs from another project. Never leave them as empty strings.

## Step 6b — Generate Process.xaml (fully scaffolded)
Process.xaml must be generated as completely as possible from the SDD. See `mvr-rpa-standards.json` → `process_xaml_scaffolding` for the full rules. Key points:

**Init vs Process split (important):**
- Any workflow that **opens an application or logs in** (e.g. `NNN_AppName_OpenEnInloggen.xaml`) is invoked from `Framework/InitAllApplications.xaml` — NOT from Process.xaml. It runs once per robot run.
- Only **per-transaction business logic steps** go in Process.xaml. These are steps that act on a specific queue item.
- When a login/open step is moved to InitAllApplications, add a note to the Process.xaml root annotation: `"NB: [AppName] openen en inloggen ([filename]) wordt uitgevoerd vanuit Framework/InitAllApplications.xaml — niet hier."`

- **Root Sequence annotation**: process name, note about init steps in InitAllApplications (if any), numbered per-transaction steps from SDD, uitvalpad. `IsAnnotationDocked: True`.
- **Variables block**: declare all inter-workflow data variables derived from the SDD (names, types, type prefixes).
- **LogMessage** first: `"In: Process - ProjectName"` at Info level.
- **TryCatch** wrapping the main flow:
  - **Try → "Try - Hoofdproces"**: one `ui:InvokeWorkflowFile` per Geautomatiseerd step from SDD 1.2, in order.
    - `DisplayName` = `"NNN - Description (AppName)"`
    - `WorkflowFileName` = correct relative path to the sub-workflow stub
    - `Arguments` = empty Dictionary stub (wired in Studio)
    - `sap2010:Annotation.AnnotationText` = `"TODO: Wire arguments\n  in_X → localVar\n  out_Y → localVar"` (from SDD 4.N.1) with `IsAnnotationDocked: True`
  - Conditional SDD logic → generate `If` activities with bool_ variable conditions and InvokeWorkflowFile calls in Then/Else branches.
  - Final **LogMessage** at Info level logging key result variables.
  - **Catch `ui:BusinessRuleException`**: LogMessage Warn (BRE trigger text from SDD 2.2) + InvokeWorkflowFile for error handler (if SDD specifies one, with wiring TODO annotation) + Rethrow.
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
- Annotation text structure: activity purpose → numbered TODO steps derived from SDD 4.N.2. Use `&#xA;` for newlines and `&quot;` for quotes inside the attribute value.
- See `mvr-rpa-standards.json` → `todo_in_xaml` for the full rules and a XAML example.

## Step 8 — Confirm
After generating all files, print a summary:
- Project name used
- List of generated workflow stubs
- List of Config.xlsx entries pre-populated (all 6 sheets)
- List of assets added to both Assets sheet and CreateAssets sheet
- List of TODOs the developer must complete in UiPath Studio
