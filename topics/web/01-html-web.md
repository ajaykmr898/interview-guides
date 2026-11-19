# HTML e Web

HTML (HyperText Markup Language) descrive la struttura e il significato dei contenuti di una pagina web; CSS ne definisce la presentazione.

## Indice

- [Struttura del documento](#struttura-del-documento)
- [Semantica HTML](#semantica-html)
- [Form e validazione](#form-e-validazione)
- [Accessibilità](#accessibilità)
- [SEO e metadati](#seo-e-metadati)
- [Browser e caricamento delle risorse](#browser-e-caricamento-delle-risorse)

## Struttura del documento

### Documento HTML

«HTML5» indica la generazione di funzionalità introdotta dopo HTML4, ma lo standard HTML attuale è mantenuto come **Living Standard**, non come versione numerata da selezionare nel documento.

```html
<!DOCTYPE html>
<html lang="it">
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>Profilo sviluppatore</title>
  </head>
  <body><main><h1>Profilo sviluppatore</h1></main></body>
</html>
```

- `<!DOCTYPE html>` richiede la modalità standards, evitando il rendering legacy *quirks mode*; non specifica una versione HTML.
- `<html lang="it">` dichiara la lingua principale, utile a pronuncia assistiva e indicizzazione. `<head>` contiene metadati e riferimenti a risorse; `<body>` contiene il contenuto del documento.
- `charset="utf-8"` dichiara la codifica; la configurazione del server deve essere coerente. Il meta viewport consente al layout mobile di usare la larghezza del dispositivo; non sostituisce un design responsive.

### Tag, elementi e attributi

Un **tag** è la sintassi di apertura o chiusura; un **elemento** comprende tag, contenuto e struttura risultante (gli elementi *void*, come `img`, non hanno tag di chiusura). Gli **attributi** nel tag di apertura configurano l'elemento: `id` lo identifica nel documento, `class` raggruppa elementi, `href` indica la destinazione di un link.

«Block» e «inline» descrivono il comportamento di rendering predefinito, modificabile con CSS, non due categorie universali della sintassi HTML. La validità del contenuto segue invece le categorie e i modelli di contenuto HTML: un `<p>`, per esempio, non può contenere un `<div>`. Per display e box model vedi [CSS e SCSS](02-css-scss.md#box-model-e-layout).

## Semantica HTML

Gli elementi **semantici** comunicano il ruolo del contenuto a browser e tecnologie assistive; `<div>` e `<span>` sono contenitori generici senza significato aggiuntivo. Scegliere l'elemento in base alla funzione, non all'aspetto che avrà con CSS.

| Elemento | Uso |
| --- | --- |
| `<header>` / `<footer>` | Introduzione e chiusura della pagina o di una sezione. |
| `<nav>` | Gruppo di link di navigazione principali o locali. |
| `<main>` | Contenuto dominante del documento; normalmente un solo `main` visibile. |
| `<article>` | Contenuto autonomo e riutilizzabile, per esempio una notizia. |
| `<section>` | Sezione tematica, normalmente con un heading; crea una sezione del documento. |
| `<aside>` | Contenuto indirettamente correlato, per esempio una nota laterale. |

```html
<header><nav aria-label="Navigazione principale"><a href="/">Home</a></nav></header>
<main>
  <article>
    <h1>Novità del prodotto</h1>
    <section><h2>Disponibilità</h2><p>Dettagli del rilascio.</p></section>
  </article>
</main>
<footer><p>Informazioni editoriali</p></footer>
```

Gli **heading** `<h1>`–`<h6>` formano una gerarchia leggibile: scegliere il livello per la struttura, non per la dimensione grafica. `section` è adatto a un'unità tematica con titolo; `div` serve a raggruppare elementi per layout o script quando non esiste un significato specifico. L'outline HTML automatico non è un sostituto affidabile di livelli di heading espliciti.

| Controllo | Scopo e comportamento |
| --- | --- |
| `<a href="...">` | Naviga verso una risorsa o una posizione; supporta apertura in nuova scheda e copia del link. |
| `<button type="button">` | Esegue un'azione nella pagina; offre comportamento da controllo e attivazione da tastiera. |

Non usare un link senza destinazione per simulare un pulsante né un pulsante per navigazione ordinaria. Dentro un form, un `<button>` senza `type` è per impostazione predefinita un pulsante di invio.

## Form e validazione

`<form>` raggruppa controlli e definisce `action` (destinazione) e `method` (metodo HTTP). `<label for="id">` dà un nome accessibile al controllo associato; `<input>` raccoglie valori brevi, `<textarea>` testo multilinea, `<select>` sceglie tra `<option>`, `<button>` attiva un'azione o invia il form.

| Attributo | Significato |
| --- | --- |
| `name` | Chiave con cui il valore del controllo viene inviato; senza `name` il controllo non contribuisce ai dati del form. |
| `value` | Valore del controllo o dell'opzione inviato con `name`; per `textarea` il valore iniziale è il testo tra i tag. |
| `type` | Tipo di input e relativa UI/validazione (`email`, `number`, `checkbox`, ecc.); su `button` distingue `submit`, `button` e `reset`. |
| `autocomplete` | Suggerisce al browser il tipo di dato da completare; non valida né protegge il valore. |
| `required` | Impedisce l'invio nativo se manca un valore valido, dove applicabile. |
| `pattern` | Richiede che l'intero valore di input testuale supportato corrisponda all'espressione regolare; non sostituisce la validazione server. |

```html
<form action="/iscrizioni" method="post">
  <label for="email">Email</label>
  <input id="email" name="email" type="email" autocomplete="email" required>
  <button type="submit">Iscriviti</button>
</form>
```

La **constraint validation** del browser controlla vincoli come `required`, `type` e `pattern` prima dell'invio normale; `checkValidity()` restituisce l'esito e `reportValidity()` mostra i messaggi nativi. Può essere aggirata dal client e non verifica regole di business: il server deve sempre validare i dati. Se il server o l'app rileva un errore, associare al controllo un messaggio specifico e aggiornare il suo stato, per esempio con `aria-describedby` e `aria-invalid="true"`.

```html
<label for="email-errore">Email</label>
<input id="email-errore" name="email" type="email"
       aria-invalid="true" aria-describedby="messaggio-email">
<p id="messaggio-email">Inserisci un indirizzo email valido.</p>
```

| Metodo del form | Trasporto e uso |
| --- | --- |
| `GET` | Serializza i dati nell'URL; adatto a ricerche e operazioni di lettura condivisibili. Evitare dati sensibili nella query. |
| `POST` | Invia i dati nel corpo della richiesta; adatto a modifiche di stato e dati non destinati all'URL. Non fornisce cifratura: serve HTTPS. |

## Accessibilità

Le **WCAG** del W3C organizzano criteri verificabili attorno a contenuti percepibili, utilizzabili, comprensibili e robusti. L'accessibilità si costruisce con semantica HTML, contrasto e feedback comprensibile, oltre che con verifiche automatiche e prove da tastiera e tecnologie assistive.

- Preferire elementi nativi: un `<button>` ha già ruolo, focus e comportamento da tastiera. **ARIA** aggiunge nome, ruolo o stato quando l'HTML non basta; non crea da sola comportamento né corregge un controllo non utilizzabile.
- Ogni funzione interattiva deve essere raggiungibile e attivabile da tastiera. Mantenere un ordine di focus coerente con il DOM e un indicatore visibile; evitare `tabindex` positivi e non rimuovere l'outline senza alternativa.
- Per `<img>`, `alt` descrive il contenuto utile nel contesto; `alt=""` marca un'immagine decorativa. Un'immagine che funge da link richiede un nome che descriva la destinazione o l'azione.
- I form richiedono label persistenti, istruzioni per vincoli non ovvi e messaggi d'errore associati al controllo. Dopo un invio non valido, portare il focus all'errore o a un riepilogo; non affidarsi solo a colore o placeholder.

## SEO e metadati

I metadati aiutano motori di ricerca e servizi di condivisione a interpretare la pagina, ma non garantiscono indicizzazione o posizione nei risultati. Il contenuto visibile e la semantica restano la fonte principale del significato.

- `<title>` identifica la pagina nella scheda e può contribuire al titolo del risultato; dovrebbe essere descrittivo e specifico. `<meta name="description">` propone una sintesi che il motore può usare o sostituire nello snippet.
- `<link rel="canonical" href="...">` segnala la URL preferita tra pagine duplicate o simili: è un indizio, non un comando. `<meta name="robots" content="noindex">` chiede ai crawler conformi di non indicizzare; deve poter essere letto dal crawler e non è un controllo d'accesso.
- **Open Graph** usa proprietà come `og:title`, `og:image` e `og:url` per le anteprime social. I **dati strutturati** descrivono entità e proprietà, spesso con JSON-LD e vocabolari Schema.org; possono abilitare presentazioni speciali solo se validi e supportati dal motore.

```html
<title>Guida HTML | Esempio</title>
<meta name="description" content="Concetti HTML per il ripasso tecnico.">
<link rel="canonical" href="https://example.com/guida-html">
<meta property="og:title" content="Guida HTML">
<meta property="og:type" content="website">
<meta property="og:url" content="https://example.com/guida-html">
<meta property="og:image" content="https://example.com/anteprima.png">
```

## Browser e caricamento delle risorse

### Parsing e rendering

Il browser analizza HTML e costruisce il **DOM** (nodi e relazioni del documento); analizza CSS e costruisce il **CSSOM** (regole di stile). Il **BOM** raccoglie interfacce del contesto browser, come `window`, `location`, `history` e `navigator`; non è una singola struttura ad albero equivalente al DOM.

DOM e CSSOM contribuiscono al **render tree** degli elementi da disegnare. Il **layout/reflow** calcola dimensioni e posizioni, il **paint** produce i pixel, il **compositing** assembla eventuali layer. Modifiche di stile o DOM possono innescare solo alcune fasi o anche nuovo layout; misurare prima di ottimizzare. La manipolazione dettagliata del DOM e gli eventi sono nel capitolo [JavaScript async e DOM](../javascript/04-javascript-async-dom.md#dom-e-rendering).

| Script | Caricamento ed esecuzione |
| --- | --- |
| Classico senza attributi | Il parser HTML si ferma per caricamento ed esecuzione. |
| `defer` su script classico esterno | Scarica durante il parsing, esegue dopo il parsing nell'ordine del documento e prima di `DOMContentLoaded`. |
| `async` | Scarica durante il parsing, esegue appena pronto; ordine non garantito e il parser può fermarsi durante l'esecuzione. |
| `type="module"` | Usa moduli ES; senza `async` è differito per impostazione predefinita. Con `async` esegue quando il grafo è pronto. |

`defer` su script inline classico non ha effetto; su un modulo è superfluo. Usare `async` per script indipendenti, `defer` per script classici che richiedono DOM già analizzato o ordine di esecuzione.

### Web APIs e storage

- **Fetch** invia richieste e restituisce una Promise di `Response`; una risposta HTTP `404` o `500` non rigetta la Promise: controllare `response.ok`/`status`. Le API e l'asincronia sono approfondite in [Web API e fetch](../javascript/04-javascript-async-dom.md#web-api-e-fetch).
- **History** gestisce la cronologia della scheda: `pushState()` aggiunge una voce, `replaceState()` sostituisce quella corrente senza caricare una nuova pagina; gli URL passati devono essere della stessa origine. **URL** e `URLSearchParams` analizzano e modificano URL e query senza concatenazioni manuali.
- **IntersectionObserver** segnala quando un elemento interseca una viewport o un antenato, utile per lazy loading; non misura automaticamente la visibilità effettiva se un altro elemento lo copre. **MutationObserver** osserva modifiche a nodi, attributi o testo del DOM. **ResizeObserver** notifica variazioni di dimensione di un elemento, utili per componenti che dipendono dalla propria misura.

| Meccanismo | Durata e ambito | Invio HTTP / accesso JS |
| --- | --- | --- |
| Cookie | Scadenza o sessione; dominio e path secondo gli attributi. | Inviato automaticamente alle richieste applicabili; `HttpOnly` ne impedisce la lettura da JavaScript. |
| `localStorage` | Per origine; persiste oltre la chiusura del browser, salvo cancellazione o policy. | Non inviato automaticamente; accessibile agli script della stessa origine. |
| `sessionStorage` | Per origine e scheda; dura per la sessione della scheda, anche dopo reload. | Non inviato automaticamente; accessibile agli script della relativa scheda e origine. |

Web Storage memorizza coppie stringa/valore tramite API sincrone: evitare dati voluminosi e non considerarlo un deposito sicuro per segreti accessibili a JavaScript. Per cookie di sessione valutare `Secure`, `HttpOnly` e `SameSite` secondo il contesto; `SameSite=None` richiede `Secure` nei browser moderni.

### HTTP, HTTPS e cache

**DNS** risolve un nome di dominio in indirizzi utili alla connessione; poi il client invia una **richiesta HTTP** con metodo, URL, header ed eventuale corpo. Il server restituisce una **risposta** con status code, header ed eventuale corpo. HTTP è stateless: cookie e applicazioni possono costruire una sessione sopra richieste distinte. **HTTPS** trasporta HTTP con TLS, che autentica il server tramite certificato e protegge confidenzialità e integrità del traffico; non rende affidabile l'applicazione in sé.

| Metodo | Semantica essenziale |
| --- | --- |
| `GET` / `HEAD` | Lettura; `HEAD` richiede i metadati senza corpo di risposta. Sicuri e idempotenti secondo la semantica HTTP. |
| `POST` | Invia dati per un'operazione; non è idempotente per definizione. |
| `PUT` / `DELETE` | Sostituisce la rappresentazione della risorsa / ne richiede la rimozione; idempotenti per semantica. |
| `PATCH` | Applica una modifica parziale; l'idempotenza dipende dall'operazione. |

Le classi di **status code** sono `2xx` successo (`200`, `201`), `3xx` redirezione o validazione (`301`, `304`), `4xx` errore del client (`400`, `401`, `403`, `404`) e `5xx` errore del server (`500`, `503`). `401` riguarda autenticazione mancante o non valida; `403` indica che il server rifiuta l'accesso. Gli **header** trasportano metadati e direttive, per esempio `Content-Type`, `Cache-Control` e `Authorization`.

La **cache HTTP** può riusare risposte fresche secondo `Cache-Control`. `no-cache` consente la memorizzazione ma richiede rivalidazione prima del riuso; `no-store` chiede di non memorizzare. Un `ETag` identifica una versione della risorsa: il client può inviare `If-None-Match` e ricevere `304 Not Modified` senza ritrasmettere il corpo se la versione coincide.

**HTTP/2** multiplexa più stream su una connessione TCP e comprime gli header; **HTTP/3** mantiene le semantiche HTTP ma usa QUIC sopra UDP, evitando il blocco tra stream dovuto alla ritrasmissione TCP. Non cambia il significato di metodi o status code.

### Isolamento e sicurezza

La **same-origin policy** limita l'accesso di uno script alle risorse di un'altra origine, definita da schema, host e porta. **CORS** consente al server di autorizzare la lettura cross-origin nel browser tramite header come `Access-Control-Allow-Origin`; non è un meccanismo di autenticazione. Per metodi o header non semplici, il browser può inviare un **preflight** `OPTIONS` per verificare i permessi prima della richiesta effettiva. Una richiesta semplice può comunque arrivare al server anche se la risposta non è leggibile dallo script.

- **XSS**: input non attendibile diventa codice eseguibile nella pagina. Usare output encoding adatto al contesto e API testuali; se serve HTML, sanitizzarlo. **CSP** limita origini e tipi di codice caricabile, preferibilmente tramite header con nonce o hash: è una difesa aggiuntiva, non sostituisce l'encoding.
- **CSRF**: un sito esterno induce il browser di una vittima autenticata a inviare una richiesta con credenziali ambientali. Per operazioni che cambiano stato, verificare token anti-CSRF o origine della richiesta e usare `SameSite` come difesa aggiuntiva; CORS non basta a prevenire CSRF.
- **Authentication** verifica l'identità; **authorization** decide quali operazioni o risorse sono permesse. Il server deve applicare i controlli di autorizzazione a ogni richiesta rilevante, anche se l'interfaccia nasconde i comandi non consentiti.
