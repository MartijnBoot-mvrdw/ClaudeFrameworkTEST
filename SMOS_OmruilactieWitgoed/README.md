# SMOS_OmruilactieWitgoed

**Afdeling:** SMOS
**Gemeente:** Eindhoven
**Type:** Unattended — Queue-based
**Opdrachtgever:** Patrick Adelaars
**Developer:** Martijn Boot
**Procesuitvoerder:** Bente Elast / Pettry Backx
**Functioneel beheerder:** Els Veraa & Angela den Adel

## Doel

Automatisch doorfaseren van werkprocessen behorende bij de Witgoed Omruilactie (fase 10 → fase 40 → fase 90) in Suite. Bij ontbrekende BRP-koppeling wordt het werkproces verplaatst naar de WGO 2 werkbak.

## Applicaties

| Applicatie | Type        | Browser      |
|------------|-------------|--------------|
| Suite      | Zaaksysteem | Edge |

## Procesflow

1. Open werkproces uit WGO werkbak (sluit geheimhoudingsmelding indien aanwezig)
2. Controleer BRP-koppeling via taken
   - **Ja BRP** → Faseer naar fase 40 → Faseer naar fase 90 → Klaar
   - **Nee BRP** → Verplaats medewerker naar "WGO2 Witgoed Omruilactie uitval BRP" → BusinessRuleException

## Queue

| Eigenschap       | Waarde                       |
|-----------------|------------------------------|
| Queue naam       | SMOS_OmruilactieWitgoed      |
| Verwacht volume  | ~13.000 items (eenmalige run)|
| QueueRetry       | False                        |

## Orchestrator Assets

| Asset naam          | Type       | Omschrijving                        |
|--------------------|------------|-------------------------------------|
| Suite_Credential   | Credential | Inloggegevens Suite                 |
| Suite_URL          | Text       | URL van Suite omgeving              |
| MaxTransactions    | Text       | Aantal te verwerken items (ALL)     |
| Folder_Temp        | Text       | Pad tijdelijke bestanden            |
| Folder_Log         | Text       | Pad logbestanden                    |
| LogMessageAddress  | Text       | E-mailadres logmail (- = geen mail) |

## Uitval (BusinessRuleException)

- **Geen BRP koppeling**: werkproces wordt verplaatst naar WGO2 werkbak; item gemarkeerd als Business Exception in Orchestrator.

## Workflows

| Bestand                            | Omschrijving                                      |
|------------------------------------|---------------------------------------------------|
| `Main.xaml`                        | REFramework state machine (MvR framework)         |
| `Process.xaml`                     | Hoofdorkestratie per transactie                   |
| `Processes/001_Suite_OpenWerkproces.xaml`   | Werkproces openen, geheimhouding sluiten |
| `Processes/002_Suite_ControleerBRP.xaml`    | BRP-koppeling controleren via taken      |
| `Processes/003_Suite_FaseerNaarFase40.xaml` | Doorfaseren naar fase 40                 |
| `Processes/004_Suite_FaseerNaarFase90.xaml` | Doorfaseren naar fase 90 (afd. ABV)      |
| `Processes/005_Suite_VerplaatsNaarWGO2.xaml`| Medewerker wijzigen naar WGO2 uitval BRP |

## Rapportage

- Uitvalreden in omschrijving werkproces in Suite
- Logmail bij technische/functionele uitval naar Angela den Adel en Els Veraa
- Alle transactie-items zichtbaar in Orchestrator dashboard
