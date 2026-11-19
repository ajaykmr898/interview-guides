# Ecosistema librerie e convenzioni Express

**Last reviewed:** 2026-11-19.

## Indice

- [Criteri di scelta e stato](#criteri-di-scelta-e-stato)
- [Routing e middleware](#routing-e-middleware)
- [Validazione e sicurezza](#validazione-e-sicurezza)
- [Persistenza e accesso ai dati](#persistenza-e-accesso-ai-dati)
- [Testing](#testing)
- [Performance e operatività](#performance-e-operatività)
- [Tooling e convenzioni](#tooling-e-convenzioni)

## Criteri di scelta e stato

L'**ecosistema Express** comprende le API (Application Programming Interface) del framework, middleware distribuiti separatamente e strumenti di altri progetti. Express 5 richiede Node.js 18 o successivo; non include database, validator, session store o test runner. Le opzioni qui sotto sono scelte *condizionali* per colloqui e progetti reali, non uno stack obbligatorio. «Team Express» indica un pacchetto separato mantenuto dal team Express, non una funzione integrata; «esterno» non implica approvazione ufficiale di Express.

Scegliere prima in base al contratto dell'app, poi verificare release supportate, requisiti Node/TypeScript, compatibilità con Express 5, manutenzione e avvisi di sicurezza; usare una release Node ancora mantenuta. Un middleware presente nel catalogo Express non è necessariamente mantenuto dal team: la [pagina ufficiale distingue i propri moduli dai pacchetti terzi](https://expressjs.com/en/resources/middleware/). Per il ruolo dei middleware vedi [routing e middleware](01-app-routing-middleware.md#middleware); per le scelte di runtime e package manager vedi [Node e npm](../../topics/node-tooling/08-node-npm.md).

## Routing e middleware

| Opzione e stato | Scopo e uso comune | Alternativa e limite |
| --- | --- | --- |
| [`express.Router()`](https://expressjs.com/en/5x/api/router/) — **core Express** | Raggruppa route e middleware per risorsa, montabili sotto un path. | `app.METHOD()` basta per poche route; ordine e sintassi dei path di Express 5 vanno verificati nelle migrazioni da 4. |
| [`express.json()` / `express.urlencoded()`](https://expressjs.com/en/5x/api/express/) — **core Express** | Analizzano body con media type compatibile prima degli handler; configurare limiti di dimensione. | Per altri formati serve un parser adatto; il parsing non valida forma e regole del dominio, e `req.body` può restare `undefined`. |
| [`express.static()`](https://expressjs.com/en/5x/starter/static-files/) — **core Express** | Serve asset locali in applicazioni semplici. | Per asset pubblici ad alto traffico un reverse proxy o servizio statico può gestire cache e distribuzione; controllare directory esposta e cache. |

Express esegue questi middleware nell'ordine di registrazione: montare parser e controlli prima delle route che li usano. L'esempio [TodoList](07-todolist-end-to-end.md#router-e-controller) mostra il router senza introdurre un'altra libreria.

## Validazione e sicurezza

| Opzione e stato | Scopo e uso comune | Alternativa e limite |
| --- | --- | --- |
| [Zod](https://zod.dev/basics) — **esterno, facoltativo** | Schemi riutilizzabili per validare dati a runtime, anche fuori da Express; utile con tipi TypeScript condivisi. | Controlli locali bastano per input piccoli; gli schemi richiedono adattamento degli errori a HTTP (Hypertext Transfer Protocol). Zod 4 richiede un TypeScript supportato e modalità `strict` nei progetti TypeScript. |
| [express-validator](https://express-validator.github.io/docs/guides/validation-chain/) — **esterno, facoltativo** | Catene di validazione/sanitizzazione per `body`, `param` e `query` come middleware di route. | Zod separa meglio gli schemi dal trasporto; i validator standard possono convertire valori in stringa, quindi controllare i tipi e leggere gli errori prima del controller. |
| [Helmet](https://helmet.js.org/) — **esterno, facoltativo** | Imposta header di sicurezza HTTP per app rivolte al browser. | Gli header si possono configurare direttamente o al proxy; verificare la Content Security Policy rispetto agli asset dell'app. Non sostituisce autenticazione o autorizzazione. |
| [`cors`](https://expressjs.com/en/resources/middleware/cors/) — **Team Express, pacchetto separato** | Configura gli header CORS (Cross-Origin Resource Sharing) quando un browser chiama un'origine diversa. | Una policy al gateway è possibile; CORS controlla la lettura nel browser, non l'accesso dei client non browser. Con credenziali servono origini esplicite. |
| [`express-session`](https://expressjs.com/en/resources/middleware/session/) — **Team Express, pacchetto separato** | Mantiene stato di sessione lato server e invia al browser un cookie con l'identificatore. | Un flusso con token bearer ha diverso costo di lookup e revoca; `MemoryStore` non è adatto alla produzione: scegliere uno store condiviso e verificare cookie/proxy. |
| [`express-rate-limit`](https://express-rate-limit.mintlify.app/quickstart/usage) — **esterno, facoltativo** | Limita richieste a login o route costose e può rispondere `429`. | Il gateway può applicare limiti prima dell'app; lo store in memoria è per processo, quindi più istanze richiedono store condiviso o limite al gateway. Verificare `trust proxy` e chiave usata. |

La validazione resta obbligatoria sul server anche se esiste un form client. Per CSRF (Cross-Site Request Forgery), autorizzazione e cookie vedi [sicurezza e sessioni](04-security-auth-sessions.md#difese-delle-operazioni); l'elenco non prescrive un singolo pacchetto di autenticazione.

## Persistenza e accesso ai dati

| Opzione e stato | Scopo e uso comune | Alternativa e limite |
| --- | --- | --- |
| [`pg` / node-postgres](https://node-postgres.com/features/pooling) — **esterno, facoltativo** | Driver PostgreSQL con pool e query SQL (Structured Query Language) parametrizzate; adatto quando si vuole SQL esplicito e controllo delle transazioni. | Un ORM (Object-Relational Mapping) riduce parte del codice di mapping ma aggiunge convenzioni; con `pg` schema, migrazioni e mapping restano responsabilità dell'app. |
| [Prisma ORM 7](https://www.prisma.io/docs/orm/v7) — **esterno, facoltativo** | Accesso ai dati tipizzato e migrazioni, utile in progetti TypeScript che adottano il suo modello. | `pg` offre più controllo diretto sul SQL. Alla data di revisione Prisma 8 è [release candidate](https://www.prisma.io/docs/orm/release-status): CLI (Command Line Interface), API e requisiti Node/TypeScript differiscono; fissare la major e usare la documentazione corrispondente. |

Nessuno dei due è incluso in Express. Un repository applicativo è utile se isola query e persistenza; non serve aggiungerlo come semplice passaggio di chiamate. Vedi [driver, pool, transazioni e migrazioni](05-data-architecture-persistence.md#accesso-ai-dati).

## Testing

| Opzione e stato | Scopo e uso comune | Alternativa e limite |
| --- | --- | --- |
| [`node:test`](https://nodejs.org/api/test.html) — **core Node.js, non Express** | Test runner e asserzioni con `node:assert`, adatti a unit e test d'integrazione senza runner esterno. | Un runner già adottato dal progetto può offrire altre integrazioni; il runner da solo non invia richieste HTTP. |
| [Supertest](https://github.com/forwardemail/supertest#readme) — **esterno, facoltativo** | Invia richieste a un'app Express senza gestire manualmente una porta e verifica status, header e body. | Un client HTTP generico esercita anche il server in ascolto, ma richiede gestione di porta e lifecycle; isolare database e servizi esterni nei test mirati. |

Separare creazione di `app` da `listen()` facilita i test. Per il confine tra unit e integration test vedi [testing Express](06-testing-operations-migrations.md#test); per un test completo della risorsa vedi [TodoList](07-todolist-end-to-end.md#test-rappresentativo).

## Performance e operatività

| Opzione e stato | Scopo e uso comune | Alternativa e limite |
| --- | --- | --- |
| [Pino](https://github.com/pinojs/pino#readme) — **esterno, facoltativo** | Log JSON (JavaScript Object Notation) strutturati per correlare richieste ed errori in produzione. | [`morgan`](https://expressjs.com/en/resources/middleware/morgan/) registra richieste HTTP ed è mantenuto dal team Express, ma da solo non struttura i log applicativi; non registrare segreti. |
| [OpenTelemetry JavaScript](https://opentelemetry.io/docs/languages/js/getting-started/nodejs/) — **ecosistema OpenTelemetry, esterno a Express** | Tracce e metriche per seguire richieste fra servizi; l'auto-instrumentation include Express. | Per una piccola app possono bastare log e metriche del servizio; richiede SDK (Software Development Kit), exporter e inizializzazione prima dell'app. Verificare versione Node e modalità di moduli. |
| [`compression`](https://expressjs.com/en/resources/middleware/compression/) — **Team Express, pacchetto separato** | Comprime risposte quando l'app invia contenuti comprimibili direttamente. | Un reverse proxy può comprimere fuori dal processo; evitare lavoro doppio e misurare carico del processore, dimensione e latenza prima di abilitarlo ovunque. |

Misurare prima di ottimizzare; in produzione servono anche health check, shutdown e configurazione del proxy, spiegati in [operatività Express](06-testing-operations-migrations.md#operatività).

## Tooling e convenzioni

| Opzione e stato | Scopo e uso comune | Alternativa e limite |
| --- | --- | --- |
| [`node --watch`](https://nodejs.org/api/cli.html#--watch) — **core Node.js, non Express** | Riavvia il processo quando cambiano i file durante lo sviluppo. | [nodemon](https://nodemon.io/) offre controlli ulteriori sul watch; `--watch` è stabile da Node 20.13/22, mentre Express 5 supporta anche Node 18: verificare il runtime del progetto. Non usarlo come supervisore di produzione. |
| [ESLint](https://eslint.org/docs/latest/use/getting-started) — **esterno, facoltativo** | Regole statiche JavaScript/TypeScript in editor e CI (Continuous Integration). | Review e test coprono comportamenti che il lint non prova; scegliere una configurazione coerente con il progetto e controllare i requisiti Node della major installata. |

Convenzione pratica: script di progetto per avvio, test e lint; configurazione e segreti fuori dal codice; processo di produzione gestito dal deploy. Per gli strumenti Node e npm vedi il [capitolo canonico](../../topics/node-tooling/08-node-npm.md); per CI e osservabilità generali vedi [Engineering](../../topics/engineering/15-architecture-devops.md).
