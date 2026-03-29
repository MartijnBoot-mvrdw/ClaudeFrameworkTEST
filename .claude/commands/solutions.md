# Generate SDD — MvR REFramework

Generate a Solution Design Document (SDD) in Markdown from a PDD.
PDDs are always stored in `Documentation/` at the repo root.
The SDD is saved to `<ProjectName>/Documentation/` (inside the automation project folder)
and becomes the input for `/scaffold-robot`. Do not scaffold files in this skill — only generate the SDD.

Follow these steps exactly, in order:

## Step 1 — Read standards

Read `mvr-rpa-standards.json` completely. Pay attention to:
- `sdd_generation` — the authoritative definition of every SDD section, its columns,
  derivation rules, and what scaffold-robot consumes from each section.
- `project_naming` — rules for deriving `Department_ProcessName`.
- `configuration` → `config_sheets` → `Assets` / `CreateAssets` — asset naming conventions.

Do not proceed without reading it.

## Step 2 — Locate PDD

Find the PDD file in `Documentation/` matching `Documentation/PDD_*.pdf` or `Documentation/PDD_*.docx`.
- If **no PDD** is found: stop immediately and ask the user to provide or point to the PDD.
  Remind them that PDDs must be placed in the `Documentation/` folder at the repo root.
- If **multiple PDDs** are found: list them and ask which one to use.

## Step 3 — Read PDD and extract

Read the PDD fully. For every section in `mvr-rpa-standards.json` → `sdd_generation.sections`
where `scaffold_robot_consumes` is `true`, the extraction must be **explicit and complete** —
scaffold-robot will not fall back to the PDD.

Extract the following (use `<<ONBEKEND>>` for anything not found):

| What to extract | Where to look in the PDD |
|---|---|
| Procesnaam | Title page or process description header |
| Afdeling | Department reference in title or scope |
| Projectnaam | Derive as `Department_ProcessName` CamelCase |
| Beschrijving | Process scope / doel section |
| Frequentie | Trigger / schedule section |
| Verwacht aantal transacties | Volume / SLA section |
| Maximale doorlooptijd | SLA section |
| Type robot | Attended/Unattended and queue/linear indication |
| Transactietype | Queue-based or Lineair |
| Dispatcher/Performer | Architecture / design section |
| Stakeholders (4 roles) | Stakeholder / betrokkenen table |
| Procesflow steps | Swimlane diagram or step overview (automated vs manual, predecessor) |
| Login vereiste per applicatie | For each unique application: is a login required? What credential asset? Does the robot open the app itself or is it already open? |
| Gedetailleerde stappen per applicatie | Work instructions per step |
| Benodigde rechten + browser | Access / permissions + browser specification |
| Business rules → BRE triggers | Exception / uitval section |
| Credentials (names only, no values) | Credential / access section |
| URLs, environment settings, config values | Config / environment section |
| Queue naam | Queue definition, or derive as Projectnaam |
| QueueRetry required | Exception handling / retry section |
| Test vs Prod differences | Environment section |

## Step 3b — Architecture check and feasibility assessment

Before generating the SDD, run these checks against the extracted PDD data. Results feed
directly into Step 4 pre-flight and SDD Section 2.1.

### Architecture check (Rule P-2)

Answer these questions in order:

1. **Does the PDD describe BOTH (a) reading/scraping a list of items AND (b) processing each
   item individually with per-item status?**
   → YES = recommend **Dispatcher + Performer** (two separate projects). Flag in pre-flight.
   → NO = single performer or sequence (continue to Q2).
2. **Does the process only process queue items that another system creates?**
   → YES = performer-only.
3. **Does the process use Config.xlsx, retry logic, and a structured init/process/end lifecycle?**
   → YES = REFramework single-project (performer pattern, no dispatcher).
4. **Everything else** → simple sequence.

If Dispatcher+Performer is recommended, the SDD must note this in Section 2.1 and Section 2.4.

### Per-step feasibility table

For every `Geautomatiseerd` row in the procesflow, classify the step and flag risks:

| Classification | Signs in PDD | Activity | Risk flags |
|---|---|---|---|
| Web UI — login | "inloggen", "authenticeren", browser app | NApplicationCard, NTypeInto, NClick | MFA/2FA? CAPTCHA? SSO redirect? |
| Web UI — action | "klik", "open", "invul", "navigeer", browser app | NApplicationCard, NClick, NTypeInto | Dynamic selectors — inspection required |
| Web UI — read | "lees", "extraheer", "kopieer", browser app | NGetText, NGetAttribute | Pagination? Dynamic table? |
| SAP | "SAP", "transactiecode", "dynpro" | NSAPLogon, NSAPClickToolbarButton, NSAPReadStatusbar | SAP-specific rules S-1 through S-5 |
| Desktop | "applicatie", "Windows programma", geen browser | NApplicationCard, NClick, NTypeInto | No Playwright — requires PowerShell inspection |
| API / REST | "API", "REST", "webservice", "endpoint" | HttpClient (UiPath.WebAPI.Activities) | Credentials, rate limits, OAuth? |
| Queue / Orchestrator | "queue item", "SetTransactionStatus" | GetQueueItem, SetTransactionStatus | Already addressed by REFramework |
| Data / Excel | "Excel", "CSV", "DataTable", "rapportage" | UseExcelFile, ReadRange, WriteRange | File path in Config? |
| Email | "e-mail", "Outlook", "verstuur bericht" | GetIMAPMailMessages, SendMail | UiPath.Mail.Activities required |
| PDF | "PDF", "rapportage PDF" | ReadPDFText, ReadPDFWithOCR | UiPath.PDF.Activities required |

**Not-automatable risks — flag these in pre-flight:**
- CAPTCHA on login page → mark as `⚠️ CAPTCHA — handmatige ingreep vereist`
- Hardware MFA token → mark as `⚠️ MFA — authenticatie vereist`
- Physical actions (scan, print, physical handling) → mark as `✗ Buiten scope robotisering`
- "Judgment calls" described as human decisions without rules → mark as `⚠️ Beslissingsregel ontbreekt in PDD`

### Architecture rules pre-check (A-1, A-3, A-5)

Verify these against the extracted procesflow BEFORE writing the SDD:

- **A-1**: Every application has an explicit `OpenEnInloggen` step as the FIRST step for that app.
  If any app is missing this → add the step to the procesflow, note it in pre-flight.
- **A-3**: No handelingen step passes credentials as arguments between workflows.
  Credentials are fetched inside the workflow that uses them via `GetRobotCredential`.
  If PDD implies passing username/password → flag in pre-flight and correct in SDD.
- **A-5**: No single workflow step is described in the PDD as doing too many things (e.g.,
  "open app, login, navigate, fill form, submit, extract result" = should be split into
  multiple stubs). Flag over-broad steps for decomposition.

### NuGet package forecast

Based on the automation types identified above, list the NuGet packages that will be needed.
Include this forecast in Section 2.1 of the SDD and pass it to scaffold-robot.

| Package | When needed |
|---|---|
| `UiPath.System.Activities` | Always |
| `UiPath.UIAutomation.Activities` | Any Web UI, Desktop UI, or SAP step |
| `UiPath.Excel.Activities` | Any Excel/data step |
| `UiPath.Mail.Activities` | Any email step |
| `UiPath.PDF.Activities` | Any PDF step |
| `UiPath.WebAPI.Activities` | Any API/REST step |
| `UiPath.Testing.Activities` | Standard REFramework testing |
| `UiPath.Database.Activities` | Any database query step |
| `UiPath.Persistence.Activities` + `UiPath.FormActivityLibrary` | Action Center form tasks |

## Step 4 — Pre-flight check (before writing the SDD)

Print a summary of the extracted data:

- **Projectnaam afgeleid**: `Department_ProcessName` and the reasoning
- **Project folder**: check whether `<ProjectName>/` already exists at the repo root.
  - If it exists: SDD will be written to `<ProjectName>/Documentation/`.
  - If it does **not** exist: the folder and `<ProjectName>/Documentation/` will be created.
- **Architectuur**: Dispatcher+Performer / REFramework single / Sequence — and the P-2 reasoning
- **Applicaties gevonden**: list with automation type (Web UI / Desktop / SAP / API) per app
- **Processtappen** (numbered, with Handmatig/Geautomatiseerd label)
- **Feasibility tabel** — one row per Geautomatiseerd stap:

  | Nr. | Stap | Type | Activiteiten | Risico |
  |---|---|---|---|---|
  | 1 | OpenEnInloggen (AppName) | Web UI login | NApplicationCard, NTypeInto, NClick | Controleer MFA |
  | 2 | ... | ... | ... | ... |

- **Niet-automatiseerbaar**: flag any ⚠️ or ✗ risks found in Step 3b
- **Business rules / BRE triggers** identified
- **Stakeholders** (fill `<<ONBEKEND>>` for any not found in the PDD)
- **Asset lijst**: credentials + config values + URLs found
- **Verwachte NuGet packages**: list from Step 3b forecast
- **Ambiguïteiten / ontbrekende PDD-secties**: anything that will become `<<ONBEKEND>>`

Then ask: **"Bevestig om verder te gaan, of corrigeer bovenstaande."**

Do **not** generate the SDD until the user confirms.

## Step 5a — Generate Markdown SDD

After confirmation, create `<ProjectName>/Documentation/` if it does not already exist,
then generate `<ProjectName>/Documentation/SDD_<ProjectName>.md` in **Dutch**.
Follow the section structure and rules defined in `mvr-rpa-standards.json` → `sdd_generation.sections` exactly.
Use `<<ONBEKEND>>` for any field that could not be determined from the PDD.
This Markdown file is the machine-readable source for `/scaffold-robot`.

**Heading hierarchy — follow exactly:**

| Level | Markdown | Used for |
|---|---|---|
| `#` | `# Solution Design Document — …` | Document title only |
| `##` | `## Versie- en revisietabel` | Intro sections (Versie, Stakeholders) and major sections (1, 2, 3, 4) |
| `###` | `### 1.1 Samenvatting` | Numbered subsections (1.1–1.4, 2.1–2.6, 4.N) |
| `####` | `#### 1.4.1 Browser` | Sub-subsections (1.4.1, 4.N.1, 4.N.2) |

Generate sections in this order:

**Document header**
```
# Solution Design Document — [Procesnaam]
**Project**: [Projectnaam]
**Afdeling**: [Afdeling]
**Datum**: [vandaag]
```

## Versie- en revisietabel
Markdown table: `Versie | Datum | Auteur | Omschrijving`
First row: `0.1 | <vandaag> | <<AUTEUR>> | Eerste concept op basis van PDD`

## Stakeholders
Markdown table: `Rol | Naam (Klant) | Naam (MvR DW)`
Rows: Opdrachtgever, Proceseigenaar, Functioneel-applicatiebeheerder, Tester

---

## 1 Overzicht oplossing

### 1.1 Samenvatting
Key-value table: `Eigenschap | Waarde`
Rows (in order): Procesnaam, Projectnaam, Afdeling, Beschrijving, Frequentie,
Verwacht aantal transacties, Maximale doorlooptijd, Type robot, Transactietype, Dispatcher/Performer

Allowed values for Type robot: `Onbeheerd (queue-based)` / `Onbeheerd (lineair)` / `Attended`
Allowed values for Transactietype: `Queue-based` / `Lineair`
Allowed values for Dispatcher/Performer: `Ja` / `Nee`

### 1.2 Procesflow
Table: `Nr. | Sub-proces | Applicatie | Handmatig / Geautomatiseerd | Voorganger`
- One row per major process step from the PDD.
- `Handmatig / Geautomatiseerd`: use **exactly** `Geautomatiseerd` or `Handmatig` — scaffold-robot filters on this value.
- `Voorganger`: Nr. of the preceding step, or `-` for the first step.
- **LOGIN RULE**: For every unique application in the process, add an explicit `Geautomatiseerd` row for opening/logging in to that application **before** the first functional step for that application. Use `Sub-proces` = `OpenEnInloggen` (opens and logs in) or `Inloggen` (app is already open). This row becomes a workflow stub — never assume login is handled implicitly.

### 1.3 Decompositie Processtappen
Short prose overview of the overall flow, followed by:
- A **numbered list** of all automated steps (Nr. + Sub-proces, matching 1.2).
- A separate bullet: **Uitvalpad**: description of the BRE scenario.

### 1.4 Benodigde rechten, applicaties en functionaliteiten
Table: `Applicatie | Type | Browser | Rechten benodigd | Opmerkingen`

#### 1.4.1 Browser
Specify the required browser and any relevant settings (e.g. automatic download location, popup handling).

---

## 2 Beschrijving technische workflow

### 2.1 Algemeen robotontwerp
Short prose on the MvR REFramework state machine, followed by a key-value table:
`Eigenschap | Waarde`
Rows: Framework (`MvR_REFramework`), Dispatcher aanwezig (`Ja`/`Nee`), Queue naam,
QueueRetry (`Ja`/`Nee`), MaxRetry (framework) (default: `0`)

Also include a **NuGet packages** row listing the packages identified in Step 3b, comma-separated.
Example: `UiPath.System.Activities, UiPath.UIAutomation.Activities, UiPath.Excel.Activities`
This is the authoritative source for scaffold-robot's `resolve_nuget.py` call.

### 2.2 Queue en retry mechanisme
Prose explaining retry strategy. Then a table: `Type uitval | Trigger | Gevolg`
Must include at least one `BusinessRuleException` row and one `SystemException` row.
The Trigger column describes the exact condition that triggers each exception type.

### 2.3 Init-fase
Prose covering: applications opened during Init, credential assets, io_TransactionData type.
Then the **master asset table**: `Asset naam | Type | Omschrijving | Waarde (indien bekend)`

- Always include the 4 default framework rows (MaxTransactions, Folder_Temp, Folder_Log, LogMessageAddress).
- Add one row per Credential from the PDD: Type = `Credential`, Waarde = *(leeg)*.
- Add one row per URL / config value: Type = `Text`, Waarde = known value or `<<FILL_BEFORE_FIRST_RUN>>`.
- Asset naming: `AppName_Purpose` — e.g. `SAP_Credential`, `Suite_URL`.
- This table is the **only** source scaffold-robot uses for Config.xlsx Assets and CreateAssets sheets.

### 2.4 GetTransactionData-fase
Prose: describe whether transactions come from a queue (queue-based) or are built internally (linear).

### 2.5 Procesfase
Prose: describe the main try-catch structure, BRE path, and uitvalscenario.

### 2.6 Eindprocesfase
Prose: describe log mail, cleanup, SetTransactionStatus completion.

---

## 3 Omgevingsafhankelijkheden
Prose + table: `Omgeving | Eigenschap | Waarde | Opmerkingen`
Cover: Test vs Prod differences, browser download settings, DocBot configs, Outlook rules (where applicable).

---

## 4 Processtappen

Generate one sub-section `### 4.N` per row in Section 1.2 where
`Handmatig / Geautomatiseerd` = `Geautomatiseerd`. N matches the Nr. from that row.

### 4.N Stap N: [Sub-proces] ([Applicatie])

#### 4.N.1 Algemene informatie
Key-value table: `Eigenschap | Waarde`

| Eigenschap | Waarde |
|---|---|
| Workflowbestand | `NNN_AppName_Action.xaml` (exact filename matching 1.2 derivation rule) |
| Applicatie | AppName |
| Doel | description of what this step achieves (from PDD) |
| Navigatie | how to reach the starting screen |
| Input argumenten | `in_Config (Dictionary<String,Object>)`, plus any others (always include in_Config; add `in_TransactionItem (QueueItem)` for queue-based robots) |
| Output argumenten | `out_X (Type)`, or *(geen)* if none |

#### 4.N.2 Omschrijving handelingen
Numbered list of step-by-step actions for this process step, derived from the PDD work instructions.
Be specific about which UI elements, fields, buttons or screens are involved.
These become the numbered TODO steps in the UiPath Studio stub annotation AND the `handelingen`
array in the `.spec.json` file that `/build` reads.

**Annotate each step with the expected UiPath activity** using `(activity: X)` at the end of the
line. Use the classification from Step 3b. This lets `/build` generate accurate specs without
re-reading the PDD.

| Step type | Activity annotation |
|---|---|
| Get credential | `(activity: GetRobotCredential)` |
| Open browser | `(activity: NApplicationCard — OpenMode=Always)` |
| Attach to open browser | `(activity: NApplicationCard — OpenMode=Never)` |
| Click button/link | `(activity: NClick)` |
| Type text | `(activity: NTypeInto)` |
| Type password | `(activity: NTypeInto — secure)` |
| Read text from screen | `(activity: NGetText)` |
| Check page loaded | `(activity: NCheckAppState)` |
| Get queue item field | `(activity: Assign — in_TransactionItem.SpecificContent("key").ToString)` |
| Read Excel | `(activity: UseExcelFile + ReadRange)` |
| Send email | `(activity: SendMail)` |
| HTTP API call | `(activity: HttpClient)` |
| Throw business exception | `(activity: Throw BusinessRuleException)` |
| Navigate to URL | `(activity: NClick or NApplicationCard navigate)` |

**For OpenEnInloggen / Inloggen steps**, always include these actions as a baseline:
1. Get Credential via `in_Config("<AppName>_Credential")` (activity: GetRobotCredential)
2. Open [AppName] (browser: navigate to `in_Config("<AppName>_URL")` / desktop: start application) (activity: NApplicationCard — OpenMode=Always)
3. TypeInto gebruikersnaamveld (selector vereist) (activity: NTypeInto)
4. TypeSecureText wachtwoordveld (selector vereist) (activity: NTypeInto — secure)
5. Click inlogknop (selector vereist) (activity: NClick)
6. Check App State: wacht tot hoofdpagina geladen is (activity: NCheckAppState)
Extend or adjust these based on what the PDD says about the login flow for that application.

## Step 5b — Write JSON data file

Write `<ProjectName>/Documentation/SDD_<ProjectName>_data.json` containing all extracted data.
This file is the input for the Python script in Step 5c.

Use exactly this JSON structure (all keys required; use `null` for unknown values):

```json
{
  "procesnaam":                  "...",
  "projectnaam":                 "Department_ProcessName",
  "afdeling":                    "...",
  "datum":                       "YYYY-MM-DD",
  "beschrijving":                "...",
  "frequentie":                  "...",
  "verwacht_aantal_transacties": "...",
  "maximale_doorlooptijd":       "...",
  "type_robot":                  "Onbeheerd (queue-based) | Onbeheerd (lineair) | Attended",
  "transactietype":              "Queue-based | Lineair",
  "dispatcher_performer":        "Ja | Nee",
  "versie_tabel": [
    { "versie": "0.1", "datum": "YYYY-MM-DD", "auteur": "<<AUTEUR>>", "omschrijving": "Eerste concept op basis van PDD" }
  ],
  "stakeholders": [
    { "rol": "Opdrachtgever",                    "naam_klant": "...", "naam_mvr": "<<ONBEKEND>>" },
    { "rol": "Proceseigenaar",                   "naam_klant": "...", "naam_mvr": "<<ONBEKEND>>" },
    { "rol": "Functioneel-applicatiebeheerder",  "naam_klant": "...", "naam_mvr": "<<ONBEKEND>>" },
    { "rol": "Tester",                           "naam_klant": "...", "naam_mvr": "<<ONBEKEND>>" }
  ],
  "procesflow": [
    { "nr": 1, "sub_proces": "OpenEnInloggen", "applicatie": "AppName", "type": "Geautomatiseerd", "voorganger": "-" }
  ],
  "decompositie_prose":  "...",
  "decompositie_stappen": ["1. ...", "2. ..."],
  "uitvalpad":           "...",
  "rechten": [
    { "applicatie": "...", "type": "Web | Desktop", "browser": "Edge | ...", "rechten": "...", "opmerkingen": "..." }
  ],
  "browser_settings":    "...",
  "robotontwerp_prose":  "...",
  "dispatcher_aanwezig": "Ja | Nee",
  "queue_naam":          "Department_ProcessName",
  "queue_retry":         "Ja | Nee",
  "max_retry":           "0",
  "retry_prose":         "...",
  "uitval_tabel": [
    { "type": "BusinessRuleException", "trigger": "...", "gevolg": "Queue item status = Business Exception" },
    { "type": "SystemException",       "trigger": "...", "gevolg": "Retry door framework" }
  ],
  "init_prose":             "...",
  "assets": [
    { "naam": "MaxTransactions", "type": "Text",       "omschrijving": "Must be integer, or INPUTDIALOG, or ALL", "waarde": "ALL" },
    { "naam": "Folder_Temp",     "type": "Text",       "omschrijving": "",                                        "waarde": "Data\\Temp" },
    { "naam": "Folder_Log",      "type": "Text",       "omschrijving": "",                                        "waarde": "Data\\Log" },
    { "naam": "LogMessageAddress","type": "Text",      "omschrijving": "A single dash = Nothing",                 "waarde": "-" }
  ],
  "get_transaction_prose": "...",
  "proces_prose":           "...",
  "eindproces_prose":       "...",
  "omgeving_prose":         "...",
  "omgevingen": [
    { "omgeving": "Test", "eigenschap": "...", "waarde": "...", "opmerkingen": "..." }
  ],
  "processtappen": [
    {
      "nr":               1,
      "sub_proces":       "OpenEnInloggen",
      "applicatie":       "AppName",
      "workflowbestand":  "001_AppName_OpenEnInloggen.xaml",
      "doel":             "...",
      "navigatie":        "...",
      "input_argumenten": "in_Config (Dictionary<String,Object>)",
      "output_argumenten": "(geen)",
      "handelingen":      ["1. Get Credential via in_Config(\"AppName_Credential\")", "2. ..."]
    }
  ]
}
```

Populate every field from the data extracted in Step 3.
The `assets` array must include the 4 default framework rows plus all process-specific assets.
The `processtappen` array must include **every** Geautomatiseerd row from `procesflow`
(including all OpenEnInloggen steps) — one entry per row, in the same order.

## Step 5c — Generate .docx

Run the following command to produce the Word document:

```bash
python Scripts/generate_sdd_docx.py \
  "<ProjectName>/Documentation/SDD_<ProjectName>_data.json" \
  "<ProjectName>/Documentation/SDD_<ProjectName>.docx" \
  "Templates/MvR DW - Solution Design Document - Template.docx"
```

If `python-docx` is not installed, run `pip install python-docx` first.
If the command succeeds, the script prints the output path. If it fails, report the error to the user.

## Step 6 — Confirm

After all three output files are written, print:

- **Gegenereerde bestanden**:
  - `ProjectName/Documentation/SDD_ProjectName.md` — machine-readable, input for `/scaffold-robot`
  - `ProjectName/Documentation/SDD_ProjectName_data.json` — intermediate data file
  - `ProjectName/Documentation/SDD_ProjectName.docx` — Word document for stakeholders
- **Gegenereerde secties**: (list all sections)
- **`<<ONBEKEND>>` placeholders**: list each one with its section id (these must be resolved before scaffolding)
- **Volgende stap**: "Controleer het SDD (.docx voor review, .md voor scaffolding), vul eventuele `<<ONBEKEND>>` velden in beide bestanden in, en voer `/scaffold-robot` uit om het project te genereren."
