# SMOS_OmruilactieWitgoed

**Gemeente Eindhoven** | RPA – Functioneel Ontwerp v1.0
**Framework:** MvR REFramework (Windows)
**Type:** Unattended | Queue-based

---

## Procesbeschrijving

Doorfaseren van werkprocessen omtrent de Witgoed Omruilactie van fase 10 t/m fase 90 in Suite (zaaksysteem). Het betreft een eenmalige bulkactie van ca. 13.000 werkprocessen.

### Procesflow

```
Begin → [1] Open werkproces → Koppeling met BRP?
                                      │
                          JA ─────────┤
                          │           │ NEE
                          ▼           ▼
                [2] Fase 40     [4] Verplaats naar
                          │          WGO2 werkbak
                          ▼           │
                [3] Fase 90       Einde (BRE)
                          │
                        Einde
```

### Stappen

| # | Stap | Workflow | Output |
|---|------|----------|--------|
| 1 | Open werkproces uit WGO werkbak | `002_Suite_OpenWerkproces` | Geopend werkproces |
| 2 | Controleer BRP koppeling | `003_Suite_CheckBRPKoppeling` | bool_HasBRPKoppeling |
| 3a | Faseer naar fase 40 | `004_Suite_FaseerNaarFase40` | Werkproces fase 40 |
| 3b | Faseer naar fase 90 | `005_Suite_FaseerNaarFase90` | Werkproces fase 90, werkbak ABV |
| 4 | Verplaats naar WGO2 (uitval) | `006_Suite_VerplaatsNaarWGO2` | Werkproces in WGO2 werkbak |

### Functionele uitval

| Situatie | Actie robot | Rapportage |
|----------|-------------|------------|
| Cliënt heeft geen BRP koppeling | Verplaats naar WGO2 werkbak, gooi BusinessRuleException | Uitvalreden in omschrijving werkproces |
| Technische fout | SystemException → Framework retry | Mail naar Angela den Adel & Els Veraa |

---

## Betrokkenen

| Rol | Naam |
|-----|------|
| Developer | Martijn Boot |
| Opdrachtgever | Patrick Adelaars |
| (Hoofd) procesuitvoerder | Bente Elast / Pettry Backx |
| Functioneel beheerder | Els Veraa & Angela den Adel |

---

## Technische specificaties

| Item | Waarde |
|------|--------|
| Applicatie | Suite (zaaksysteem) |
| Browser | Microsoft Edge |
| Inlogmethode | Orchestrator Credential Asset |
| Verwachte transacties | Eenmalig 13.000 stuks |
| Gem. handmatige duur | 1 min/item |
| Robot type | Unattended |

---

## Projectstructuur

```
SMOS_OmruilactieWitgoed/
├── Main.xaml                          # REFramework state machine (niet aanpassen)
├── Process.xaml                       # Proceslogica per transactie ← HIER WERKEN
├── Sandbox.xaml                       # Testomgeving
├── project.json
│
├── Processes/
│   └── Suite/
│       ├── 001_Suite_Login.xaml           # Login Suite in Edge
│       ├── 002_Suite_OpenWerkproces.xaml  # Open werkproces uit WGO werkbak
│       ├── 003_Suite_CheckBRPKoppeling.xaml # BRP koppeling aanwezig?
│       ├── 004_Suite_FaseerNaarFase40.xaml  # Wijzigen fase → fase 40
│       ├── 005_Suite_FaseerNaarFase90.xaml  # Wijzigen fase → fase 90 + ABV werkbak
│       └── 006_Suite_VerplaatsNaarWGO2.xaml # Uitval: medewerker → WGO2 werkbak
│
├── Framework/
│   ├── InitAllApplications.xaml       # Roep 001_Suite_Login aan hier
│   ├── CloseAllApplications.xaml
│   ├── GetTransactionData.xaml
│   ├── InitAllSettings.xaml
│   ├── SetTransactionStatus.xaml
│   ├── SendLogEmail.xaml
│   └── ...
│
├── Data/
│   ├── Config.xlsx                    # Settings / Assets / Constants
│   └── Log/
│
└── Documentation/
    ├── Config_SMOS_OmruilactieWitgoed.md   # Config waarden overzicht
    ├── Dictionaries.xlsx
    └── REFramework_MvR_Handleiding.docx
```

---

## Config.xlsx — vereiste waarden

Zie [Documentation/Config_SMOS_OmruilactieWitgoed.md](Documentation/Config_SMOS_OmruilactieWitgoed.md) voor het volledige overzicht van alle benodigde Settings, Assets en Constants.

---

## Orchestrator

| Item | Waarde |
|------|--------|
| Queue naam | `SMOS_OmruilactieWitgoed` |
| Credential asset | `Suite_Credential` |
| URL asset | `Suite_URL` |
| Notification email asset | `SMOS_NotificationEmail` |
| Folder structuur | `[Omgeving] / SMOS / SMOS_OmruilactieWitgoed` |

---

## TODO — openstaande acties in UiPath Studio

Alle workflow stubs zijn gereed. De volgende acties moeten nog worden uitgevoerd in UiPath Studio:

### Framework
- [ ] **InitAllApplications.xaml**: Voeg Invoke toe van `001_Suite_Login.xaml`; haal credentials op via Get Credential (`in_Config("Suite_Credential")`)
- [ ] **GetTransactionData.xaml**: Configureer queue item ophalen voor queue `SMOS_OmruilactieWitgoed`

### Process.xaml
- [ ] Wire argument-bindingen van alle InvokeWorkflowFile activiteiten (zie TODO-annotaties in het bestand)

### Workflow stubs — selectors opnemen
- [ ] `001_Suite_Login`: Open Browser (Edge), login-selectors, Check App State startpagina
- [ ] `002_Suite_OpenWerkproces`: Navigatie WGO werkbak, click werkproces, geheimhoudingsmelding
- [ ] `003_Suite_CheckBRPKoppeling`: Element Exists taak 'Client koppelen aan BRP', pop-up sluiten
- [ ] `004_Suite_FaseerNaarFase40`: Click 'Wijzigen fase', click 'Opslaan en sluiten'
- [ ] `005_Suite_FaseerNaarFase90`: Click 'Wijzigen fase', SelectItem '10 Toekenning', Check 'Automatisch beslissen', Opslaan en sluiten
- [ ] `006_Suite_VerplaatsNaarWGO2`: Pop-up sluiten, click tab 'Werkproces', SelectItem medewerker WGO2, Opslaan en sluiten

### Orchestrator configuratie
- [ ] Credential asset `Suite_Credential` aanmaken (username + wachtwoord in 1Password opslaan)
- [ ] Asset `Suite_URL` aanmaken
- [ ] Asset `SMOS_NotificationEmail` aanmaken
- [ ] Queue `SMOS_OmruilactieWitgoed` aanmaken en vullen met 13.000 items
- [ ] Log mail configureren (adressen Angela den Adel & Els Veraa)

---

## Rapportage

- Uitvalreden bij geen BRP koppeling staat in omschrijving van het werkproces
- Bij technische of functionele uitval wordt een mail gestuurd naar Angela den Adel en Els Veraa
- Alle verwerkte items zijn terug te zien in het Orchestrator dashboard (inzage RPA team)
