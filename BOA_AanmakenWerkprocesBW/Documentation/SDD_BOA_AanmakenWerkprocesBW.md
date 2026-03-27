# Solution Design Document — Aanmaken Werkproces BW
**Project**: BOA_AanmakenWerkprocesBW
**Afdeling**: BOA
**Datum**: 2026-03-27

## Versie- en revisietabel

| Versie | Datum | Auteur | Omschrijving |
|---|---|---|---|
| 0.1 | 2026-03-27 | <<AUTEUR>> | Eerste concept op basis van PDD |

## Stakeholders

| Rol | Naam (Klant) | Naam (MvR DW) |
|---|---|---|
| Opdrachtgever | Martine Geerts | <<ONBEKEND>> |
| Proceseigenaar | Mark Blokland, Marieke van Breugel | <<ONBEKEND>> |
| Functioneel-applicatiebeheerder | Jacques van de Wal, Jerry Timmermans | <<ONBEKEND>> |
| Tester | <<ONBEKEND>> | <<ONBEKEND>> |

---

## 1 Overzicht oplossing

### 1.1 Samenvatting

| Eigenschap | Waarde |
|---|---|
| Procesnaam | Aanmaken Werkproces BW |
| Projectnaam | BOA_AanmakenWerkprocesBW |
| Afdeling | BOA |
| Beschrijving | De robot verwerkt binnengekomen aanvragen voor Beschermd Wonen die via het webformulier zijn ingediend. Per aanvraag wordt het aanvraagformulier opgehaald uit Outlook, wordt een werkproces aangemaakt in Suite (inclusief aanmaken of koppelen van de client aan BRP), en wordt het aanvraagformulier opgeslagen in DIS. Bij functionele uitval wordt de aanvraag doorgestuurd naar de uitval-mailbox. |
| Frequentie | Dagelijks |
| Verwacht aantal transacties | 18 items per dag |
| Maximale doorlooptijd | <<ONBEKEND>> |
| Type robot | Onbeheerd (queue-based) |
| Transactietype | Queue-based |
| Dispatcher/Performer | Nee |

### 1.2 Procesflow

| Nr. | Sub-proces | Applicatie | Handmatig / Geautomatiseerd | Voorganger |
|---|---|---|---|---|
| 1 | OpenEnInloggen | Outlook | Geautomatiseerd | - |
| 2 | AanvraagFormulierOphalen | Outlook | Geautomatiseerd | 1 |
| 3 | OpenEnInloggen | Suite | Geautomatiseerd | 2 |
| 4 | ClientControle | Suite | Geautomatiseerd | 3 |
| 5 | ClientAanmaken | Suite | Geautomatiseerd | 4 |
| 6 | BrpKoppelingControle | Suite | Geautomatiseerd | 4 |
| 7 | WerkprocesControle | Suite | Geautomatiseerd | 5 |
| 8 | WerkprocesAanmaken | Suite | Geautomatiseerd | 7 |
| 9 | ToelichtingInWerkproces | Suite | Geautomatiseerd | 8 |
| 10 | OpenEnInloggen | DIS | Geautomatiseerd | 3 |
| 11 | AanvraagFormulierOpslaan | DIS | Geautomatiseerd | 10 |
| 12 | Uitval | Outlook | Geautomatiseerd | 1 |

> Stap 5 (ClientAanmaken) en stap 6 (BrpKoppelingControle) zijn wederzijds uitsluitend: stap 5 wordt uitgevoerd als de client niet bestaat, stap 6 als de client al bestaat. Stap 8 (WerkprocesAanmaken) wordt alleen uitgevoerd als er nog geen werkproces bestaat of als de aanvraag een herindicatie betreft. Stap 9 (ToelichtingInWerkproces) wordt alleen uitgevoerd als er een opmerking in de aanvraag is gevonden. Stap 12 (Uitval) wordt alleen uitgevoerd bij functionele uitval (BRE).

### 1.3 Decompositie Processtappen

De robot wordt dagelijks geactiveerd en verwerkt alle ongelezen aanvraagmails uit de mailbox GM_SD_startzorg – BW aanvragen Regio. Per mail wordt het aanvraagformulier (PDF) geopend en worden de benodigde gegevens (BSN, DatumBinnenkomst, Indicatie, Opmerking) uitgelezen. Vervolgens wordt ingelogd in Suite, wordt de client opgezocht en indien nodig aangemaakt en aan BRP gekoppeld. Daarna wordt het werkproces gecontroleerd en indien nodig aangemaakt. Optioneel wordt een toelichting toegevoegd bij aanwezigheid van een opmerking. Tot slot wordt het aanvraagformulier opgeslagen in DIS. Bij functionele uitval (BSN niet gevonden, opmerking in aanvraag) wordt de aanvraag doorgestuurd naar de uitval-mailbox.

1. OpenEnInloggen (Outlook) — verbinding maken met de gedeelde mailbox
2. AanvraagFormulierOphalen (Outlook) — ophalen van meest recente aanvraagmail en uitlezen van PDF-bijlage
3. OpenEnInloggen (Suite) — inloggen in Suite zaaksysteem
4. ClientControle (Suite) — zoeken op BSN of client al bekend is in Suite
5. ClientAanmaken (Suite) — aanmaken van nieuwe client via BRP-koppeling (als client niet bestaat)
6. BrpKoppelingControle (Suite) — controleren en instellen van BRP-koppeling (als client al bestaat)
7. WerkprocesControle (Suite) — controleren of er al een werkproces WMO VF/VB bestaat
8. WerkprocesAanmaken (Suite) — aanmaken werkproces met regeling WMO, groep op basis van Indicatie
9. ToelichtingInWerkproces (Suite) — toevoegen toelichting bij werkproces indien opmerking aanwezig
10. OpenEnInloggen (DIS) — openen van DIS documentmanagementsysteem
11. AanvraagFormulierOpslaan (DIS) — opslaan van aanvraagformulier en bijlagen in DIS onder werkprocesnummer
12. Uitval (Outlook) — versturen van uitvalmail naar GM_SD_startzorg – BW aanvragen Regio Uitval

- **Uitvalpad**: BusinessRuleException wanneer het BSN niet gevonden wordt in de BRP bij het aanmaken van een nieuwe client (stap 5), of wanneer er een opmerking in de aanvraag is gevonden (stap 2). De robot stuurt de aanvraag met bijlagen en reden van uitval naar de uitval-mailbox.

### 1.4 Benodigde rechten, applicaties en functionaliteiten

| Applicatie | Type | Browser | Rechten benodigd | Opmerkingen |
|---|---|---|---|---|
| Outlook | Desktop / Exchange | N.v.t. | Toegang en bewerkingsrechten op gedeelde mailbox GM_SD_startzorg – BW aanvragen Regio en GM_SD_startzorg – BW aanvragen Regio Uitval | Gedeelde mailbox; credentials via Orchestrator |
| Suite | Web | Microsoft Edge | Toegang en bewerkingsrechten (raadplegen, werkvoorraad ADBW, aanmaken client, aanmaken werkproces) | Zaaksysteem; werkbakcode ADBW |
| DIS | Desktop | N.v.t. | Toegang tot werkbak WMO/JW Begeleiding & Verblijf Admin; opslaan van documenten | Documentmanagementsysteem; inlogmethode onbekend (vermoedelijk SSO) |

#### 1.4.1 Browser

De gemeente Eindhoven gebruikt **Microsoft Edge** voor webtoepassingen. Suite is een webapplicatie en dient te worden benaderd via Microsoft Edge. Voor Suite dient de downloadlocatie zo geconfigureerd te worden dat tijdelijke bestanden in de `Folder_Temp`-map worden opgeslagen. Pop-ups voor bevestigingsdialogen dienen toegestaan te zijn.

---

## 2 Beschrijving technische workflow

### 2.1 Algemeen robotontwerp

De robot is gebouwd op het MvR_REFramework, een state machine bestaande uit de fasen Initialization, GetTransactionData, Process en EndProcess. In de Initialization-fase worden applicaties geopend (Outlook, Suite, DIS) en wordt de configuratie geladen. In GetTransactionData wordt het volgende te verwerken queue-item opgehaald. In de Process-fase wordt de businesslogica uitgevoerd. In EndProcess worden logmails verstuurd en wordt de robot afgesloten.

| Eigenschap | Waarde |
|---|---|
| Framework | MvR_REFramework |
| Dispatcher aanwezig | Nee |
| Queue naam | BOA_AanmakenWerkprocesBW |
| QueueRetry | Nee |
| MaxRetry (framework) | 0 |

### 2.2 Queue en retry mechanisme

De queue `BOA_AanmakenWerkprocesBW` bevat één item per aanvraagmail. Queue-items worden aangemaakt vanuit de GetTransactionData-fase op basis van ongelezen mails in de mailbox. Er wordt geen gebruik gemaakt van Orchestrator QueueRetry. BusinessRuleExceptions worden niet herhaald — de aanvraag wordt bij uitval doorgestuurd naar de uitval-mailbox voor handmatige verwerking.

| Type uitval | Trigger | Gevolg |
|---|---|---|
| BusinessRuleException | BSN niet gevonden in BRP bij aanmaken client (stap 5), of opmerking aanwezig in aanvraag (stap 2) | Queue item status = Business Exception; uitvalmail verstuurd naar GM_SD_startzorg – BW aanvragen Regio Uitval |
| SystemException | Onverwachte applicatiefout (bijv. Suite niet bereikbaar, selector niet gevonden, time-out) | Retry door framework (MaxRetry = 0, dus geen herhalingen); queue item status = System Exception |

### 2.3 Init-fase

Tijdens de Initialization-fase worden de volgende applicaties geopend en ingelogd:
- **Outlook**: verbinding via Exchange/Outlook activiteiten met behulp van `Outlook_Credential`.
- **Suite**: browser geopend en ingelogd via `Suite_URL` en `Suite_Credential`.
- **DIS**: desktopapplicatie geopend (vermoedelijk SSO via robotaccount).

Het argument `io_TransactionData` is van het type `QueueItem`. De config-dictionary wordt gevuld vanuit de Assets in Config.xlsx.

| Asset naam | Type | Omschrijving | Waarde (indien bekend) |
|---|---|---|---|
| MaxTransactions | Text | Must be integer, or INPUTDIALOG, or ALL | ALL |
| Folder_Temp | Text | Tijdelijke opslagmap voor aanvraagformulieren en bijlagen | Data\Temp |
| Folder_Log | Text | Logmap | Data\Log |
| LogMessageAddress | Text | A single dash = Nothing | - |
| Outlook_Credential | Credential | Inloggegevens voor gedeelde Outlook-mailbox GM_SD_startzorg | *(leeg)* |
| Suite_Credential | Credential | Inloggegevens voor Suite zaaksysteem | *(leeg)* |
| Suite_URL | Text | URL van Suite zaaksysteem | <<FILL_BEFORE_FIRST_RUN>> |
| Outlook_Mailbox | Text | Naam/adres van de inkomende gedeelde mailbox | GM_SD_startzorg – BW aanvragen Regio |
| Outlook_UitvalMailbox | Text | Naam/adres van de uitval-mailbox | GM_SD_startzorg – BW aanvragen Regio Uitval |

### 2.4 GetTransactionData-fase

Transacties zijn queue-based. De GetTransactionData-fase haalt het volgende item op uit de Orchestrator-queue `BOA_AanmakenWerkprocesBW`. De robot vult deze queue zelf op basis van ongelezen mails in de inkomende mailbox (geen aparte dispatcher). Wanneer er geen items meer zijn, wordt `io_TransactionItem = Nothing` teruggegeven en stopt de robot.

### 2.5 Procesfase

In Process.xaml wordt per transactie-item de businesslogica uitgevoerd. De hoofdstroom is omvat in een TryCatch. In het Try-blok worden achtereenvolgens de stappen AanvraagFormulierOphalen, ClientControle, ClientAanmaken of BrpKoppelingControle (afhankelijk van of de client bestaat), WerkprocesControle, WerkprocesAanmaken (conditioneel), ToelichtingInWerkproces (conditioneel) en AanvraagFormulierOpslaan uitgevoerd. In het Catch-blok (BusinessRuleException) wordt de uitvalstap aangeroepen die de aanvraag met reden doorstuurt naar de uitval-mailbox.

### 2.6 Eindprocesfase

Na verwerking van alle transacties verstuurt de robot een afsluitende logmail aan de procesuitvoerder met daarin een overzicht van alle verwerkte items en eventuele uitzonderingen. Tijdelijke bestanden in `Folder_Temp` worden opgeruimd. SetTransactionStatus rondt elk queue-item af met de juiste status (Successful, BusinessException of SystemException).

---

## 3 Omgevingsafhankelijkheden

De robot draait in het ICT-landschap van Gemeente Eindhoven en heeft geen externe internetverbinding nodig buiten de verbinding met UiPath Orchestrator. De applicaties Suite, DIS en Outlook zijn beschikbaar binnen het gemeentelijke netwerk.

| Omgeving | Eigenschap | Waarde | Opmerkingen |
|---|---|---|---|
| Test | Suite_URL | <<FILL_BEFORE_FIRST_RUN>> | Test-URL van Suite invullen voor eerste run |
| Productie | Suite_URL | <<FILL_BEFORE_FIRST_RUN>> | Productie-URL van Suite invullen voor eerste run |
| Test | Outlook_Mailbox | <<FILL_BEFORE_FIRST_RUN>> | Test-mailbox gebruiken tijdens testfase |
| Productie | Outlook_Mailbox | GM_SD_startzorg – BW aanvragen Regio | Productie gedeelde mailbox |
| Test | Outlook_UitvalMailbox | <<FILL_BEFORE_FIRST_RUN>> | Test-uitvalmailbox gebruiken tijdens testfase |
| Productie | Outlook_UitvalMailbox | GM_SD_startzorg – BW aanvragen Regio Uitval | Productie uitval-mailbox |
| Alle | DIS werkbak | WMO/JW Begeleiding & Verblijf Admin | Vaste werkbak in DIS |
| Alle | Suite werkbakcode | ADBW | Vaste werkbakcode voor assistent in Suite |

---

## 4 Processtappen

### 4.1 Stap 1: OpenEnInloggen (Outlook)

#### 4.1.1 Algemene informatie

| Eigenschap | Waarde |
|---|---|
| Workflowbestand | `001_Outlook_OpenEnInloggen.xaml` |
| Applicatie | Outlook |
| Doel | Verbinding maken met de gedeelde Outlook-mailbox GM_SD_startzorg – BW aanvragen Regio via Exchange/Outlook-activiteiten |
| Navigatie | Start vanuit de Init-fase; Outlook dient bereikbaar te zijn op het robotaccount |
| Input argumenten | `in_Config (Dictionary<String,Object>)` |
| Output argumenten | *(geen)* |

#### 4.1.2 Omschrijving handelingen

1. Get Credential via `in_Config("Outlook_Credential")` — ophalen gebruikersnaam en wachtwoord
2. Verbinding maken met de gedeelde mailbox via Exchange-/Outlook-activiteiten (gebruik `in_Config("Outlook_Mailbox")` als mailboxadres)
3. Check App State: controleer of verbinding succesvol is tot stand gekomen
4. TypeInto gebruikersnaamveld indien aanmeldscherm verschijnt (selector vereist)
5. TypeSecureText wachtwoordveld indien aanmeldscherm verschijnt (selector vereist)
6. Click inlogknop indien aanmeldscherm verschijnt (selector vereist)

### 4.2 Stap 2: AanvraagFormulierOphalen (Outlook)

#### 4.2.1 Algemene informatie

| Eigenschap | Waarde |
|---|---|
| Workflowbestand | `002_Outlook_AanvraagFormulierOphalen.xaml` |
| Applicatie | Outlook |
| Doel | Ophalen van de meest recente ongelezen aanvraagmail met subject 'Ingezonden formulier van: Indicatie voor Beschermd Wonen aanvragen of wijzigen *jaartal*', uitlezen van PDF-bijlage, extraheren van BSN, DatumBinnenkomst, Indicatie en Opmerking |
| Navigatie | Gebruik bestaande mailboxverbinding uit Init-fase; navigeer naar inkomende mailbox |
| Input argumenten | `in_Config (Dictionary<String,Object>)`, `in_TransactionItem (QueueItem)` |
| Output argumenten | `out_BSN (String)`, `out_DatumBinnenkomst (String)`, `out_Indicatie (String)`, `out_Opmerking (String)`, `out_AanvraagPad (String)`, `out_BijlagenPaden (List<String>)` |

#### 4.2.2 Omschrijving handelingen

1. Haal de meest recente ongelezen mail op met subject-filter 'Ingezonden formulier van: Indicatie voor Beschermd Wonen aanvragen of wijzigen'
2. Sla de PDF-bijlage op in `in_Config("Folder_Temp")` — sla pad op in variabele `out_AanvraagPad`
3. Sla eventuele overige bijlagen op in `in_Config("Folder_Temp")` — sla paden op in `out_BijlagenPaden`
4. Open het opgeslagen PDF-bestand en lees de inhoud uit
5. Extraheer het Burgerservicenummer (BSN) uit het veld 'Burgerservicenummer' — sla op in `out_BSN`
6. Extraheer de DatumBinnenkomst (ontvangstdatum van de mail) — sla op in `out_DatumBinnenkomst`
7. Extraheer de Indicatie ('15BW1 Beschermd Wonen' of 'Begeleid Wonen') en bepaal of het gaat om Verblijf of Begeleiding — sla op in `out_Indicatie`
8. Controleer of er een opmerking in het aanvraagformulier staat — sla op in `out_Opmerking` (leeg als geen opmerking)
9. Als er een opmerking aanwezig is: gooi `New BusinessRuleException("Opmerking aanwezig in aanvraag. Handmatige controle vereist.")` om uitvalpad te activeren

### 4.3 Stap 3: OpenEnInloggen (Suite)

#### 4.3.1 Algemene informatie

| Eigenschap | Waarde |
|---|---|
| Workflowbestand | `003_Suite_OpenEnInloggen.xaml` |
| Applicatie | Suite |
| Doel | Openen van Suite in Microsoft Edge en inloggen met robotaccount-credentials |
| Navigatie | Navigeer naar `in_Config("Suite_URL")` in Microsoft Edge |
| Input argumenten | `in_Config (Dictionary<String,Object>)` |
| Output argumenten | *(geen)* |

#### 4.3.2 Omschrijving handelingen

1. Get Credential via `in_Config("Suite_Credential")` — ophalen gebruikersnaam en SecureString wachtwoord
2. Open Microsoft Edge en navigeer naar `in_Config("Suite_URL")`
3. TypeInto gebruikersnaamveld — veld 'Gebruiker' (selector vereist)
4. TypeSecureText wachtwoordveld — veld 'Wachtwoord' (selector vereist)
5. Click inlogknop — knop 'Log In' (selector vereist)
6. Check App State: wacht tot hoofdpagina Suite geladen is (selector vereist)

### 4.4 Stap 4: ClientControle (Suite)

#### 4.4.1 Algemene informatie

| Eigenschap | Waarde |
|---|---|
| Workflowbestand | `004_Suite_ClientControle.xaml` |
| Applicatie | Suite |
| Doel | Controleren of de client al bekend is in Suite op basis van BSN; ophalen van de clientcode indien de client bestaat |
| Navigatie | Vanuit de Suite-hoofdpagina; klik op 'Raadplegen' in het menu |
| Input argumenten | `in_Config (Dictionary<String,Object>)`, `in_TransactionItem (QueueItem)`, `in_BSN (String)` |
| Output argumenten | `out_ClientBekend (Boolean)`, `out_ClientCode (String)` |

#### 4.4.2 Omschrijving handelingen

1. Klik op menuoptie 'Raadplegen' (selector vereist)
2. Vul het BSN in het zoekveld 'BSN' in via TypeInto (selector vereist)
3. Voer de zoekopdracht uit (Enter of zoekknop)
4. Check App State: controleer of zoekresultaten zijn geladen
5. Controleer zoekresultatentabel — is er een rij aanwezig met de client?
6. Als client niet gevonden: stel `out_ClientBekend = False`, `out_ClientCode = ""`; ga terug naar aanroeper
7. Als client gevonden: haal de clientcode op uit de eerste rij van de zoekresultaten — sla op in `out_ClientCode`; stel `out_ClientBekend = True`

### 4.5 Stap 5: ClientAanmaken (Suite)

#### 4.5.1 Algemene informatie

| Eigenschap | Waarde |
|---|---|
| Workflowbestand | `005_Suite_ClientAanmaken.xaml` |
| Applicatie | Suite |
| Doel | Aanmaken van een nieuwe client in Suite via BRP-koppeling wanneer de client nog niet bestaat |
| Navigatie | Vanuit Suite-hoofdpagina; navigeer naar Werkvoorraad → werkbakcode ADBW |
| Input argumenten | `in_Config (Dictionary<String,Object>)`, `in_TransactionItem (QueueItem)`, `in_BSN (String)` |
| Output argumenten | `out_ClientCode (String)` |

#### 4.5.2 Omschrijving handelingen

1. Klik op menuoptie 'Werkvoorraad' (selector vereist)
2. Vul werkbakcode 'ADBW' in het code-veld en druk op Enter (selector vereist)
3. Klik op aanmaakknop ('+' icoon) (selector vereist)
4. Klik op 'Zoek client' knop (selector vereist)
5. Vul het BSN in het BSN-zoekveld (selector vereist)
6. Check App State: controleer zoekresultaat BRP
7. Als geen resultaat gevonden bij BSN: gooi `New BusinessRuleException("BSN niet gevonden in BRP: " + in_BSN)` — robot gaat naar uitvalpad
8. Klik op 'BRP cliënt' knop (selector vereist)
9. Kies bij gemeente '772 Eindhoven' in de dropdown (selector vereist)
10. Haal de Clientcode op uit het scherm — sla op in `out_ClientCode`
11. Klik op 'Accepteren' knop (selector vereist)
12. Klik op 'Zoeken' knop (selector vereist)
13. Klik op 'Opslaan en sluiten' knop (selector vereist)
14. Navigeer terug naar werkvoorraad ADBW

### 4.6 Stap 6: BrpKoppelingControle (Suite)

#### 4.6.1 Algemene informatie

| Eigenschap | Waarde |
|---|---|
| Workflowbestand | `006_Suite_BrpKoppelingControle.xaml` |
| Applicatie | Suite |
| Doel | Controleren of de bestaande client gekoppeld is aan de BRP en indien niet, de koppeling alsnog instellen |
| Navigatie | Vanuit het geopende raadpleegscherm van de gevonden client |
| Input argumenten | `in_Config (Dictionary<String,Object>)`, `in_TransactionItem (QueueItem)`, `in_ClientCode (String)` |
| Output argumenten | *(geen)* |

#### 4.6.2 Omschrijving handelingen

1. Lees veld 'Status BRP' uit in het raadpleegscherm van de client (selector vereist)
2. Als Status BRP = 'Ja / Status: Gekoppeld': navigeer naar werkvoorraad ADBW en stop workflow (koppeling al aanwezig)
3. Als client nog niet gekoppeld: klik op 'Raadplegen dossier' (selector vereist)
4. Klik op 'BRP cliënt' knop (selector vereist)
5. Klik op 'Ja' in het bevestigingsdialoog (selector vereist)
6. Klik op 'Accepteren' knop (selector vereist)
7. Klik op 'Opslaan en sluiten' knop (selector vereist)
8. Navigeer naar werkvoorraad ADBW

### 4.7 Stap 7: WerkprocesControle (Suite)

#### 4.7.1 Algemene informatie

| Eigenschap | Waarde |
|---|---|
| Workflowbestand | `007_Suite_WerkprocesControle.xaml` |
| Applicatie | Suite |
| Doel | Controleren of er al een werkproces bestaat voor de client in de regeling WMO, groep Verblijf (VF) of Begeleiding (VB); bepalen of een nieuw werkproces aangemaakt moet worden |
| Navigatie | Vanuit werkvoorraad ADBW; klik op 'Toevoegen', voer clientcode in, klik op 'Werkprocessen aanwezig' |
| Input argumenten | `in_Config (Dictionary<String,Object>)`, `in_TransactionItem (QueueItem)`, `in_ClientCode (String)`, `in_Indicatie (String)` |
| Output argumenten | `out_WerkprocesBestaatAl (Boolean)`, `out_WerkprocesNummer (String)` |

#### 4.7.2 Omschrijving handelingen

1. Klik op 'Toevoegen' (+) knop in werkvoorraad ADBW (selector vereist)
2. Voer de clientcode in het 'Cliënt' zoekveld in (selector vereist)
3. Klik op knop 'Werkprocessen aanwezig' (selector vereist)
4. Controleer het overzicht op aanwezigheid van een werkproces met Regeling WMO en Groep Verblijf (VF) of Begeleiding (VB)
5. Als er al een werkproces bestaat: haal het werkprocesnummer op — sla op in `out_WerkprocesNummer`; stel `out_WerkprocesBestaatAl = True`; navigeer naar Suite-startpagina
6. Als er nog geen werkproces bestaat: stel `out_WerkprocesBestaatAl = False`, `out_WerkprocesNummer = ""`

### 4.8 Stap 8: WerkprocesAanmaken (Suite)

#### 4.8.1 Algemene informatie

| Eigenschap | Waarde |
|---|---|
| Workflowbestand | `008_Suite_WerkprocesAanmaken.xaml` |
| Applicatie | Suite |
| Doel | Aanmaken van een nieuw werkproces in Suite met de juiste regeling, groep en overige instellingen op basis van de Indicatie uit het aanvraagformulier |
| Navigatie | Vanuit het werkproces-aanmaakscherm in werkvoorraad ADBW |
| Input argumenten | `in_Config (Dictionary<String,Object>)`, `in_TransactionItem (QueueItem)`, `in_ClientCode (String)`, `in_Indicatie (String)` |
| Output argumenten | `out_WerkprocesNummer (String)` |

#### 4.8.2 Omschrijving handelingen

1. Zet veld 'Regeling' op '11 Wmo' (selector vereist)
2. Kies bij 'Groep' op basis van `in_Indicatie`: als Verblijf → 'Af Aanvraag verblijf'; als Begeleiding → 'Verkort Begeleiding' (selector vereist)
3. Zet veld 'Aard bijstand' op 'Incidenteel' (selector vereist)
4. Zet veld 'Aard verzoek' op 'VG Verzoek generalist' (selector vereist)
5. Zet veld 'Urgentie' op 'Normaal' (selector vereist)
6. Zet veld 'Medewerker' op 'ADBW' (selector vereist)
7. Zet veld 'Team' op 'VPRW Adm PGB / BW' (selector vereist)
8. Klik op 'Opslaan' knop (selector vereist)
9. Haal het werkprocesnummer op uit het scherm — sla op in `out_WerkprocesNummer`
10. Klik op 'Opslaan en sluiten' knop (selector vereist)

### 4.9 Stap 9: ToelichtingInWerkproces (Suite)

#### 4.9.1 Algemene informatie

| Eigenschap | Waarde |
|---|---|
| Workflowbestand | `009_Suite_ToelichtingInWerkproces.xaml` |
| Applicatie | Suite |
| Doel | Plaatsen van een toelichting in het werkproces wanneer er een opmerking in de aanvraag is gevonden |
| Navigatie | Navigeer via Basis Sociaal Domein → Werkbeheersing → Werkprocessen; zoek op werkprocesnummer |
| Input argumenten | `in_Config (Dictionary<String,Object>)`, `in_TransactionItem (QueueItem)`, `in_WerkprocesNummer (String)` |
| Output argumenten | *(geen)* |

#### 4.9.2 Omschrijving handelingen

1. Klik op menu 'Basis Sociaal Domein' (selector vereist)
2. Klik op submenu 'Werkbeheersing' (selector vereist)
3. Klik op submenu 'Werkprocessen' (selector vereist)
4. Vul het `in_WerkprocesNummer` in het zoekzeld 'Werkproces' en druk op Enter (selector vereist)
5. Vul in het veld 'Toelichting' de vaste tekst: "Opmerking gevonden in aanvraag. Zie aanvraag formulier voor verdere toelichting." (selector vereist)
6. Klik op 'Opslaan en sluiten' (selector vereist)

### 4.10 Stap 10: OpenEnInloggen (DIS)

#### 4.10.1 Algemene informatie

| Eigenschap | Waarde |
|---|---|
| Workflowbestand | `010_DIS_OpenEnInloggen.xaml` |
| Applicatie | DIS |
| Doel | Openen van DIS documentmanagementsysteem; inloggen indien vereist |
| Navigatie | Start desktopapplicatie DIS; inlogmethode vermoedelijk Windows SSO via robotaccount |
| Input argumenten | `in_Config (Dictionary<String,Object>)` |
| Output argumenten | *(geen)* |

#### 4.10.2 Omschrijving handelingen

1. Start DIS desktopapplicatie (selector vereist)
2. Check App State: wacht tot DIS hoofdscherm geladen is (selector vereist)
3. Als aanmeldscherm verschijnt: TypeInto gebruikersnaamveld (selector vereist)
4. Als aanmeldscherm verschijnt: TypeSecureText wachtwoordveld (selector vereist)
5. Als aanmeldscherm verschijnt: Click inlogknop (selector vereist)
6. Check App State: wacht tot DIS werkbakscherm beschikbaar is

### 4.11 Stap 11: AanvraagFormulierOpslaan (DIS)

#### 4.11.1 Algemene informatie

| Eigenschap | Waarde |
|---|---|
| Workflowbestand | `011_DIS_AanvraagFormulierOpslaan.xaml` |
| Applicatie | DIS |
| Doel | Opslaan van het aanvraagformulier en eventuele bijlagen in DIS onder het juiste werkprocesnummer, met de correcte metagegevens (DatumBinnenkomst, BSN, WerkprocesNummer, behandelaar, classificatie) |
| Navigatie | Vanuit DIS-hoofdscherm; klik op werkbak 'WMO/JW Begeleiding & Verblijf Admin' |
| Input argumenten | `in_Config (Dictionary<String,Object>)`, `in_TransactionItem (QueueItem)`, `in_WerkprocesNummer (String)`, `in_DatumBinnenkomst (String)`, `in_BSN (String)`, `in_AanvraagPad (String)`, `in_BijlagenPaden (List<String>)` |
| Output argumenten | *(geen)* |

#### 4.11.2 Omschrijving handelingen

1. Klik op werkbak '- WMO/JW Begeleiding & Veblijf Admin' in DIS (selector vereist)
2. Open Windows Verkenner en navigeer naar `in_Config("Folder_Temp")`
3. Sleep het aanvraagformulier (`in_AanvraagPad`) en eventuele bijlagen (`in_BijlagenPaden`) naar DIS
4. Klik op 'Ja' in het bevestigingsdialoog 'WZI Werkvoorraad' (selector vereist)
5. Vul in het veld 'Datum binnenkomst' de waarde `in_DatumBinnenkomst` in (selector vereist)
6. Vul in het veld 'Cliëntnr / BSN' de waarde `in_BSN` in (selector vereist)
7. Selecteer in het veld 'Werkprocesnr' de waarde `in_WerkprocesNummer` (selector vereist)
8. Kies bij 'Behandelaar' de optie 'Clusterbakken SD / - WMO/JW Begeleiding & Veblijf Admin' (selector vereist)
9. Als classificatieveld getoond wordt: classificeer het aanvraagformulier als 'Aanvraag' en bijlagen als 'Aanvullende informatie'
10. Klik op 'OK' knop (selector vereist)

### 4.12 Stap 12: Uitval (Outlook)

#### 4.12.1 Algemene informatie

| Eigenschap | Waarde |
|---|---|
| Workflowbestand | `012_Outlook_Uitval.xaml` |
| Applicatie | Outlook |
| Doel | Versturen van een uitvalmail met het aanvraagformulier, eventuele bijlagen en de reden van uitval naar de uitval-mailbox GM_SD_startzorg – BW aanvragen Regio Uitval |
| Navigatie | Gebruik bestaande Outlook-verbinding uit Init-fase; maak een nieuw mailitem aan |
| Input argumenten | `in_Config (Dictionary<String,Object>)`, `in_TransactionItem (QueueItem)`, `in_RedenUitval (String)`, `in_AanvraagPad (String)`, `in_BijlagenPaden (List<String>)` |
| Output argumenten | *(geen)* |

#### 4.12.2 Omschrijving handelingen

1. Stel een nieuw mailitem op met als ontvanger `in_Config("Outlook_UitvalMailbox")`
2. Stel het onderwerp in: "Uitval aanvraag Beschermd Wonen — " + reden van uitval
3. Voeg de reden van uitval toe aan de mailbody (`in_RedenUitval`)
4. Voeg het aanvraagformulier toe als bijlage (`in_AanvraagPad`)
5. Voeg eventuele overige bijlagen toe (`in_BijlagenPaden`)
6. Verstuur de mail als ongelezen mail en plaats deze in de map `in_Config("Outlook_UitvalMailbox")`
