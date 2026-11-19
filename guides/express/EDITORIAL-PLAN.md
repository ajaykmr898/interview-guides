# Piano editoriale — Express.js

Requisiti dei capitoli di questa directory. Seguire le regole comuni in [AGENTS.md](../../AGENTS.md). Tutte le voci **Da coprire** sono obbligatorie; P0/P1/P2 determinano la profondità, non l'esclusione.

## Ambito e confini

- Trattare Express 5 come riferimento per le API correnti e distinguere Express 4 nelle migrazioni; verificare comportamento e compatibilità sulle [guide ufficiali](https://expressjs.com/en/guide/). Express è un framework di routing e middleware: database, validazione, sessioni e test runner richiedono integrazioni esplicite.
- Per runtime, event loop, stream, moduli e package manager usare [Node e npm](../../topics/node-tooling/08-node-npm.md); per Promise e cancellazione usare [JavaScript async](../../topics/javascript/04-javascript-async-dom.md). Qui spiegare solo il comportamento specifico degli handler e middleware Express.
- Per metodi, status, idempotenza e design delle risorse usare [API RESTful](../../topics/api/16-restful-apis.md); per HTTP, cookie, CORS e sicurezza browser usare [HTML e Web](../../topics/web/01-html-web.md); per pattern, test e osservabilità generali usare [Engineering](../../topics/engineering/15-architecture-devops.md). Nei capitoli mantenere esempi e decisioni di integrazione Express.

## `01-app-routing-middleware.md` — App routing e middleware Express

**Da coprire:**
- **P0 — Applicazione e ciclo richiesta-risposta / express() e server HTTP; Ordine di registrazione:** ruolo di `express()`, differenza tra `app` e server Node, richiesta-risposta, ordine di route e middleware e conseguenze di una risposta non conclusa.
- **P0 — Router e route / Metodi, path e parametri:** metodi `app.METHOD`, matching del path, parametri di route e distinzione tra metodo HTTP e handler.
- **P0 — Middleware / next e controllo del flusso:** firme di middleware ordinario, passaggio con `next()`, risposta terminata e rischio di chiamate multiple o richieste pendenti; rinviare gli error handler al capitolo 03.
- **P1 — Router e route / Router montati e precedenza:** `express.Router()`, mount path, route annidate e precedenza data dall'ordine; quando separare router per risorsa.
- **P1 — Middleware / Middleware applicativi, router e integrati:** scope e ordine di `app.use`/`router.use`, `express.json`, `express.urlencoded` e `express.static`; limiti di parsing e ambito di applicazione.
- **P2 — Middleware / next e controllo del flusso:** `next('route')` e sub-stack condizionali solo per distinguere salto di route, avanzamento ordinario ed errore.

## `02-request-response-validation.md` — Richieste risposte e validazione Express

**Da coprire:**
- **P0 — Input della richiesta / Params, query e body; Parsing e confini di fiducia:** provenienza e tipi effettivi di `req.params`, `req.query` e `req.body`, parser necessario, dati non attendibili e differenza fra typing statico e validazione a runtime.
- **P0 — Costruzione della risposta / Status, header e JSON:** `res.status`, `res.json`, `res.send` e `res.end`, ritorni per chaining, serializzazione e conclusione della risposta; evitare doppie risposte e status impliciti errati.
- **P1 — Validazione / Schemi e normalizzazione; Errori di input:** validare forma, tipi e vincoli prima della logica applicativa, normalizzare in modo esplicito e produrre errori client coerenti senza delegare la sicurezza al solo parser.
- **P1 — Input della richiesta / Parsing e confini di fiducia:** media type, limiti della dimensione del body e comportamento di input assente o malformato; rinvio al [contratto REST](../../topics/api/16-restful-apis.md#rappresentazioni-errori-e-collezioni) per la semantica generale.
- **P2 — Costruzione della risposta / Redirect, download e streaming:** quando usare `res.redirect`, `res.download` o stream e quali responsabilità restano al chiamante dopo l'invio degli header.

## `03-async-errors-lifecycle.md` — Asincronia errori e ciclo di vita Express

**Da coprire:**
- **P0 — Handler asincroni / Promise e propagazione degli errori; Express 4 e Express 5:** eccezioni sincrone, Promise rifiutate negli handler Express 5 e inoltro esplicito con `next(err)` nei flussi asincroni Express 4; distinguere callback esterne dalla Promise restituita dall'handler.
- **P0 — Pipeline degli errori / Middleware di errore e handler predefinito:** firma a quattro argomenti, ordine, delega e comportamento dell'handler predefinito; separare errori attesi da guasti inattesi.
- **P1 — Pipeline degli errori / 404, errori attesi e headersSent:** route non trovata distinta da un'eccezione, mapping coerente degli errori applicativi, e delega quando gli header sono già stati inviati.
- **P1 — Ciclo di vita della richiesta / Abort, timeout e cleanup:** disconnessione client, operazioni in corso e risorse da chiudere; un abort locale non implica rollback di una scrittura già avvenuta.
- **P2 — Ciclo di vita della richiesta / Stream e risposta parziale:** errori dopo l'inizio dello stream e impossibilità di sostituire una risposta parziale con un nuovo JSON.

## `04-security-auth-sessions.md` — Sicurezza autenticazione e sessioni Express

**Da coprire:**
- **P0 — Confini di sicurezza / Input, header e proxy:** request come input non attendibile, header e configurazione server; non presentare il middleware di sicurezza come sostituto della validazione o dell'autorizzazione.
- **P0 — Autenticazione e autorizzazione / Token e middleware di autorizzazione:** distinguere identità e permesso, verificare le credenziali sul server e autorizzare ogni risorsa/operazione, con ordine corretto dei middleware.
- **P1 — Autenticazione e autorizzazione / Sessioni e cookie:** integrazione di session store esterno, cookie sicuri e differenze pratiche rispetto ai token bearer; evitare sessioni in memoria per deployment condivisi.
- **P1 — Confini di sicurezza / CORS e rate limiting; Difese delle operazioni / CSRF e dati sensibili:** CORS non è autenticazione, quote e gestione degli abusi, protezione CSRF per credenziali inviate automaticamente, segreti e messaggi d'errore non divulgativi.
- **P2 — Confini di sicurezza / Input, header e proxy:** `trust proxy` e header forwarded dipendono dalla topologia del reverse proxy; descrivere il rischio di fidarsi di valori controllabili dal client.

## `05-data-architecture-persistence.md` — Architettura dei dati e persistenza Express

**Da coprire:**
- **P0 — Responsabilità dei layer / Router, controller e service; Repository e dipendenze:** Express gestisce trasporto e routing, non impone MVC né include persistenza; distinguere traduzione HTTP, regole di business e accesso ai dati con layer giustificati dal caso d'uso.
- **P0 — Accesso ai dati / Driver, pool e query:** collegare un driver o ORM esplicito, attendere I/O asincrono e usare query parametrizzate; spiegare il confine tra dato persistente e oggetti request-scoped.
- **P1 — Accesso ai dati / Errori e mapping HTTP; Consistenza / Transazioni e concorrenza:** non esporre errori del database, tradurre assenza/conflitto, collocare le transazioni attorno a una unità di business e considerare aggiornamenti concorrenti.
- **P1 — Responsabilità dei layer / Repository e dipendenze:** quando un repository o service facilita test, sostituzione della fonte dati e riuso; evitare wrapper che inoltrano soltanto chiamate.
- **P2 — Consistenza / Migrazioni e unità di lavoro:** collocazione di schema migration e scelta tra driver, query builder e ORM; rinviare i principi architetturali generali a [Engineering](../../topics/engineering/15-architecture-devops.md).

## `06-testing-operations-migrations.md` — Testing operatività e migrazioni Express

**Da coprire:**
- **P0 — Test / Handler e middleware; Test HTTP di integrazione:** separare creazione di `app` da `listen`, testare status/body/header e flussi di errore con un client HTTP di test, isolando database e servizi esterni; distinguere unit e integration test.
- **P1 — Operatività / Configurazione e osservabilità; Avvio, shutdown e reverse proxy:** configurazione per ambiente, log strutturati senza segreti, health check, gestione degli errori di processo, chiusura del server e delle connessioni e fiducia nel proxy.
- **P1 — Evoluzione / Migrazione Express 4 a Express 5:** differenze verificate in path matching, handler Promise e API modificate/rimosse; test e migrazione incrementale, senza trattare cambiamenti introdotti come già presenti in Express 4.
- **P2 — Evoluzione / Compatibilità Node e dipendenze:** controllare runtime minimo, peer dependencies, middleware legacy e release di sicurezza; valutare profiling o load test solo su colli di bottiglia misurati.

## `07-todolist-end-to-end.md` — Esempio TodoList end-to-end Express

**Da coprire:**
- **P0 — Modello e contratto REST:** usare `Todo { id: string, title: string, completed: boolean }` e gli endpoint condivisi in [AGENTS.md](../../AGENTS.md), con input, output, status e proprietà di ogni operazione.
- **P0 — Struttura e responsabilità / Router e controller; Service e repository:** mostrare una struttura minima, router per endpoint, controller per traduzione HTTP, service per regole del dominio e repository per persistenza; non attribuire questi layer al core Express.
- **P0 — Flusso CRUD e persistenza:** seguire lettura, creazione, modifica e cancellazione fino alla fonte dati e ritorno alla risposta; usare una persistenza rappresentativa e gestire ID e stato `completed` lato server.
- **P1 — Validazione, errori e cleanup:** trim e titolo non vuoto, booleano per `completed`, almeno un campo nel `PATCH`, `400`/`404`/`5xx`, `204` senza corpo e gestione delle risorse in caso di fallimento o interruzione.
- **P1 — Test rappresentativo:** un test HTTP minimo che attraversi router, controller e service con confine della persistenza controllato, verificando contratto e risultato.
- **P2 — Struttura e responsabilità / Service e repository:** motivare repository e scelta del driver, evitando ORM, astrazioni o stato globale non necessari all'esempio.

## `08-ecosystem-libraries.md` — Ecosistema librerie e convenzioni Express

**Da coprire:**
- **P0 — Criteri di scelta e stato:** selezione curata per colloqui; distinguere API e middleware inclusi in Express, strumenti ufficiali e librerie esterne facoltative. Per ogni opzione indicare scopo, casi d'uso, alternativa e trade-off, stato, link alla documentazione del progetto e compatibilità/manutenzione quando rilevanti; aggiornare **Last reviewed** dopo la verifica.
- **P1 — Routing e middleware; Validazione e sicurezza:** scelte diffuse per parsing, validazione, header, CORS, sessioni e rate limiting, senza presentare preferenze della community come standard ufficiali.
- **P1 — Persistenza e accesso ai dati; Testing:** driver/ORM e strumenti di test pertinenti, con limiti e rinvii ai capitoli concettuali.
- **P1 — Performance e operatività; Tooling e convenzioni:** strumenti e pratiche comuni per log, osservabilità, compressione, esecuzione e linting, distinguendo funzioni del core da integrazioni opzionali.
- **P2 — Criteri di scelta e stato:** alternative specialistiche, vincoli di versione e segnali di manutenzione; evitare cataloghi esaustivi o pacchetti aggiunti solo per simmetria con altre guide.
