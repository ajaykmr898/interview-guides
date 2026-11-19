# API RESTful

Un'**API (Application Programming Interface) RESTful** applica i vincoli di **REST (Representational State Transfer)** alla progettazione di risorse accessibili tramite **HTTP (Hypertext Transfer Protocol)**. Identificatori, rappresentazioni e messaggi descrivono le interazioni; usare endpoint HTTP non basta, da solo, a rispettare tutti i vincoli REST.

## Indice

- [Modello REST e risorse](#modello-rest-e-risorse)
- [Contratto HTTP delle risorse](#contratto-http-delle-risorse)
- [Rappresentazioni, errori e collezioni](#rappresentazioni-errori-e-collezioni)
- [Cache, concorrenza ed evoluzione](#cache-concorrenza-ed-evoluzione)
- [Affidabilità delle operazioni](#affidabilità-delle-operazioni)
- [Sicurezza e verifica del contratto](#sicurezza-e-verifica-del-contratto)

## Modello REST e risorse

### Vincoli architetturali REST e differenza da API HTTP

REST è uno stile architetturale, non un formato di payload o un set di URL convenzionali:

- **Client-server:** separa responsabilità di interfaccia e dati.
- **Stateless:** ogni richiesta contiene il contesto necessario a interpretarla; il server può conservare lo stato delle risorse, senza dipendere dallo stato di sessione del client per capire la richiesta.
- **Cacheable:** le risposte dichiarano quando possono essere riusate; **layered system** permette intermediari senza esporre l'intera topologia al client.
- **Uniform interface:** vincolo sull'interazione, articolato nei quattro punti sotto. **Code-on-demand** è opzionale: il server può trasferire codice eseguibile al client.

Una generica API HTTP può usare metodi e corpi strutturati senza soddisfare la uniform interface. Una **RPC (Remote Procedure Call)** espone invece operazioni come chiamate a procedure; REST centra il contratto sulle risorse. La scelta dipende dal dominio e dai client, non dal solo stile dei nomi degli endpoint.

### Uniform interface: risorse, rappresentazioni, messaggi autodescrittivi e HATEOAS

| Vincolo della uniform interface | Implicazione nel contratto |
| --- | --- |
| Identificazione delle risorse | Una **URI (Uniform Resource Identifier)** identifica la risorsa indipendentemente dalla rappresentazione restituita. |
| Manipolazione tramite rappresentazioni | Il client invia o riceve rappresentazioni e metadati per chiedere cambiamenti, senza accedere direttamente allo storage. |
| Messaggi autodescrittivi | Metodo, status, header e media type esplicitano come interpretare il messaggio; i dettagli del dominio restano nel contratto della rappresentazione. |
| **HATEOAS (Hypermedia as the Engine of Application State)** | Link e azioni con relazioni esplicite indicano le transizioni disponibili nello stato corrente, anziché imporre al client ogni URI successiva. |

Progettare URI stabili per risorse e relazioni, per esempio `/orders/42` e `/orders/42/items`; evitare di codificare ogni operazione nel percorso. I link possono rendere navigabile il workflow, ma un semplice elenco di URI senza semantica delle azioni non realizza da solo HATEOAS. Molte API dette “RESTful” adottano solo parte di questi vincoli: descrivere il compromesso con precisione.

Esempio di rappresentazione **JSON (JavaScript Object Notation)** con un formato di link applicativo, non standardizzato da REST:

```json
{
  "id": 42,
  "links": [
    { "rel": "self", "href": "/orders/42", "method": "GET" },
    { "rel": "cancel", "href": "/orders/42/cancellation", "method": "POST" }
  ]
}
```

## Contratto HTTP delle risorse

### Metodi, safety e idempotenza

Un metodo **safe** non richiede un cambiamento di stato; **idempotente** significa che ripetere la stessa richiesta ha lo stesso effetto previsto sul server, anche se le risposte possono differire. Logging e metriche non rendono automaticamente insicuro `GET`. Per la [semantica HTTP generale](../web/01-html-web.md#http-https-e-cache) vedi il capitolo Web.

| Metodo | Risorsa di esempio | Semantica rilevante |
| --- | --- | --- |
| `GET` | `/orders/42` | Legge una rappresentazione; safe e idempotente. |
| `POST` | `/orders` | Il server elabora il payload, spesso creando una risorsa; non idempotente per definizione. |
| `PUT` | `/orders/42` | Crea o sostituisce lo stato della risorsa target, se consentito; idempotente. |
| `PATCH` | `/orders/42` | Applica un documento di modifica; l'idempotenza dipende dalla modifica. |
| `DELETE` | `/orders/42` | Richiede la rimozione; idempotente nell'effetto previsto, anche se un secondo tentativo può avere status diverso. |

### Creazione, aggiornamento e cancellazione

Con `POST` su una collezione il server può assegnare l'identificatore; `PUT` indirizza la risorsa da creare o sostituire e non equivale a un aggiornamento parziale. `PATCH` richiede una semantica del documento di modifica concordata: impostare un valore e incrementarlo hanno proprietà di retry diverse. `DELETE` non promette cancellazione fisica immediata: il contratto può prevedere soft delete o elaborazione asincrona.

Per una creazione completata, la risposta tipica identifica la nuova risorsa; `Location` può contenere una URI relativa:

```http
HTTP/1.1 201 Created
Location: /orders/42
Content-Type: application/json

{"id":42,"status":"open"}
```

### Status code e header di risposta

| Esito nel contratto | Risposta tipica |
| --- | --- |
| Lettura o modifica con corpo; creazione; successo senza corpo | `200 OK`; `201 Created` con `Location` per la risorsa creata; `204 No Content`. |
| Operazione accettata ma non completata | `202 Accepted`: non garantisce l'esito finale. |
| Input non valido; risorsa assente; conflitto di dominio | `400`/`422` secondo il tipo di errore; `404`; `409`. |
| Precondizione HTTP non soddisfatta | `412 Precondition Failed`, distinta da `409 Conflict`. |
| Guasto del server o indisponibilità temporanea | `500` o `503`; non mascherare questi errori come `200`. |

Gli status descrivono l'esito HTTP, mentre il corpo spiega il caso applicativo. Se il metodo è noto ma non permesso per quella risorsa, `405 Method Not Allowed` richiede l'header `Allow`; non usare `404` per nascondere indiscriminatamente errori di contratto.

## Rappresentazioni, errori e collezioni

### Media type e content negotiation

La rappresentazione non è la risorsa: la stessa URI può offrire formati diversi. `Content-Type` dichiara il formato del corpo inviato; `Accept` indica i formati di risposta accettabili dal client. JSON è comune, non obbligatorio per REST. Un formato di richiesta non supportato può ricevere `415`; se non si può produrre una rappresentazione accettabile, `406` è una possibile risposta. Quando la scelta dipende da `Accept`, `Vary: Accept` evita riusi errati nelle cache.

### Validazione e Problem Details

Validare parametri, query e corpo al confine, distinguendo sintassi, regole del dominio e conflitti con lo stato corrente. Restituire status coerenti e un errore leggibile dalla macchina, senza esporre dettagli interni. **Problem Details** fornisce `application/problem+json` con un `type` identificativo e campi come `title`, `detail` e `status`; non sostituisce lo status HTTP, e `status` nel corpo è facoltativo e solo informativo.

```http
HTTP/1.1 422 Unprocessable Content
Content-Type: application/problem+json

{
  "type": "https://api.example.test/problems/invalid-quantity",
  "title": "Quantità non valida",
  "status": 422
}
```

Usare `422` quando il media type e la sintassi sono validi ma l'istruzione non è elaborabile; `409` riguarda invece un conflitto con lo stato della risorsa. Problem Details non definisce gli errori di campo dell'applicazione: servono estensioni concordate, e migrare consumer che attendono un formato precedente. Il formato degli errori deve restare coerente anche fuori dal percorso felice.

### Filtri, ordinamento e paginazione

Una collezione può esporre filtri, ordinamento e dimensione della pagina tramite query; nomi dei parametri e limiti sono parte del contratto, non uno standard REST. Un ordinamento stabile con un criterio di spareggio evita ambiguità, ma inserimenti e rimozioni possono comunque spostare i risultati, soprattutto con offset.

| Strategia | Utile quando | Limite |
| --- | --- | --- |
| Offset | Accesso a pagine numerate e dataset relativamente stabili. | Inserimenti/rimozioni possono spostare risultati; offset elevati possono costare. |
| Cursor opaco | Scorrimento progressivo di dataset grandi o mutevoli. | Non offre accesso diretto alla pagina N; il cursor deve incorporare o riferire ordinamento e filtri. |

Restituire un link o cursor per la pagina successiva; un conteggio totale può essere costoso e non è sempre necessario.

## Cache, concorrenza ed evoluzione

### Cache e richieste condizionali

`Cache-Control` definisce la riusabilità della risposta; `ETag` è un validatore della rappresentazione. Con `If-None-Match`, una lettura può ottenere `304 Not Modified` senza ritrasmettere il corpo. Separare risposte pubbliche da dati personali (`private` o `no-store` secondo la necessità) e dichiarare `Vary` quando header della richiesta selezionano rappresentazioni diverse. I dettagli della [cache HTTP](../web/01-html-web.md#http-https-e-cache) sono nel capitolo Web.

### Concorrenza con ETag

Per evitare *lost update*, il client legge un `ETag` e invia `If-Match` quando modifica la risorsa. Il server confronta un validator forte: se la rappresentazione è cambiata, non applica la modifica e può rispondere `412`; se richiede sempre una precondizione, può usare `428 Precondition Required`. `409` resta per conflitti con lo stato del dominio, non per una precondizione HTTP fallita.

```http
PUT /orders/42 HTTP/1.1
Host: api.example.test
If-Match: "v7"
Content-Type: application/json
Content-Length: 30

{"id":42,"status":"confirmed"}
```

### Versioning e compatibilità

Un cambio che rimuove campi, ne altera il significato o modifica errori e precondizioni può rompere i consumer. Aggiungere campi è spesso più semplice, ma non sempre compatibile con client rigidi. Versioni nella URI, nel media type o in header hanno trade-off di discovery e gestione della cache: scegliere una politica, documentare la deprecazione e verificare i consumer prima di eliminare il vecchio contratto.

## Affidabilità delle operazioni

### Retry, idempotency key e operazioni asincrone

Un retry automatico è più sicuro per metodi idempotenti; ripetere `POST` dopo un timeout può duplicare l'effetto. Il contratto può prevedere una **idempotency key** per richieste non idempotenti: stessa chiave e stesso input, entro uno scope e una finestra dichiarati, recuperano l'esito della prima operazione; una chiave riusata con input diverso va rifiutata. È una garanzia applicativa, non una proprietà automatica di `POST`.

Per lavori lunghi, `202 Accepted` conferma solo l'accettazione. Esporre una risorsa di stato consultabile e documentare esiti finali, errori e durata di conservazione; non presentare `202` come successo definitivo.

## Sicurezza e verifica del contratto

### Autenticazione, autorizzazione e rate limiting

Il contratto specifica come il client presenta le credenziali e quali operazioni richiedono permessi. Verificare l'autorizzazione per ogni risorsa e operazione, non solo per la route: `401` indica credenziali mancanti o non valide e richiede `WWW-Authenticate`; `403` rifiuta una richiesta compresa, mentre `404` può celare l'esistenza di una risorsa vietata. Le regole di [CORS (Cross-Origin Resource Sharing) e sicurezza browser](../web/01-html-web.md#isolamento-e-sicurezza) non sostituiscono questi controlli.

Definire nel contratto scope e quota del rate limiting; `429 Too Many Requests` può includere `Retry-After` per suggerire quando riprovare. Per middleware e dettagli di implementazione vedi [Node e npm](../node-tooling/08-node-npm.md#backend-http-ed-express).

### OpenAPI e contract test

**OpenAPI** descrive operazioni HTTP, parametri, rappresentazioni, risposte e requisiti di sicurezza in un formato leggibile da strumenti e persone; descrivere un endpoint non ne prova il comportamento né lo rende automaticamente RESTful. I **contract test** verificano che provider e consumer rispettino le richieste e risposte concordate, inclusi errori e compatibilità; completano, senza sostituire, i test del flusso applicativo. Per i [contratti e i tipi di test](../engineering/15-architecture-devops.md#api-e-contratti) vedi il capitolo Engineering.
