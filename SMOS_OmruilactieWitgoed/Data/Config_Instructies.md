# Config.xlsx — Invulinstructies SMOS_OmruilactieWitgoed

Vul Data/Config.xlsx in met de onderstaande waarden vóór de eerste run.
Verwijder dit bestand nadat Config.xlsx volledig is ingevuld.

---

## Sheet: Settings

| Name | Value | Description |
|------|-------|-------------|
| UseOrchestratorTransactions | True | Must be True or False |
| OrchestratorQueueName | SMOS_OmruilactieWitgoed | Orchestrator Queue Name. Match with server. |
| logF_BusinessProcessName | OmruilactieWitgoed | Groups log data under one business process name |

---

## Sheet: Constants

| Name | Value | Description |
|------|-------|-------------|
| ProjectWithDispatcher | False | Geen dispatcher/performer split |
| QueueRetry | False | Geen Orchestrator QueueRetry |
| MaxRetryNumber_FrameworkRetry | 0 | Must be 0 if working with Orchestrator queues |
| MaxConsecutiveSystemExceptions | 0 | Set to 0 to disable consecutive exception limit |
| ExScreenshotsFolderPath | Exceptions_Screenshots | Relative path for exception screenshots |
| LogMessage_GetTransactionData | Processing Transaction Number:  | Static logging message prefix |
| LogMessage_GetTransactionDataError | Error getting transaction data for Transaction Number:  | Static logging message prefix |
| LogMessage_Success | Transaction Successful. | Logged on successful transaction |
| LogMessage_BusinessRuleException | Business rule exception.  | Logged on BusinessRuleException |
| LogMessage_ApplicationException | System exception.  | Logged on ApplicationException |
| ExceptionMessage_ConsecutiveErrors | The maximum number of consecutive system exceptions was reached.  | Logged when MaxConsecutiveSystemExceptions is reached |

---

## Sheet: Assets

| Name | Asset | Folder | Description |
|------|-------|--------|-------------|
| MaxTransactions | MaxTransactions | | Must be integer, or INPUTDIALOG, or ALL (unlimited) |
| Folder_Temp | Folder_Temp | | Tijdelijk mappad |
| Folder_Log | Folder_Log | | Logmap pad |
| LogMessageAddress | LogMessageAddress | | A single dash will be converted to Nothing by the framework |
| Suite_Credential | Suite_Credential | | Inloggegevens Suite (gebruikersnaam + wachtwoord) |
| Suite_URL | Suite_URL | | URL van de Suite omgeving |

---

## Sheet: CreateAssets

| Name | ValueType | Value | CreateAsset | Comments |
|------|-----------|-------|-------------|----------|
| MaxTransactions | Text | ALL | True | Must be integer, or INPUTDIALOG, or ALL (unlimited) |
| Folder_Temp | Text | Data\Temp | True | |
| Folder_Log | Text | Data\Log | True | |
| LogMessageAddress | Text | - | True | A single dash will be converted to Nothing by the framework |
| Suite_Credential | Credential | | True | **VÓÓR EERSTE RUN INVULLEN** |
| Suite_URL | Text | <<FILL_BEFORE_FIRST_RUN>> | True | URL van de Suite omgeving |

---

## Sheet: LogMessageHtml

| variable | replace with |
|----------|-------------|
| [var_subject] | RPA-proces - OmruilactieWitgoed |
| [var_procesnaam] | OmruilactieWitgoed |
| [var_afzender] | Virtual Assistant RPA |

---

## Sheet: LogmailAttachment

| KeyTransactionDetails | Description | General Comments |
|-----------------------|-------------|-----------------|
| WerkprocesID | Werkproces-ID / volgnummer uit Suite | |
| HeeftBRPKoppeling | True = gefaseerd, False = uitval BRP | |
