# SMOS_OmruilactieWitgoed

**Proces**: Faseren Omruilactie Witgoed
**Afdeling**: SMOS — Gemeente Eindhoven
**Opdrachtgever**: Patrick Adelaars
**Proceseigenaar**: Bente Elast / Pettry Backx

## Doel

Alle werkprocessen omtrent de witgoed actie doorfaseren van fase 10 naar fase 90.
Wanneer een werkproces geen BRP-koppeling heeft, wordt het verplaatst naar de werkbak WGO2 Witgoed Omruilactie uitval BRP.

## Type robot

Onbeheerd (queue-based) | Eenmalige bulkactie

## Applicaties

| Applicatie | Type | Browser | Rechten |
|---|---|---|---|
| Suite | Web | Microsoft Edge | Toegang en bewerkingsrechten WGO werkbak |

## Queue

| Eigenschap | Waarde |
|---|---|
| Queue naam | SMOS_OmruilactieWitgoed |
| Verwacht volume | Eenmalig 13.000 stuks |
| QueueRetry | Nee |
| MaxRetry (framework) | 0 |

## Assets (Orchestrator)

| Asset naam | Type | Omschrijving |
|---|---|---|
| MaxTransactions | Text | ALL |
| Folder_Temp | Text | Data\Temp |
| Folder_Log | Text | Data\Log |
| LogMessageAddress | Text | - (geen logmail adres) |
| Suite_Credential | Credential | Inloggegevens Suite — instellen vóór eerste run |
| Suite_URL | Text | URL Suite-omgeving — instellen vóór eerste run per omgeving |

## Workflow stubs

| Bestand | Doel |
|---|---|
| `Processes/001_Suite_OpenEnInloggen.xaml` | Suite openen en inloggen (Init) |
| `Processes/002_Suite_OpenWerkproces.xaml` | Werkproces openen uit WGO werkbak |
| `Processes/003_Suite_ControleerBRP.xaml` | BRP-koppeling controleren |
| `Processes/004_Suite_FaseerNaarFase40.xaml` | Werkproces doorfaseren naar fase 40 |
| `Processes/005_Suite_FaseerNaarFase90.xaml` | Werkproces doorfaseren naar fase 90 |
| `Processes/006_Suite_VerplaatsNaarWGO2.xaml` | Werkproces verplaatsen naar WGO2 uitval BRP |

## Vóór eerste run

1. Stel `Suite_Credential` in als Credential-asset in Orchestrator (testomgeving en productie)
2. Stel `Suite_URL` in als Text-asset in Orchestrator per omgeving
3. Laad queue-items (`SMOS_OmruilactieWitgoed`) vóór uitvoering
4. Vul selectors in alle stub-workflows in via UiPath Studio
5. Wire alle argumenten in Process.xaml via UiPath Studio (zie annotaties)

## Framework

MvR_REFramework v5 — state machine: Initialization → GetTransactionData → Process → EndProcess
