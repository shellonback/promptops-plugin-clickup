# Dal connettore backend al plugin: stato della migrazione ClickUp

Questo plugin è il primo passo per spostare l'integrazione ClickUp dal backend a un plugin. Oggi i due convivono: l'integrazione del backend resta quella in uso, il plugin è pronto ma la board non legge ancora dai plugin.

## Cosa è stato portato

Sorgente: `manager.promptops.it/backend/app/Services/Integrations/ClickUpProvider.php`.

| Backend | Plugin | Note |
|---|---|---|
| `validateToken()` | `validate()` | Restituisce l'account. I workspace arrivano da `listContainers(null)` |
| `getWorkspaces()`, `getSpaces()`, `getFolders()`, `getFolderlessLists()` | `listContainers(parentId)` | Un solo metodo ad albero. Gli id portano il livello: `team:`, `space:`, `folder:`, `list:` |
| liste dentro la risposta di `getFolders()` | `listContainers('folder:ID')` | Una chiamata in più per cartella aperta, in cambio di un albero uniforme |
| `getListStatuses()` | `listStatuses()` | Stesso ordinamento per `orderindex`, stesso nome normalizzato. In più il tipo: `open` diventa `todo`, `custom` diventa `in_progress`, `done` e `closed` diventano `done` |
| `fetchTasks()` | `listTasks()` | Stessi parametri: `include_closed`, `subtasks`, `date_updated_gt`, `statuses[]`. `lastPage` diventa `nextCursor` |
| `fetchTaskIds()` | non portato | Era un ciclo su `fetchTasks`: lo fa chi sincronizza, seguendo `nextCursor` |
| `fetchTask()` | `getTask()` | |
| `pushStatus()` | `setStatus()` | Restituisce il task aggiornato |
| `getTaskComments()` | `listComments()` | Stessa ricostruzione dei commenti formattati dai blocchi `comment[]` |
| `mapTask()` | `toTask()` | Stessa mappa delle priorità, email minuscole e ordinate, descrizione vuota trattata come assente |
| `http()` con retry su rete, 429 e 5xx | `clickup()` | Due tentativi in più, attesa di 1,5 secondi o di `retry-after` |
| `buildQuery()` per `statuses[]` | `query()` | |

Il contratto `tasks` dei modelli è stato esteso per non perdere nulla: `kind: folder`, `color` sugli stati, e su ogni task `priority`, `order` e `assigneeEmails`. La suite in `test/` verifica ogni riga di questa tabella.

## Cosa NON può fare un plugin, e oggi fa il backend

| Funzione | Oggi | Con il plugin |
|---|---|---|
| **Webhook** di ClickUp, aggiornamenti in tempo reale | `createWebhook()`, `deleteWebhook()`, rotta pubblica con segreto | Non disponibile. Un plugin gira sul desktop e non ha un indirizzo pubblico: solo polling |
| **Token condiviso dal team**, configurato una volta per progetto | `ProjectIntegration` con token cifrato nel backend | Il token è locale, per utente e per dispositivo. Ogni membro che vuole sincronizzare configura il suo |
| **Sync lato server**, anche con tutte le app chiuse | `TaskSyncService` su schedule e su webhook | La sincronizzazione avviene solo mentre l'app di qualcuno è aperta |
| Persistenza: link task esterno, hash del payload, cursore, ultimo errore | `TaskExternalLink`, `ProjectIntegration` | Da spostare in chi consuma il contratto, non nel plugin |
| Gestione degli accessi revocati | `IntegrationAccessFailureService` | Il plugin restituisce un errore chiaro sul 401, ma la reazione è dell'app |
| Colonne dinamiche e mappa stato-colonna | `integration-columns-modal`, `integration-settings` nella webapp | Da rifare sopra `listStatuses()` |

Le prime tre righe sono una perdita reale di funzione per i team. Prima di spegnere l'integrazione del backend serve una decisione: accettarla, oppure tenere per ClickUp un connettore di prima parte nel backend che espone lo stesso contratto del plugin.

## Cosa manca nell'app perché il plugin sostituisca il backend

1. **La board che legge dal contratto.** Oggi `plugin_runtime_invoke` chiama il plugin, ma nessuna schermata usa il risultato.
2. **Il ponte di sincronizzazione.** La webapp, parte fidata, chiama `listTasks()` seguendo `nextCursor` e scrive i task nel backend con le API esistenti. Il plugin non parla mai con il backend.
3. **Il publisher ufficiale.** L'handle `promptops` è riservato e un utente non può crearlo: il profilo va creato da seeder o da admin.
4. **Segreti nel portachiavi di sistema.** Oggi stanno in un file con permessi 0600.

## Percorso proposto

1. Pubblicare il plugin e collaudarlo su un progetto interno con il pannello **Test contract**, in parallelo all'integrazione del backend.
2. Collegare la board al contratto dietro un feature flag, solo lettura.
3. Aggiungere il ponte di sincronizzazione e lo spostamento delle card.
4. Migrare progetto per progetto. Spegnere il connettore del backend solo dopo la decisione su webhook e token di team.
