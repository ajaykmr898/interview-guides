# Testing operatività e migrazioni Express

Test e operatività verificano che un'app Express rispetti il contratto HTTP (Hypertext Transfer Protocol) e resti gestibile durante avvio, errori e rilascio. La migrazione tra versioni richiede di controllare comportamenti osservabili, non soltanto l'avvio dell'app.

## Indice

- [Test](#test)
- [Operatività](#operatività)
- [Evoluzione](#evoluzione)

## Test

### Handler e middleware

Separare la funzione che crea e configura `app` dal file che chiama `app.listen()`: i test possono così usare la stessa pipeline di route e middleware senza avviare il listener di produzione. Iniettare service e repository controllati nel test, evitando un database o chiamate esterne reali quando non sono l'oggetto della verifica.

Un **unit test** di un handler o middleware controlla una decisione locale, per esempio se chiama `next(err)` o quale risposta prepara, con dipendenze sostituite. Non basta a provare ordine dei middleware, parsing e serializzazione: questi richiedono un test HTTP di integrazione. Testare almeno successo, input non valido, risorsa assente e errore inatteso; per i [flussi degli errori](03-async-errors-lifecycle.md#pipeline-degli-errori) vedi il capitolo dedicato.

### Test HTTP di integrazione

Un test di integrazione invia una richiesta all'app e osserva status, header e corpo attraverso router, parser, controller ed error handler. `supertest` è un client di test esterno, non parte di Express: può ricevere direttamente `app` e usare una porta temporanea. L'esempio assume che `./app` esporti `createApp()` e che la factory registri `GET /health` con risposta `{ ok: true }`:

```js
const { test } = require('node:test');
const assert = require('node:assert/strict');
const request = require('supertest');
const { createApp } = require('./app');

test('GET /health', async () => {
  const response = await request(createApp()).get('/health');
  assert.equal(response.status, 200);
  assert.deepEqual(response.body, { ok: true });
});
```

Per route con dati, verificare anche `Content-Type`, body e errori `4xx`/`5xx` con un confine di persistenza controllato; un `200` da solo non prova la correttezza. Test separati con database reale servono per query, vincoli e transazioni quando questi sono il rischio. La distinzione generale tra [tipi di test](../../topics/engineering/15-architecture-devops.md#qualità-e-tradeoff) è in Engineering.

## Operatività

L'operatività comprende configurazione, osservabilità e gestione del processo che ospita `app`; Express non fornisce automaticamente un sistema di deployment.

### Configurazione e osservabilità

Leggere configurazione e segreti dall'ambiente e validarli all'avvio: porta, indirizzo del database e chiavi mancanti devono fallire prima di accettare traffico. In produzione evitare stack trace nelle risposte e usare log strutturati con metodo, path, status, durata e identificativo di correlazione, senza copiare body, token o credenziali. Misurare latenza e tasso di errori per distinguere guasti dell'app da dipendenze lente.

Un endpoint di **liveness** segnala che il processo risponde; la **readiness** indica se può ricevere traffico, considerando le dipendenze indispensabili. Un health check non sostituisce test funzionali né osservabilità. Per [log, metriche e tracing](../../topics/engineering/15-architecture-devops.md#observability) vedi Engineering.

### Avvio, shutdown e reverse proxy

`app.listen()` restituisce un server Node. All'arresto, rendere l'istanza non pronta, smettere di accettare nuove connessioni con `server.close()`, lasciare un tempo limitato alle richieste in corso e poi chiudere pool e altre risorse. Una richiesta che non termina può prolungare lo shutdown; prevedere una deadline e coordinare cancellazione e cleanup senza interrompere indiscriminatamente scritture già confermate. Non usare `uncaughtException` per continuare a servire richieste dopo un guasto che può aver lasciato lo stato incoerente: il processo va terminato e riavviato dal supervisore.

Un reverse proxy può terminare TLS (Transport Layer Security), comprimere o distribuire traffico. Configurare `trust proxy` solo per proxy realmente fidati, perché `req.ip`, `req.hostname` e `req.protocol` possono dipendere dagli header forwarded; vedi [sicurezza e proxy](04-security-auth-sessions.md#input-header-e-proxy). I dettagli del [server HTTP Node](../../topics/node-tooling/08-node-npm.md#backend-http-ed-express) restano nel capitolo Node.

## Evoluzione

L'evoluzione dell'app richiede test dei contratti e verifica delle dipendenze prima di sostituire una major di Express.

### Migrazione Express 4 a Express 5

| Aspetto | Express 4 | Express 5 |
| --- | --- | --- |
| Path string delle route | Accetta pattern legacy come `/*` e `/:file.:ext?`. | Wildcard nominato (`/*splat`); segmenti opzionali con parentesi graffe (`/:file{.:ext}`). `/{*splat}` include anche la root. |
| Handler `async` | Rejection da inoltrare esplicitamente a `next(err)`. | Rejection della Promise restituita inoltrata automaticamente. |
| Body e query | `req.body` non analizzato parte da `{}`; query parser predefinito `extended`. | `req.body` non analizzato è `undefined`; query parser predefinito `simple` e `req.query` è un getter. |
| Risposte legacy | Firme come `res.send(status)` e `res.json(body, status)` erano accettate. | Usare `res.sendStatus(status)` o `res.status(status).json(body)`. |

I path string con caratteri di espressione regolare richiedono una revisione specifica; i path definiti come `RegExp` restano supportati. Cercare route e metodi obsoleti, verificare o aggiornare i middleware esterni, poi eseguire test HTTP di successo ed errore su ogni endpoint prima di distribuire gradualmente la nuova versione. Non attribuire la propagazione automatica delle Promise a Express 4. Per i dettagli degli [handler asincroni](03-async-errors-lifecycle.md#express-4-e-express-5) vedi il capitolo precedente.

### Compatibilità Node e dipendenze

Express 5 richiede almeno Node.js 18, ma in produzione scegliere una release Node supportata e verificare la matrice del progetto. Controllare `engines`, peer dependencies, middleware legacy e compatibilità del test runner, oltre agli aggiornamenti di sicurezza di Express e delle dipendenze; il lockfile non rende sicura una versione vulnerabile. Le regole generali per [versioni e dipendenze Node](../../topics/node-tooling/08-node-npm.md#versioni-e-sicurezza-delle-dipendenze) sono nel capitolo Node. Profiling e load test sono utili per colli di bottiglia misurati, non come passaggio automatico di ogni migrazione.
