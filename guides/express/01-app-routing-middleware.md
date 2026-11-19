# App routing e middleware Express

Express è un framework Node.js per instradare richieste HTTP (Hypertext Transfer Protocol) attraverso route e middleware. L'ordine di registrazione determina quali funzioni elaborano una richiesta e chi invia la risposta.

## Indice

- [Applicazione e ciclo richiesta-risposta](#applicazione-e-ciclo-richiesta-risposta)
- [Router e route](#router-e-route)
- [Middleware](#middleware)

## Applicazione e ciclo richiesta-risposta

### express() e server HTTP

`express()` crea `app`, un'applicazione invocabile come listener di richieste del server HTTP di Node.js. `app` registra route e middleware; `app.listen()` avvia l'ascolto e restituisce un `http.Server`, che gestisce la connessione di rete e può essere chiuso separatamente. L'app può essere passata a un server o a un test senza chiamare `listen()`.

```js
const express = require('express');
const app = express();

app.get('/health', (_req, res) => res.json({ ok: true }));
const server = app.listen(3000);
```

Per ogni richiesta, Express percorre le funzioni compatibili finché una invia la risposta, passa il controllo o segnala un errore. `res.json()` invia e conclude la risposta; il valore restituito dall'handler non viene usato da Express come risposta. Per il server HTTP e il runtime sottostante, vedi [Node e npm](../../topics/node-tooling/08-node-npm.md#backend-http-ed-express).

### Ordine di registrazione

Middleware e route vengono esaminati nell'ordine in cui sono registrati. Un middleware di parsing deve precedere le route che leggono `req.body`; una route che conclude la risposta impedisce alle route successive di gestire la stessa richiesta. Un gestore per i casi senza route va dopo le route, mentre i middleware di errore vanno dopo quelli ordinari (vedi [pipeline degli errori](03-async-errors-lifecycle.md#pipeline-degli-errori)).

**Regola pratica:** in ciascun ramo dell'handler, inviare una risposta oppure passare il controllo. Se non avviene nessuna delle due cose, la richiesta resta pendente.

## Router e route

Una *route* associa un metodo HTTP e un path a uno o più handler. Un *router* raggruppa route e middleware montabili nell'app.

### Metodi, path e parametri

`app.METHOD(path, ...handlers)` registra handler per il metodo HTTP indicato, per esempio `app.get` o `app.post`; il metodo descrive la richiesta, mentre l'handler è la funzione che la elabora. Express confronta metodo e path della richiesta con la route; i segmenti `:nome` diventano parametri in `req.params`.

```js
app.get('/todos/:id', (req, res) => {
  res.json({ id: req.params.id });
});
```

Con `GET /todos/42`, `req.params.id` vale la stringa `'42'`: il parametro non viene convertito né validato automaticamente. La validazione dell'input è nel [capitolo sulle richieste](02-request-response-validation.md#input-della-richiesta); il significato dei metodi e il design delle risorse sono nel [contratto delle risorse](../../topics/api/16-restful-apis.md#contratto-http-delle-risorse).

### Router montati e precedenza

`express.Router()` crea un router che funziona come middleware. Montando `app.use('/todos', router)`, una route `router.get('/:id', ...)` gestisce `GET /todos/42`. Si possono montare router dentro altri router; per leggere nel router figlio i parametri del path del router padre serve `express.Router({ mergeParams: true })`.

Separare i router per risorsa rende riconoscibili le route e consente middleware locali con `router.use()`. Anche qui conta l'ordine: una route generica che risponde prima di una più specifica può renderla irraggiungibile. Registrare prima il caso specifico o far proseguire esplicitamente il controllo.

## Middleware

Un middleware ordinario ha firma `(req, res, next)`: può leggere o modificare richiesta e risposta, concludere la risposta o passare il controllo al livello successivo.

### Middleware applicativi, router e integrati

| Registrazione | Ambito e uso |
| --- | --- |
| `app.use(fn)` o `app.use('/prefix', fn)` | Tutte le richieste successive, oppure quelle sotto il prefisso, indipendentemente dal metodo. |
| `router.use(fn)` | Richieste che raggiungono quel router, dopo il suo mount e nell'ordine di registrazione. |
| `express.json()`, `express.urlencoded()` | Middleware integrati per analizzare body con media type compatibile e rendere disponibile `req.body`; configurare un limite di dimensione e applicarli dove servono. |
| `express.static(root)` | Middleware integrato per servire file dalla directory `root`; si può montare sotto un prefisso. |

Il parsing non sostituisce la validazione: `req.body` resta input non attendibile e può essere assente se la richiesta non ha un body o non corrisponde al media type atteso. Per parsing, limiti e confini di fiducia, vedi [input della richiesta](02-request-response-validation.md#input-della-richiesta).

### next e controllo del flusso

`next()` prosegue alla funzione successiva compatibile senza inviare una risposta. Dopo `res.send()` o `res.json()` non chiamare `next()` per proseguire la pipeline: si rischiano elaborazioni duplicate o un secondo tentativo di risposta. Anche chiamare `next()` più volte può eseguire più rami per la stessa richiesta. Un semplice `return` esce solo dalla funzione JavaScript: non invia una risposta né avanza la pipeline.

```js
app.use((req, _res, next) => {
  console.log(req.method, req.path);
  next();
});
```

Negli handler registrati con `app.METHOD()` o `router.METHOD()`, `next('route')` salta gli handler rimanenti della route corrente e cerca la prossima route compatibile: serve per un sub-stack condizionale, non equivale a `next()`. `next(err)` invece entra nella [pipeline degli errori](03-async-errors-lifecycle.md#pipeline-degli-errori); non è un modo per saltare una route.
