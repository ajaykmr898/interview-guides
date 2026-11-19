# Architettura dei dati e persistenza Express

Express instrada richieste HTTP (Hypertext Transfer Protocol), ma non prescrive un'architettura MVC (Model-View-Controller) né fornisce persistenza. Router, controller, service e repository sono confini applicativi da introdurre quando chiariscono responsabilità e dipendenze.

## Indice

- [Responsabilità dei layer](#responsabilità-dei-layer)
- [Accesso ai dati](#accesso-ai-dati)
- [Consistenza](#consistenza)

## Responsabilità dei layer

### Router, controller e service

| Layer | Responsabilità in un'app Express |
| --- | --- |
| Router | Associa metodo e path agli handler; compone middleware di autenticazione, validazione e controller. |
| Controller | Traduce input validato e risultato applicativo in status, header e corpo HTTP; non contiene query o regole di business complesse. |
| Service | Esegue un caso d'uso e le sue regole di dominio, eventualmente coordinando più accessi ai dati in una transazione. |

Il controller è una funzione handler, non una classe richiesta da Express. Passare al service valori già validati, non `req` e `res`; il service restituisce un risultato o segnala un esito applicativo che il controller mappa in risposta. Una route semplice può chiamare direttamente il data access senza un service che inoltra soltanto metodi. Per parsing e [validazione dell'input](02-request-response-validation.md#validazione) vedi il capitolo dedicato.

### Repository e dipendenze

Un **repository** espone operazioni di persistenza nel linguaggio del dominio, per esempio `findTodo(id)` o `saveTodo(todo)`, nascondendo SQL e dettagli del driver. È utile quando query, mapping o fonte dati hanno complessità reale, quando più casi d'uso li condividono o quando serve isolare la persistenza nei test. Aggiunge però indirezione e può nascondere costi e transazioni: non creare un'interfaccia per ogni tabella solo per imitare un ORM.

Creare il pool e gli oggetti service all'avvio e iniettarli nei moduli che li usano; non ricrearli per ogni richiesta né usare stato globale mutabile per dati request-scoped. Un client del pool si acquisisce per l'operazione che lo richiede e si rilascia al termine. Un repository deve poter ricevere il client della transazione quando il service coordina più scritture. Per il [pattern Repository e i suoi trade-off](../../topics/engineering/15-architecture-devops.md#pattern-e-struttura) vedi Engineering.

## Accesso ai dati

Un driver o un ORM (Object-Relational Mapper) collega l'app a un database; Express non interpreta schema, query o risultati. La scelta dipende da database, modello e complessità delle query.

### Driver, pool e query

Con un driver asincrono, attendere le operazioni di input/output (I/O) prima di rispondere. Un pool riusa connessioni tra richieste; le query singole possono acquisire e rilasciare automaticamente un client tramite un metodo del pool, mentre un client preso esplicitamente va sempre rilasciato. Chiudere il pool durante lo shutdown. Le credenziali devono arrivare da configurazione esterna, validata all'avvio.

Esempio PostgreSQL con il driver esterno `pg`, una tabella `todos` esistente e `DATABASE_URL` configurato:

```js
const { Pool } = require('pg');
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

async function findTodo(id) {
  const { rows } = await pool.query(
    'SELECT id, title, completed FROM todos WHERE id = $1', [id]);
  return rows[0] ?? null;
}
```

`pool.query(text, values)` restituisce una Promise del risultato; `rows` contiene gli oggetti letti e l'assenza è tradotta in `null` dal repository. Il placeholder `$1` separa il valore dalla stringa SQL (Structured Query Language), prevenendo l'iniezione attraverso `id`; non può sostituire nomi di tabelle o colonne, che vanno scelti da una allowlist se dinamici. Anche con parametri SQL, validare i dati prima del repository. Per la gestione generale delle [operazioni asincrone Node](../../topics/node-tooling/08-node-npm.md#runtime-node) vedi Node e npm.

### Errori e mapping HTTP

Una connessione persa, un timeout o un vincolo violato fanno fallire la Promise del driver: il repository non deve restituire silenziosamente un valore valido né esporre query e stack trace al client. Il service distingue esiti attesi (risorsa assente, conflitto noto) da guasti inattesi; controller o error handler li traducono in `404`, `409` o `5xx` secondo il contratto. Un errore del database non è automaticamente un errore di input `400`. In Express 5 una Promise rifiutata da un handler `async` raggiunge la [pipeline degli errori](03-async-errors-lifecycle.md#pipeline-degli-errori); mantenere i dettagli diagnostici nei log protetti.

## Consistenza

La consistenza riguarda le invarianti dei dati quando più operazioni o richieste agiscono sugli stessi record; Express non le garantisce tramite l'ordine dei middleware.

### Transazioni e concorrenza

Se un caso d'uso richiede più scritture atomiche, il service definisce l'unità di lavoro: acquisisce un client, avvia la transazione, esegue tutte le query sullo **stesso client**, conferma con `COMMIT` oppure esegue `ROLLBACK` in caso di errore, e rilascia il client in `finally`. Con `pg`, chiamare più volte `pool.query()` non mantiene la stessa transazione, perché ogni chiamata può usare una connessione diversa. Tenere la transazione breve e non includere attese esterne non necessarie.

Una transazione non elimina ogni conflitto concorrente. Per evitare *lost update*, si può usare un controllo ottimistico della versione nel `WHERE` dell'`UPDATE` e verificare le righe modificate: zero righe può indicare versione cambiata o risorsa assente, da distinguere. In alternativa, un lock come `SELECT ... FOR UPDATE` dentro una transazione serializza accessi alla stessa riga. Lock e isolamento più forte hanno costi di attesa; deadlock e fallimenti di serializzazione richiedono una politica di retry sicura per l'intera unità di lavoro. Per precondizioni e [concorrenza nel contratto HTTP](../../topics/api/16-restful-apis.md#cache-concorrenza-ed-evoluzione) vedi il contratto delle risorse.

### Migrazioni e unità di lavoro

Le **migrazioni** versionano le modifiche allo schema e si eseguono nel processo di rilascio, non in ogni richiesta. Coordinare il cambio dello schema con le versioni dell'app e prevedere una fase compatibile quando vecchio e nuovo codice convivono. Il driver dà controllo diretto sulle query; un query builder aiuta a costruirle, mentre un ORM aggiunge mapping di entità e spesso migrazioni, con costi di astrazione e attenzione alle query generate. Nessuna di queste scelte è imposta da Express: scegliere la più semplice che renda espliciti query, transazioni e responsabilità del caso d'uso. L'[unità di lavoro](#transazioni-e-concorrenza) resta nel service che coordina l'operazione.
