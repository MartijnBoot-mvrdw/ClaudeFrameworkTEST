# UiPath Robot Project — MvR / RPA-Z

## Standards file
All development must follow `mvr-rpa-standards.json` in this folder.
Read it fully before generating or modifying any file.

## Process definition
The process to automate is described in the PDD file in this folder (PDD_*.docx or PDD_*.pdf).

## Commands
- `/scaffold-robot` — scaffold a new robot from a PDD. Run this before writing any files.

## Framework
Always use the MvR_REFramework, not the plain UiPath REFramework.
The source framework files are in `MvR_REFramework/` in this repository — use them as the
basis for scaffolded framework stubs; do not maintain a hardcoded list of framework files.
Do not add logic to Main.xaml or SetTransactionStatus.xaml.
Process logic belongs in Process.xaml and invoked sub-workflows.

## XAML generation
Before writing any .xaml file, read `mvr-rpa-standards.json` → `xaml_generation_rules`.
These rules are critical — violations cause UiPath Studio load errors that are hard to diagnose.
