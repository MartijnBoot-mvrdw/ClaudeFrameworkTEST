# Solution Design Document — Faseren Omruilactie Witgoed
**Project**: SMOS_OmruilactieWitgoed
**Afdeling**: SMOS
**Datum**: 2026-03-26

## Versie- en revisietabel

| Versie | Datum | Auteur | Omschrijving |
|---|---|---|---|
| 0.1 | 2026-03-26 | <<AUTEUR>> | Eerste concept op basis van PDD |

## Stakeholders

| Rol | Naam (Klant) | Naam (MvR DW) |
|---|---|---|
| Opdrachtgever | Patrick Adelaars | <<ONBEKEND>> |
| Proceseigenaar | Bente Elast / Pettry Backx | <<ONBEKEND>> |
| Functioneel-applicatiebeheerder | Els Veraa & Angela den Adel | <<ONBEKEND>> |
| Tester | <<ONBEKEND>> | <<ONBEKEND>> |

---

## 1 Overzicht oplossing

### 1.1 Samenvatting

| Eigenschap | Waarde |
|---|---|
| Procesnaam | Faseren Omruilactie Witgoed |
| Projectnaam | SMOS_OmruilactieWitgoed |
| Afdeling | SMOS |
| Beschrijving | Alle werkprocessen omtrent de witgoed actie doorfaseren van fase 10 naar fase 90. Wanneer een werkproces geen BRP-koppeling heeft, wordt het verplaatst naar de werkbak WGO2 Witgoed Omruilactie uitval BRP. |
| Frequentie | Eenmalige bulkactie |
| Verwacht aantal transacties | Eenmalig 13.000 stuks |
| Maximale doorlooptijd | <<ONBEKEND>> |
| Type robot | Onbeheerd (queue-based) |
| Transactietype | Queue-based |
| Dispatcher/Performer | Nee |

### 1.2 Procesflow

| Nr. | Sub-proces | Applicatie | Handmatig / Geautomatiseerd | Voorganger |
|---|---|---|---|---|
| 1 | OpenEnInloggen | Suite | Geautomatiseerd | - |
| 2 | OpenWerkproces | Suite | Geautomatiseerd | 1 |
| 3 | ControleerBRP | Suite | Geautomatiseerd | 2 |
| 4 | FaseerNaarFase40 | Suite | Geautomatiseerd | 3 |
| 5 | FaseerNaarFase90 | Suite | Geautomatiseerd | 4 |
| 6 | VerplaatsNaarWGO2 | Suite | Geautomatiseerd | 3 |

### 1.3 Decompositie Processtappen

De robot opent Suite via Microsoft Edge en logt in. Per transactie-item (werkproces in fase 10, Witgoed Omruilactie) wordt het werkproces geopend. Vervolgens controleert de robot of de cliënt een BRP-koppeling heeft. Indien ja, wordt het werkproces doorgefaseerd naar fase 40 en vervolgens naar fase 90 (afdoening: 10 Toekenning, automatisch beslissen aan). Indien nee, wordt het werkproces verplaatst naar de werkbak WGO2 Witgoed Omruilactie uitval BRP.

1. OpenEnInloggen — Suite openen via Microsoft Edge en inloggen met credentials uit Orchestrator
2. OpenWerkproces — werkproces uit de WGO-werkbak openen
3. ControleerBRP — controleren of cliënt een BRP-koppeling heeft
4. FaseerNaarFase40 — werkproces doorfaseren naar fase 40
5. FaseerNaarFase90 — werkproces doorfaseren naar fase 90 (afdoening: 10 Toekenning, automatisch beslissen)
6. VerplaatsNaarWGO2 — werkproces verplaatsen naar WGO2 Witgoed Omruilactie uitval BRP werkbak

**Uitvalpad**: Wanneer een cliënt geen BRP-koppeling heeft (bekende functionele uitval), volgt de robot het alternatieve pad: het werkproces wordt verplaatst naar de werkbak WGO2 Witgoed Omruilactie uitval BRP en de transactie wordt als BusinessRuleException gemarkeerd.

### 1.4 Benodigde rechten, applicaties en functionaliteiten

| Applicatie | Type | Browser | Rechten benodigd | Opmerkingen |
|---|---|---|---|---|
| Suite | Web | Edge | Toegang en bewerkingsrechten WGO werkbak | Zaaksysteem gemeente Eindhoven; beschikbaar in alle OTAP-omgevingen |

#### 1.4.1 Browser

De gemeente Eindhoven gebruikt Microsoft Edge. De robot navigeert via Edge naar de Suite-omgeving. Er zijn geen speciale downloadinstellingen vereist voor dit proces. De Suite-URL is omgevingsafhankelijk en wordt beheerd via de Orchestrator-asset `Suite_URL`.

---

## 2 Beschrijving technische workflow

### 2.1 Algemeen robotontwerp

De robot maakt gebruik van het MvR_REFramework, een state machine bestaande uit de fasen Initialization, GetTransactionData, Process en EndProcess. In de Initialization-fase worden de assets geladen en wordt Suite geopend en ingelogd. In de GetTransactionData-fase worden werkprocessen één voor één uit de Orchestrator-queue opgehaald. In de Process-fase worden de processtappen uitgevoerd per transactie-item.

| Eigenschap | Waarde |
|---|---|
| Framework | MvR_REFramework |
| Dispatcher aanwezig | Nee |
| Queue naam | SMOS_OmruilactieWitgoed |
| QueueRetry | Nee |
| MaxRetry (framework) | 0 |

### 2.2 Queue en retry mechanisme

Dit is een queue-based robot zonder retry. Elk werkproces in fase 10 (Witgoed Omruilactie) wordt als queue-item aangeboden. Wanneer een item succesvol is verwerkt, wordt het als Successful gemarkeerd. Bij een BusinessRuleException (geen BRP-koppeling — alternatief pad uitgevoerd) wordt het item als Business Exception gemarkeerd. Bij een SystemException (technische fout) wordt het item als Application Exception gemarkeerd. QueueRetry staat uit; framework retry staat op 0.

| Type uitval | Trigger | Gevolg |
|---|---|---|
| BusinessRuleException | Cliënt heeft geen BRP-koppeling; werkproces verplaatst naar WGO2 uitval BRP werkbak | Queue item status = Business Exception |
| SystemException | Technische fout bij benaderen Suite (element niet gevonden, timeout, onverwachte toestand) | Queue item status = Application Exception |

### 2.3 Init-fase

Tijdens de Initialization-fase worden de configuratie-assets geladen via InitAllSettings.xaml. Vervolgens wordt Suite geopend en ingelogd via workflow `001_Suite_OpenEnInloggen.xaml`, waarbij de credential via Get Credential uit de Orchestrator-kluis wordt opgehaald op basis van de asset `Suite_Credential`. De `io_TransactionData` is van het type `QueueItem`.

| Asset naam | Type | Omschrijving | Waarde (indien bekend) |
|---|---|---|---|
| MaxTransactions | Text | Must be integer, or INPUTDIALOG, or ALL (unlimited) | ALL |
| Folder_Temp | Text | Tijdelijke bestandsmap | Data\Temp |
| Folder_Log | Text | Logbestandsmap | Data\Log |
| LogMessageAddress | Text | A single dash will be converted to Nothing by the framework | - |
| Suite_Credential | Credential | Inloggegevens voor Suite (gebruikersnaam + wachtwoord) | *(leeg)* |
| Suite_URL | Text | URL van de Suite-omgeving (omgevingsafhankelijk) | <<FILL_BEFORE_FIRST_RUN>> |

### 2.4 GetTransactionData-fase

Transacties worden opgehaald uit de Orchestrator-queue `SMOS_OmruilactieWitgoed`. Elk queue-item vertegenwoordigt één werkproces (fase 10, Witgoed Omruilactie) dat verwerkt moet worden. De queue-items worden vóór de run aangemaakt (eenmalige bulkactie). De robot haalt queue-items op via GetQueueItem totdat de queue leeg is, waarna het framework de EndProcess-fase ingaat.

### 2.5 Procesfase

De procesfase start met het openen van het werkproces (002_Suite_OpenWerkproces). Vervolgens controleert de robot de BRP-koppeling (003_Suite_ControleerBRP), die een boolean `bool_BRPGekoppeld` teruggeeft. Indien `bool_BRPGekoppeld = True`, worden stap 4 (FaseerNaarFase40) en stap 5 (FaseerNaarFase90) sequentieel uitgevoerd. Indien `bool_BRPGekoppeld = False`, wordt stap 6 (VerplaatsNaarWGO2) uitgevoerd, waarna een BusinessRuleException wordt gegooid zodat het framework het item als Business Exception registreert. De gehele procesfase is omsloten door een TryCatch die BusinessRuleException opvangt, logt en rethrowt naar het framework.

### 2.6 Eindprocesfase

Na afloop van alle transacties verstuurt het framework een logmail via SendLogEmail.xaml naar de geconfigureerde ontvangers (conform PDD: Angela den Adel en Els Veraa bij functionele of technische uitval). Tijdelijke bestanden worden opgeruimd via CleanUpTempFolder.xaml. SetTransactionStatus.xaml markeert het laatste queue-item definitief. Suite wordt gesloten via CloseAllApplications.xaml.

---

## 3 Omgevingsafhankelijkheden

De robot draait binnen het ICT-landschap van Gemeente Eindhoven. Geen externe verbinding is vereist voor Suite zelf; wel is internetverbinding nodig voor UiPath Orchestrator. De Suite-URL verschilt per omgeving (Test/Productie) en dient vóór de eerste run te worden ingesteld als Orchestrator-asset `Suite_URL`. De credential `Suite_Credential` moet eveneens per omgeving worden geconfigureerd in de Orchestrator-kluis.

| Omgeving | Eigenschap | Waarde | Opmerkingen |
|---|---|---|---|
| Test | Suite_URL | <<FILL_BEFORE_FIRST_RUN>> | Testomgeving URL invullen in Orchestrator |
| Productie | Suite_URL | <<FILL_BEFORE_FIRST_RUN>> | Productieomgeving URL invullen in Orchestrator |
| Test | Suite_Credential | *(leeg)* | Testaccount instellen in Orchestrator-kluis |
| Productie | Suite_Credential | *(leeg)* | Productieaccount instellen in Orchestrator-kluis |

---

## 4 Processtappen

### 4.1 Stap 1: OpenEnInloggen (Suite)

#### 4.1.1 Algemene informatie

| Eigenschap | Waarde |
|---|---|
| Workflowbestand | `001_Suite_OpenEnInloggen.xaml` |
| Applicatie | Suite |
| Doel | Suite openen via Microsoft Edge en inloggen met credentials uit de Orchestrator-kluis |
| Navigatie | Robot start vanuit een lege browsersessie en navigeert naar `in_Config("Suite_URL")` |
| Input argumenten | `in_Config (Dictionary<String,Object>)` |
| Output argumenten | *(geen)* |

#### 4.1.2 Omschrijving handelingen

1. Get Credential via `in_Config("Suite_Credential")` — retourneert gebruikersnaam (String) en wachtwoord (SecureString)
2. Open Microsoft Edge en navigeer naar `in_Config("Suite_URL")`
3. TypeInto gebruikersnaamveld met de opgehaalde gebruikersnaam (selector vereist)
4. TypeSecureText wachtwoordveld met het opgehaalde wachtwoord (selector vereist)
5. Click inlogknop (selector vereist)
6. Check App State: wacht tot de Suite-hoofdpagina / werkbak-overzicht volledig geladen is

### 4.2 Stap 2: OpenWerkproces (Suite)

#### 4.2.1 Algemene informatie

| Eigenschap | Waarde |
|---|---|
| Workflowbestand | `002_Suite_OpenWerkproces.xaml` |
| Applicatie | Suite |
| Doel | Werkproces uit de WGO-werkbak openen op basis van het queue-item |
| Navigatie | Robot bevindt zich in Suite na inloggen; navigeert naar de WGO werkbak |
| Input argumenten | `in_Config (Dictionary<String,Object>)`, `in_TransactionItem (QueueItem)` |
| Output argumenten | *(geen)* |

#### 4.2.2 Omschrijving handelingen

1. Lees werkprocesnummer / referentie uit `in_TransactionItem.Reference` of `in_TransactionItem.SpecificContent`
2. Navigeer naar de WGO werkbak in Suite (selector vereist)
3. Zoek het werkproces in de lijst op basis van het werkprocesnummer
4. Click op het werkproces-item om het te openen (selector vereist)
5. Check App State: wacht tot het werkproces volledig geladen is
6. Als een geheimhoudingsmelding verschijnt: sluit de melding via Click OK (Pick Branch of Check App State, selector vereist)

### 4.3 Stap 3: ControleerBRP (Suite)

#### 4.3.1 Algemene informatie

| Eigenschap | Waarde |
|---|---|
| Workflowbestand | `003_Suite_ControleerBRP.xaml` |
| Applicatie | Suite |
| Doel | Controleren of de cliënt een geldige BRP-koppeling heeft in het geopende werkproces |
| Navigatie | Robot bevindt zich in het geopende werkproces; navigeert naar tabblad Taken |
| Input argumenten | `in_Config (Dictionary<String,Object>)`, `in_TransactionItem (QueueItem)` |
| Output argumenten | `out_BRPGekoppeld (Boolean)` |

#### 4.3.2 Omschrijving handelingen

1. Navigeer naar het tabblad Taken in het geopende werkproces (selector vereist)
2. Controleer of de taak "Cliënt koppelen aan BRP" aanwezig is in de takenlijst (Element Exists of Check App State)
3. Als de taak NIET aanwezig is (BRP al gekoppeld): stel `out_BRPGekoppeld = True`
4. Als de taak WEL aanwezig is (BRP ontbreekt): stel `out_BRPGekoppeld = False`
5. Als er een popup verschijnt bij het navigeren: sluit de popup die hoort bij de actieve taak (Check App State + Click OK, selector vereist)

### 4.4 Stap 4: FaseerNaarFase40 (Suite)

#### 4.4.1 Algemene informatie

| Eigenschap | Waarde |
|---|---|
| Workflowbestand | `004_Suite_FaseerNaarFase40.xaml` |
| Applicatie | Suite |
| Doel | Het geopende werkproces doorfaseren van fase 10 naar fase 40 |
| Navigatie | Robot bevindt zich in het geopende werkproces (na ControleerBRP, BRP = Ja) |
| Input argumenten | `in_Config (Dictionary<String,Object>)`, `in_TransactionItem (QueueItem)` |
| Output argumenten | *(geen)* |

#### 4.4.2 Omschrijving handelingen

1. Click op knop "Wijzigen fase" in het werkproces (selector vereist)
2. Check App State: wacht tot het faseringsdialoogvenster geladen is
3. Click op knop "Opslaan en sluiten" (selector vereist)
4. Check App State: wacht tot het werkproces opnieuw geladen is en fase 40 toont

### 4.5 Stap 5: FaseerNaarFase90 (Suite)

#### 4.5.1 Algemene informatie

| Eigenschap | Waarde |
|---|---|
| Workflowbestand | `005_Suite_FaseerNaarFase90.xaml` |
| Applicatie | Suite |
| Doel | Het werkproces doorfaseren van fase 40 naar fase 90 met afdoening "10 Toekenning" en automatisch beslissen aangevinkt |
| Navigatie | Robot bevindt zich in het werkproces in fase 40 (direct na FaseerNaarFase40) |
| Input argumenten | `in_Config (Dictionary<String,Object>)`, `in_TransactionItem (QueueItem)` |
| Output argumenten | *(geen)* |

#### 4.5.2 Omschrijving handelingen

1. Click op knop "Wijzigen fase" in het werkproces (selector vereist)
2. Check App State: wacht tot het faseringsdialoogvenster geladen is
3. Selecteer in het veld "Afdoening" de waarde "10 Toekenning" (SelectItem of Click, selector vereist)
4. Vink de checkbox "Automatisch beslissen" aan (Check activity of Click, selector vereist)
5. Click op knop "Opslaan en sluiten" (selector vereist)
6. Check App State: wacht tot het werkproces verplaatst is naar werkbak ABV (fase 90 bevestigd)

### 4.6 Stap 6: VerplaatsNaarWGO2 (Suite)

#### 4.6.1 Algemene informatie

| Eigenschap | Waarde |
|---|---|
| Workflowbestand | `006_Suite_VerplaatsNaarWGO2.xaml` |
| Applicatie | Suite |
| Doel | Werkproces zonder BRP-koppeling verplaatsen naar werkbak WGO2 Witgoed Omruilactie uitval BRP |
| Navigatie | Robot bevindt zich in het geopende werkproces (na ControleerBRP, BRP = Nee) |
| Input argumenten | `in_Config (Dictionary<String,Object>)`, `in_TransactionItem (QueueItem)` |
| Output argumenten | *(geen)* |

#### 4.6.2 Omschrijving handelingen

1. Als er een popup verschijnt bij de actieve taak: sluit de popup (Check App State + Click OK, selector vereist)
2. Klik op het veld "Medewerker" in het werkproces om dit te wijzigen (selector vereist)
3. Selecteer of typ de waarde "WGO2 Witgoed Omruilactie uitval BRP" in het medewerker-veld (SelectItem of TypeInto + Click, selector vereist)
4. Click op knop "Opslaan en sluiten" (selector vereist)
5. Check App State: wacht tot het werkproces opgeslagen is
6. Throw BusinessRuleException met bericht "Cliënt heeft geen BRP-koppeling — werkproces verplaatst naar WGO2 uitval BRP werkbak."
