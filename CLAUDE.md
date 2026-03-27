# UiPath Robot Project — MvR / RPA-Z

## Standards file
All development must follow `mvr-rpa-standards.json` in this folder.
Read it fully before generating or modifying any file.

## Folder structure
```
<repo-root>/
├── Documentation/          ← PDDs go here (PDD_*.pdf / PDD_*.docx)
├── Templates/              ← Word templates
├── Scripts/                ← Helper scripts (e.g. generate_sdd_docx.py)
├── MvR_REFramework/        ← Framework source files
├── CLAUDE.md
├── mvr-rpa-standards.json
└── ProjectName/            ← Automation project folder (may be pre-created)
    ├── Documentation/      ← SDD goes here (SDD_*.md, SDD_*.docx, SDD_*_data.json)
    ├── Data/
    ├── Framework/
    └── Processes/
```

## Process definition
PDDs are always stored in `Documentation/` at the repo root.
The SDD (Solution Design Document) is generated from the PDD by `/solutions` and saved to
`ProjectName/Documentation/`. The SDD is the input for scaffolding — scaffold-robot
reads the SDD, not the PDD.

## Commands
- `/solutions` — generate an SDD from the PDD. Run this first, before scaffolding.
- `/scaffold-robot` — scaffold a new robot from the SDD. Run after `/solutions`.

## Framework
Always use the MvR_REFramework, not the plain UiPath REFramework.
The source framework files are in `MvR_REFramework/` in this repository — use them as the
basis for scaffolded framework stubs; do not maintain a hardcoded list of framework files.
Do not add logic to Main.xaml or SetTransactionStatus.xaml.
Process logic belongs in Process.xaml and invoked sub-workflows.

## XAML generation
Before writing any .xaml file, read `mvr-rpa-standards.json` → `xaml_generation_rules`.
These rules are critical — violations cause UiPath Studio load errors that are hard to diagnose.
