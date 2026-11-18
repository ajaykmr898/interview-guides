# JavaScript async e DOM

L'asincronia in JavaScript permette di avviare operazioni e gestirne l'esito in seguito, senza bloccare necessariamente il codice chiamante. Nel browser il DOM (Document Object Model) rappresenta il documento come albero di nodi modificabile tramite Web API (Application Programming Interface, interfacce di programmazione del browser); event loop, timer, eventi e Fetch sono servizi del runtime, non della sintassi ECMAScript.

## Indice

- [Event loop e task](#event-loop-e-task)
- [Promise e async await](#promise-e-async-await)
- [Errori e cancellazione](#errori-e-cancellazione)
- [DOM e rendering](#dom-e-rendering)
- [Eventi e delega](#eventi-e-delega)
- [Web API e fetch](#web-api-e-fetch)

## Event loop e task

### Esecuzione sincrona, asincrona e callback

Il codice *sincrono* procede una chiamata alla volta sul call stack; una chiamata lunga blocca il thread che la esegue. Un'operazione *asincrona* consegna il risultato più tardi tramite callback, Promise o evento: l'attesa non blocca il call stack, ma il codice JavaScript eseguito al termine resta sincrono sul proprio thread.

La *concurrency* consiste nel far avanzare più operazioni in intervalli sovrapposti; il *parallelism* consiste nell'eseguirle contemporaneamente su risorse distinte. Il browser può svolgere operazioni di rete fuori dal thread principale e i Web Worker possono eseguire JavaScript su altri thread; usare `async` non rende parallelo un calcolo sul thread principale.

Un *callback* è una funzione passata per gestire un risultato futuro. Callback annidati per operazioni dipendenti possono creare *callback hell*: codice difficile da leggere, con gestione degli errori e controllo del flusso frammentati. Promise e `async`/`await` rendono esplicita la sequenza, ma non eliminano automaticamente i problemi di concorrenza.

### Event loop, code e rendering

L'*event loop* del browser coordina l'esecuzione di task, microtask e opportunità di rendering. Un *task* (spesso chiamato macrotask) può provenire da uno script iniziale, un evento o un timer; viene eseguito fino al termine. Quando il call stack si libera, il browser svuota la *microtask queue*, usata per gli handler delle Promise e `queueMicrotask()`, prima della task successiva. Tra un'iterazione e l'altra può aggiornare layout e disegno: non è garantito un frame dopo ogni task.

```js
console.log("A");
setTimeout(() => console.log("D"), 0);
Promise.resolve().then(() => console.log("C"));
queueMicrotask(() => console.log("C2"));
console.log("B");
// A, B, C, C2, D
```

- `setTimeout(fn, delay)` programma una task **non prima** del ritardo richiesto; `0` non significa esecuzione immediata. `clearTimeout(id)` la annulla se non è già partita.
- `setInterval(fn, delay)` programma chiamate periodiche finché non si usa `clearInterval(id)`; cadenza e tempo effettivo dipendono dal runtime e dal lavoro sul thread.
- `queueMicrotask(fn)` programma una microtask dopo il codice corrente. Una catena che continua ad aggiungere microtask può ritardare task, input e rendering.

**Best practice:** lasciare brevi i callback sul thread principale. Per lavoro CPU intenso usare, se appropriato, un Web Worker; per aggiornamenti visivi coordinati con il frame, `requestAnimationFrame()`.

## Promise e async await

### Stati, thenable e chaining

Una *Promise* rappresenta un risultato asincrono: è inizialmente `pending` e diventa `fulfilled` con un valore o `rejected` con una ragione. Una volta conclusa non cambia stato. Un *thenable* è un oggetto con un metodo `then` chiamabile: la risoluzione di una Promise ne assimila l'esito. “Resolved” non significa necessariamente già `fulfilled`, perché può ancora attendere un thenable.

`.then(onFulfilled, onRejected)` restituisce una **nuova** Promise. Il valore restituito dall'handler diventa il risultato del passo successivo; una Promise o un thenable restituiti vengono attesi, mentre un'eccezione causa rejection. `.catch(onRejected)` gestisce una rejection nella catena; `.finally(onFinally)` esegue cleanup indipendentemente dall'esito e normalmente lascia passare valore o errore, salvo che l'handler fallisca. Un errore non gestito si propaga fino alla fine della catena.

Per una richiesta HTTP (Hypertext Transfer Protocol), lo status va controllato esplicitamente prima di leggere il risultato:

```js
fetch("/api/user")
  .then((response) => {
    if (!response.ok) throw new Error(`HTTP ${response.status}`);
    return response.json();
  })
  .then((user) => console.log(user.name))
  .catch((error) => console.error(error))
  .finally(() => console.log("richiesta conclusa"));
```

### Combinatori di Promise

I combinatori ricevono un iterabile di valori o Promise e restituiscono una nuova Promise: coordinano gli esiti, ma non cancellano le operazioni sottostanti. Gli array restituiti da `all` e `allSettled` seguono l'ordine degli input, non quello di completamento.

| Metodo | Quando termina | Uso e limite principale |
| --- | --- | --- |
| `Promise.all()` | Tutte fulfilled, oppure alla prima rejection. | Risultati tutti necessari; le altre operazioni proseguono dopo una rejection. |
| `Promise.allSettled()` | Tutte concluse, con oggetti `{ status, value }` o `{ status, reason }`. | Raccoglie successi e fallimenti senza scartarli. |
| `Promise.race()` | Al primo esito, fulfilled o rejected. | Sceglie il primo risultato; non interrompe gli altri partecipanti. |
| `Promise.any()` | Alla prima fulfillment, oppure rejected con `AggregateError` se tutte falliscono. | Usa il primo successo e ignora i fallimenti intermedi. |

Con input vuoto, `all` e `allSettled` producono `[]`, `any` fallisce con `AggregateError` e `race` rimane pending.

L'esempio legge JSON (JavaScript Object Notation) da due endpoint indipendenti e verifica entrambi gli status:

```js
async function loadBoth() {
  const getJSON = async (url) => {
    const response = await fetch(url);
    if (!response.ok) throw new Error(`HTTP ${response.status}`);
    return response.json();
  };
  const [user, settings] = await Promise.all([
    getJSON("/api/user"), getJSON("/api/settings"),
  ]);
  return { user, settings };
}
```

### `async`/`await` e race conditions

Una funzione `async` restituisce sempre una Promise. `await` sospende **quella funzione** fino all'esito del valore atteso e riprende in una microtask; non blocca il thread. `try`/`catch` intercetta eccezioni sincrone e rejection delle Promise effettivamente attese; un `finally` è adatto al cleanup. Se due operazioni sono indipendenti, avviarle entrambe prima dell'`await` evita attese sequenziali inutili.

```js
async function loadName() {
  try {
    const response = await fetch("/api/user");
    if (!response.ok) throw new Error(`HTTP ${response.status}`);
    const user = await response.json();
    return user.name;
  } catch (error) {
    console.error(error);
    throw error;
  }
}
```

Una *race condition* nasce quando il risultato dipende dall'ordine non garantito di completamento: per esempio, una ricerca vecchia può sovrascrivere una più recente. Usare un identificatore di richiesta per ignorare risultati obsoleti, oppure annullare la richiesta precedente quando l'API lo consente. `Promise.all()` coordina gli esiti, ma non rende atomiche modifiche a stato condiviso.

### Promise, Observable e Subscription

| Concetto | Cosa rappresenta | Gestione dell'esecuzione |
| --- | --- | --- |
| Promise | Un esito futuro: un valore o un errore. | Non offre cancellazione generica; gli handler osservano l'esito. |
| Observable di RxJS | Una sorgente che può emettere più valori nel tempo, poi completare o fallire. | `subscribe()` avvia un'esecuzione per le sorgenti *cold*; esistono anche sorgenti *hot* condivise. |
| Subscription di RxJS | La risorsa restituita da `subscribe()`. | `unsubscribe()` interrompe l'esecuzione e attiva il teardown previsto dalla sorgente. |

Observable e Subscription di RxJS sono astrazioni di libreria, non primitive ECMAScript. `unsubscribe()` annulla il lavoro sottostante solo se la sorgente ha collegato il proprio teardown a quel lavoro. Per operatori, multicasting e gestione degli errori vedere [RxJS](../rxjs/10-rxjs.md#observable-e-subscription).

## Errori e cancellazione

### Errori asincroni e cleanup

Un `throw` dentro un handler di Promise produce una rejection nella Promise restituita; un `try`/`catch` esterno a una chiamata asincrona non intercetta un errore che arriva in seguito. Occorre `await` dentro il `try` oppure un `.catch()` sulla catena. Una Promise ignorata può produrre una rejection non gestita: gestire gli errori nel punto che conosce il contesto o propagarli esplicitamente al chiamante.

### AbortController e AbortSignal

`AbortController` crea un `AbortSignal` passato a un'API che supporta l'annullamento; `controller.abort()` segnala la richiesta di stop. Con Fetch interrompe la richiesta o la lettura del body e normalmente fa rifiutare l'operazione con `AbortError`. Una Promise generica non ha un metodo di cancellazione: interrompere l'attesa di un risultato non interrompe automaticamente il lavoro che lo produce. Un segnale già annullato non si riusa per una nuova richiesta.

Per aggiornamenti che possono sovrapporsi, annullare Fetch precedente riduce lavoro inutile; se il risultato è già stato applicato, serve comunque un controllo di pertinenza nello stato dell'applicazione. Il segnale è utilizzabile anche per il cleanup di listener DOM.

## DOM e rendering

### Albero DOM, selezione e modifiche

Il DOM rappresenta documento, elementi, testo e commenti come *Node* collegati in un albero. Un *Element* è un Node che rappresenta un tag e offre proprietà come `classList` e selettori CSS (Cascading Style Sheets); un nodo di testo non è un Element. `querySelector(selector)` restituisce il primo Element corrispondente o `null`; un selettore non valido genera `SyntaxError`.

Per attraversare l'albero, `parentNode` e `childNodes` includono anche nodi non Element, mentre `parentElement`, `children` e `nextElementSibling` sono adatti a percorrere solo elementi. `createElement()` crea un elemento scollegato; `append()` inserisce nodi o stringhe, `remove()` stacca un nodo e `replaceChildren()` sostituisce tutti i figli. Inserire un nodo già presente lo **sposta**, non lo clona.

```js
const container = document.querySelector("#result");
if (container) {
  const message = document.createElement("p");
  message.textContent = "Nuovo";
  container.append(message);
  message.remove();
  container.replaceChildren("Nessun elemento");
}
```

### `textContent`, `innerHTML` e sicurezza

`textContent` legge o assegna testo senza interpretarlo come markup: è la scelta normale per dati non fidati. `innerHTML` interpreta una stringa come HTML e sostituisce i figli dell'elemento; dati controllati da un attaccante possono introdurre cross-site scripting (XSS, esecuzione di codice nel contesto della pagina). Se serve davvero HTML dinamico, usare markup fidato o una sanitizzazione adatta al contesto, non concatenare input non fidato.

### Layout, repaint e prestazioni

Una modifica al DOM o agli stili può invalidare il *layout* (ricalcolo di dimensioni e posizioni, spesso detto reflow) e il *paint* (ridisegno dei pixel); non ogni modifica richiede entrambi. Il browser normalmente raggruppa il lavoro prima di un frame, ma leggere geometria dopo scritture che la invalidano può forzare un layout immediato. Evitare alternanze ripetute fra scritture e letture di misure; usare `requestAnimationFrame()` per sincronizzare modifiche visive quando serve.

## Eventi e delega

### Registrazione, propagazione e controllo

`addEventListener(type, handler, options)` registra più listener per lo stesso evento; `removeEventListener()` richiede lo stesso tipo, la stessa funzione e lo stesso valore di `capture`. L'evento percorre prima la fase di *capturing* dagli antenati verso il target, poi la fase di *bubbling* verso gli antenati se `bubbles` è vero. I listener ascoltano di default nel bubbling; `{ capture: true }` li sposta nel capturing.

- `event.target` è il target dell'evento; `event.currentTarget` è l'oggetto sul quale sta girando il listener. Durante il bubbling spesso sono diversi; oltre un confine Shadow DOM il target può essere rimappato.
- `preventDefault()` impedisce l'azione predefinita **se l'evento è cancellabile**; non ferma la propagazione. Non funziona in un listener `{ passive: true }`.
- `stopPropagation()` ferma il percorso verso altri nodi ma non gli altri listener sul nodo corrente. `stopImmediatePropagation()` ferma anche questi ultimi.

La *delegation* registra un listener su un antenato per gestire eventi dei discendenti: è utile per liste dinamiche e riduce il numero di listener. Funziona solo per eventi che raggiungono l'antenato; usare `target` o `closest()` per individuare l'elemento interessato.

```js
const list = document.querySelector("#items");
list?.addEventListener("click", (event) => {
  if (!(event.target instanceof Element)) return;
  const button = event.target.closest("button[data-id]");
  if (button && list.contains(button)) console.log(button.dataset.id);
});
```

### Cleanup, CustomEvent e Shadow DOM

Un listener mantiene riferimenti al suo callback e ai dati catturati; se il target resta vivo, registrazioni ripetute senza cleanup possono trattenere memoria o duplicare effetti. Rimuoverlo quando non serve più con `removeEventListener()` oppure con un `AbortSignal` associato; `{ once: true }` lo rimuove dopo la prima invocazione.

```js
function registerSave(button) {
  const onSave = () => console.log("salvato");
  button.addEventListener("click", onSave);
  return () => button.removeEventListener("click", onSave);
}
// Conservare la funzione restituita e chiamarla nel cleanup.
```

`CustomEvent` permette di emettere un evento applicativo con dati in `detail`; `dispatchEvent()` lo consegna **sincronicamente** ai listener. Se deve propagarsi, impostare `{ bubbles: true }`; per attraversare un confine Shadow DOM serve anche `{ composed: true }`.

```js
const cart = document.querySelector("#cart");
cart?.dispatchEvent(new CustomEvent("cart:add", {
  detail: { id: 42 }, bubbles: true,
}));
```

Lo *Shadow DOM* crea un sottoalbero associato a un host, con incapsulamento di struttura e stili. `host.attachShadow({ mode: "open" })` espone `host.shadowRoot`; la modalità `closed` limita tale accesso tramite la proprietà, ma non è una barriera di sicurezza. I selettori del documento non attraversano automaticamente il confine e gli eventi possono essere rimappati verso l'host.

## Web API e fetch

### Web API e runtime browser

Le Web API sono interfacce fornite dal browser, come DOM, timer e Fetch: l'engine esegue JavaScript, mentre il runtime fornisce queste operazioni e mette in coda i relativi callback. La disponibilità e il comportamento di un'API dipendono dall'ambiente; non assumere che `document` esista in un Web Worker o in un runtime server. La concorrenza delle operazioni di rete non rende non bloccante un callback JavaScript lungo sul thread principale.

### Fetch: richiesta, risposta e JSON

`fetch(url, options)` crea una richiesta HTTP e restituisce una Promise di `Response`. `Request` descrive metodo, URL, header e body; `Headers` rappresenta gli header della richiesta o risposta. Il metodo `response.json()` legge il body e lo interpreta come JSON, restituendo a sua volta una Promise; un JSON malformato causa errore di parsing e il body non si può consumare due volte.

`fetch()` può rifiutare per errori di rete o annullamento, **non** per un normale status HTTP di errore: anche un 404 restituisce una `Response`. `response.status` è il codice numerico; `response.ok` è vero per status 200–299. Verificare `ok` prima di trattare il body come successo, poi distinguere errori HTTP, di rete, di parsing e annullamento secondo il caso d'uso.

```js
const controller = new AbortController();
async function loadItems() {
  try {
    const response = await fetch("/api/items", {
      headers: { Accept: "application/json" }, signal: controller.signal,
    });
    if (!response.ok) throw new Error(`HTTP ${response.status}`);
    return await response.json();
  } catch (error) {
    if (error.name === "AbortError") return;
    throw error;
  }
}
const cancel = () => controller.abort(); // da chiamare nel cleanup
```

Per inviare JSON, serializzare il body con `JSON.stringify()` e indicare `Content-Type: application/json`; `Accept` indica invece il formato di risposta desiderato. L'annullamento non annulla effetti già compiuti dal server.
