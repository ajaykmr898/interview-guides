# Asincronia errori e ciclo di vita Express

In Express, la gestione degli errori collega handler e middleware alla risposta HTTP (Hypertext Transfer Protocol); il ciclo di vita della richiesta comprende anche disconnessioni, timeout e risorse ancora attive. L'app deve distinguere un esito atteso da un guasto e concludere o interrompere il lavoro quando la risposta non può più essere inviata.

## Indice

- [Handler asincroni](#handler-asincroni)
- [Pipeline degli errori](#pipeline-degli-errori)
- [Ciclo di vita della richiesta](#ciclo-di-vita-della-richiesta)

## Handler asincroni

### Promise e propagazione degli errori

Express intercetta le eccezioni sincrone lanciate durante l'esecuzione di route e middleware. In Express 5, se un handler restituisce una Promise che viene rifiutata, Express passa automaticamente il motivo a `next(err)`; vale anche per una funzione `async`, che restituisce sempre una Promise. Un esito asincrono riuscito **non** invia da solo la risposta: l'handler deve chiamare `res.json()`, `res.send()` o un altro metodo di risposta.

```js
app.get('/reports/:id', async (req, res) => {
  const report = await loadReport(req.params.id);
  res.json(report);
});
```

Con `app` già creata come nel [primo capitolo](01-app-routing-middleware.md#applicazione-e-ciclo-richiesta-risposta), `loadReport(id)` rappresenta un service che restituisce una Promise: se rifiuta, Express 5 avvia la pipeline degli errori. Se invece si avvia una Promise senza restituirla né attenderla, Express non ne osserva il rifiuto. Le basi di [Promise, errori e cancellazione](../../topics/javascript/04-javascript-async-dom.md#errori-e-cancellazione) sono nel capitolo JavaScript.

### Express 4 e Express 5

| Origine dell'errore | Express 4 | Express 5 |
| --- | --- | --- |
| `throw` sincrono durante l'handler | Intercettato e inoltrato. | Intercettato e inoltrato. |
| Promise restituita o funzione `async` che rifiuta | Non inoltrata automaticamente: usare `try`/`catch` con `next(err)` oppure una catena `.catch(next)`. | Inoltrata automaticamente a `next(err)`. |
| Callback o evento asincrono esterno alla Promise restituita | Inoltrare esplicitamente l'errore con `next(err)` o gestire l'evento. | Stessa responsabilità: l'inoltro automatico non copre lavoro asincrono scollegato. |

In Express 4, per un handler `async` usare per esempio `try { ... } catch (err) { next(err); }`. In Express 5, non combinare inutilmente `next(err)` manuale con il rilancio dello stesso errore: si rischia una doppia propagazione. Una callback di filesystem o un timer che fallisce dopo il ritorno dell'handler richiede comunque gestione esplicita, anche in Express 5.

## Pipeline degli errori

`next(err)` salta gli handler ordinari rimanenti e passa al prossimo middleware di errore. Una route non trovata senza errori segue invece il normale percorso di fallthrough.

### Middleware di errore e handler predefinito

Un middleware di errore ha **quattro** argomenti `(err, req, res, next)` e va registrato dopo route e middleware ordinari. Può inviare una risposta oppure delegare con `next(err)`; se non fa nessuna delle due cose, la richiesta resta pendente. Più middleware di errore possono separare classificazione, logging e risposta, ma servono solo se aggiungono una responsabilità concreta.

Se nessun middleware personalizzato gestisce l'errore, l'handler predefinito di Express usa `err.status` o `err.statusCode` se è uno status 4xx/5xx, altrimenti `500`. In produzione invia una pagina HTML (HyperText Markup Language) con il messaggio dello status; fuori produzione include lo stack trace. Per un'API (Application Programming Interface), un handler applicativo evita di esporre dettagli interni e mantiene il formato d'errore concordato.

### 404, errori attesi e headersSent

Una richiesta senza route corrispondente è un `404`, **non** un'eccezione: collocare un normale middleware di fallback dopo le route e prima degli error handler. Anche una risorsa assente può meritare `404`, ma va decisa nel flusso applicativo. Errori attesi, come input non valido o risorsa assente, richiedono status e corpo coerenti; guasti inattesi richiedono una risposta `5xx` senza dettagli sensibili. Per la semantica generale vedi [errori nel contratto HTTP](../../topics/api/16-restful-apis.md#rappresentazioni-errori-e-collezioni).

```js
app.use((_req, res) => res.status(404).json({ error: 'Route assente' }));
app.use((err, _req, res, next) => {
  if (res.headersSent) return next(err);
  const code = err?.status;
  const status = Number.isInteger(code) && code >= 400 && code < 500 ? code : 500;
  res.status(status).json({ error: status === 500 ? 'Errore interno' : 'Richiesta non valida' });
});
```

L'esempio è un fallback generico: mappare separatamente gli errori di dominio quando il contratto richiede messaggi specifici. `res.headersSent` è vero dopo l'invio degli header; a quel punto non si può sostituire la risposta con un nuovo JSON (JavaScript Object Notation). Delegare con `next(err)` all'handler predefinito, che chiude la connessione se la risposta era già iniziata. Inoltrare lo stesso errore una sola volta.

## Ciclo di vita della richiesta

La fine della connessione non garantisce che le operazioni avviate dall'handler si fermino: cancellazione e cleanup dipendono dalle API usate.

### Abort, timeout e cleanup

Se il client si disconnette, evitare di continuare un lavoro costoso o tentare una risposta tardiva. Sul `res` di Node, l'evento `close` può indicare sia una risposta completata sia una connessione interrotta: controllare `res.writableFinished` per riconoscere la chiusura prematura. Non usare il solo `req.on('close')` come segnale di abbandono della risposta: l'evento della richiesta può indicare semplicemente che il body è stato letto.

Un timeout del server, del proxy o dell'operazione applicativa non annulla automaticamente tutto il lavoro a valle. Quando il client o una deadline non hanno più bisogno del risultato, usare un `AbortController` e passare il suo `signal` solo alle operazioni che supportano l'annullamento; rimuovere listener, cancellare timer e rilasciare connessioni o stream nel cleanup. Dopo un abort, non inviare una risposta se la connessione è già chiusa. L'annullamento locale non esegue rollback di una scrittura già confermata: il contratto deve considerare retry e idempotenza. Per le primitive di [AbortController e AbortSignal](../../topics/javascript/04-javascript-async-dom.md#errori-e-cancellazione) vedi JavaScript async.

### Stream e risposta parziale

Durante uno stream gli header e parte del body possono essere già stati inviati quando arriva un errore. `res.headersSent` impedisce di rispondere allora con un nuovo status o JSON; propagare l'errore e terminare il flusso/connessione secondo lo stato della risposta, pulendo la sorgente. Con stream Node, `pipeline()` coordina errori e chiusura meglio di una `pipe()` senza gestione esplicita. Per stream e backpressure vedi [Node e npm](../../topics/node-tooling/08-node-npm.md#runtime-node).
