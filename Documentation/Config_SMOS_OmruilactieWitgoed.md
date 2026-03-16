# Config.xlsx — SMOS_OmruilactieWitgoed

Overzicht van alle benodigde waarden in `Data/Config.xlsx`.
Vul dit bestand in UiPath Studio in via de Excel-activiteiten of handmatig.

---

## Sheet: Settings

Statische waarden die in het configuratiebestand worden opgeslagen.

| Name | Value | Beschrijving |
|------|-------|--------------|
| `logText_RobotName` | `SMOS_OmruilactieWitgoed` | Naam van de robot voor in log-mails en log messages |
| `logText_ProcessName` | `Doorfaseren Omruilactie Witgoed` | Procesnaam voor rapportage en log-mails |
| `in_QueueName` | `SMOS_OmruilactieWitgoed` | Naam van de Orchestrator-queue |
| `MaxRetryNumber` | `1` | Maximum aantal retries per transactie bij SystemException |
| `ShouldMarkJobAsFaulted` | `False` | Job als faulted markeren bij te veel systeemfouten |
| `logMail_Subject` | `[SMOS_OmruilactieWitgoed] Procesrapportage` | Onderwerp van de log-mails |

---

## Sheet: Assets

Asset-namen die in Orchestrator worden beheerd.
De waarden in deze kolom zijn de **namen** van de assets, niet de inhoud zelf.

| Name | Asset Name | Beschrijving |
|------|------------|--------------|
| `Suite_Credential` | `Suite_Credential` | Login-credentials voor Suite (username + wachtwoord) |
| `Suite_URL` | `Suite_URL` | URL van de Suite-applicatie |
| `SMOS_NotificationEmail` | `SMOS_NotificationEmail` | E-mailadres(sen) voor meldingen bij uitval (Angela den Adel & Els Veraa) |

> **Let op:** Sla alle credentials ook op in **1Password**. De Orchestrator-asset bevat alleen de versleutelde referentie.

---

## Sheet: Constants

Vaste waarden die nooit wijzigen en geen voordeel hebben van Orchestrator-beheer.

| Name | Value | Beschrijving |
|------|-------|--------------|
| `WGO_Werkbak` | `WGO Witgoed Omruilactie` | Naam van de WGO-werkbak in Suite (invoer fase 10) |
| `WGO2_Medewerker` | `WGO2 Witgoed Omruilactie uitval BRP` | Medewerkersnaam voor uitval-werkprocessen (geen BRP) |
| `Fase90_Afdoening` | `10 Toekenning` | Waarde voor het Afdoening-dropdown bij fasering naar fase 90 |
| `BRP_TaakNaam` | `Client koppelen aan BRP` | Taaknaam waarmee BRP-koppeling ontbrekend wordt herkend |

---

## Orchestrator Assets — aanmaken

Maak de volgende assets aan in de Orchestrator-map `[Omgeving] / SMOS / SMOS_OmruilactieWitgoed`:

| Asset Name | Type | Waarde / Instructie |
|------------|------|---------------------|
| `Suite_Credential` | Credential | Username + SecureString wachtwoord. Ook opslaan in 1Password. |
| `Suite_URL` | Text | URL van Suite-omgeving (DEV/TEST/PROD per omgeving apart) |
| `SMOS_NotificationEmail` | Text | `angela.denadel@eindhoven.nl;els.veraa@eindhoven.nl` (puntkomma-gescheiden) |

---

## Orchestrator Queue — aanmaken

| Eigenschap | Waarde |
|------------|--------|
| Queue naam | `SMOS_OmruilactieWitgoed` |
| Map | `[Omgeving] / SMOS` |
| Max retries | `1` |
| Unieke referentie | Zaaknummer werkproces |
| Specific Content sleutels | `ZaakNummer`, `WerkprocesID` (aan te vullen bij queue-vulling) |

---

## Notities

- Maak **per OTAP-omgeving** (DEV / TEST / PROD) aparte Orchestrator-mappen en assets aan.
- De URL-asset (`Suite_URL`) mag afwijken per omgeving; wijzig nooit de code hiervoor.
- Het `CreateAssets.xaml` workflow in de `Framework/CreateAssets/` map kan worden gebruikt om assets in bulk aan te maken vanuit de Config.xlsx (`Create asset`-sheet).
