# Richieste risposte e validazione Express

In Express, `req` espone i dati della richiesta HTTP (Hypertext Transfer Protocol) e `res` costruisce la risposta. Parametri, query e body attraversano un confine di fiducia: occorre interpretarli e validarli prima della logica applicativa.

## Indice

- [Input della richiesta](#input-della-richiesta)
- [Validazione](#validazione)
- [Costruzione della risposta](#costruzione-della-risposta)

## Input della richiesta

### Params, query e body

| Fonte | Provenienza e tipo effettivo |
| --- | --- |
| `req.params` | Segmenti del path definiti dalla route, per esempio `:id`; i parametri nominati sono stringhe decodificate, mentre i wildcard di Express 5 possono produrre array di segmenti. |
| `req.query` | Query string analizzata dal parser configurato; in Express 5 il parser predefinito è `simple`. I valori possono essere stringhe o array per chiavi ripetute; un parser diverso può produrre altre forme. |
| `req.body` | Corpo analizzato da un middleware adatto al media type; senza parser applicabile è `undefined`. La forma dipende dal parser e dal payload: per esempio oggetto o array con `express.json()`. |

Nessuno di questi dati viene convertito automaticamente nel tipo del dominio: `req.params.id === '42'` è una stringa, anche se l'ID è numerico nel database. Le annotazioni TypeScript descrivono un'aspettativa statica, ma non validano il payload a runtime. Il matching delle route è nel [capitolo su app e middleware](01-app-routing-middleware.md#router-e-route).

### Parsing e confini di fiducia

`express.json()` e `express.urlencoded()` restituiscono middleware che leggono rispettivamente JSON (JavaScript Object Notation) e dati `application/x-www-form-urlencoded` con `Content-Type` compatibile. Registrarli prima degli handler che usano `req.body`, globalmente o sulla sola route interessata. `express.json()` accetta di default oggetti e array JSON; `express.urlencoded()` produce un oggetto di valori form, con struttura dipendente dall'opzione `extended`.

Se il body è assente o il media type non corrisponde, il parser non popola `req.body`; JSON malformato passa alla pipeline degli errori (tipicamente `400`), mentre un body oltre `limit` produce un errore di dimensione (tipicamente `413`). Un media type non supportato va rifiutato esplicitamente se il contratto lo richiede: il solo parser non invia automaticamente `415`. Impostare limiti proporzionati al caso d'uso e non leggere proprietà o chiamare metodi su `req.body` prima di verificarne la forma. L'esempio nella sezione seguente mostra un parser limitato alla singola route. Per la semantica di media type e risposte d'errore vedi [API RESTful](../../topics/api/16-restful-apis.md#rappresentazioni-errori-e-collezioni).

## Validazione

La validazione al confine traduce input non attendibile in valori utilizzabili dal dominio; Express non include uno schema validator. Può essere implementata con controlli espliciti o con una libreria, purché le regole siano applicate sul server.

### Schemi e normalizzazione

Verificare presenza, forma, tipo e vincoli dei campi prima di invocare service o repository; fare lo stesso per parametri di route e query. Normalizzare solo con una regola dichiarata, per esempio `trim()` su un titolo dopo aver verificato che sia una stringa. Non convertire silenziosamente una stringa come `'false'` in booleano né affidarsi a un tipo TypeScript o al solo successo del parsing.

```js
app.post('/greet', express.json({ limit: '16kb' }), (req, res) => {
  const name = req.body?.name;
  if (typeof name !== 'string' || !name.trim()) {
    return res.status(400).json({ error: 'name non valido' });
  }
  return res.status(200).json({ message: `Ciao, ${name.trim()}` });
});
```

L'esempio assume `app` ed `express` creati come nel [primo capitolo](01-app-routing-middleware.md#express-e-server-http). Il `return` interrompe l'handler dopo l'invio, evitando una seconda risposta; il valore restituito non è il payload usato da Express.

### Errori di input

Mappare errori sintattici del parser, campi mancanti o tipi e vincoli non validi a risposte client coerenti, con status e corpo stabili. `400` è adatto all'input non valido dell'esempio; un contratto può distinguere `422` per contenuto sintatticamente valido ma non elaborabile. Non esporre stack trace, payload grezzi o dettagli interni. Gli errori del parser arrivano ai [middleware di errore](03-async-errors-lifecycle.md#pipeline-degli-errori); i criteri generali per il formato sono nel [contratto HTTP](../../topics/api/16-restful-apis.md#rappresentazioni-errori-e-collezioni).

## Costruzione della risposta

`res` estende la risposta HTTP di Node.js: status e header si impostano prima dell'invio, poi un metodo conclude la risposta o si scrive uno stream.

### Status, header e JSON

| API | Effetto e ritorno |
| --- | --- |
| `res.status(code)` | Imposta lo status numerico e restituisce `res` per il chaining; da sola non invia nulla. |
| `res.set(field, value)` | Imposta un header e restituisce `res`; usarla prima dell'invio. |
| `res.json(value)` | Serializza con `JSON.stringify`, imposta il media type JSON, invia e conclude la risposta; restituisce `res`. |
| `res.send(value)` | Invia e conclude la risposta, restituendo `res`; una stringa è trattata come HTML, oggetti e array come JSON, un `Buffer` come dati binari. |
| `res.end()` | Conclude la risposta, di norma senza corpo; restituisce `res`, senza scegliere automaticamente una rappresentazione per un valore applicativo. |

Senza uno status esplicito, una risposta ordinaria parte da `200`: per una creazione usare per esempio `res.status(201).json(todo)`, per un successo senza corpo `res.status(204).end()`. `res.status(400)` da solo lascia la richiesta aperta. In Express 5 le vecchie firme `res.send(status)` e `res.json(body, status)` non sono supportate: impostare lo status prima dell'invio. Dopo `send`, `json` o `end` non tentare un secondo invio; `res.headersSent` indica se gli header sono già partiti. Per la scelta degli status vedi il [contratto delle risorse](../../topics/api/16-restful-apis.md#contratto-http-delle-risorse).

### Redirect, download e streaming

- `res.redirect(path)` invia un redirect `302` per default; si può specificare uno status. Usare destinazioni consentite, perché un URL derivato dall'input può creare un open redirect.
- `res.download(path, filename?, callback?)` invia un file come attachment. Verificare il path e limitarlo a una directory prevista; con callback, gestire anche errori dopo l'invio parziale, controllando `res.headersSent`.
- Per contenuti grandi o prodotti gradualmente, uno stream scrive sulla risposta senza costruire tutto il body in memoria. Dopo l'invio degli header non si può sostituire un flusso fallito con un nuovo JSON: gestire errori, disconnessione e cleanup come nel [ciclo di vita della richiesta](03-async-errors-lifecycle.md#ciclo-di-vita-della-richiesta).
