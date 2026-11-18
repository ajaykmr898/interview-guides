# JavaScript avanzato ed ECMAScript

Questo capitolo tratta i meccanismi avanzati di JavaScript e l'evoluzione di ECMAScript, lo standard che ne definisce linguaggio e libreria di base. Distingue le API (Application Programming Interface, interfacce di programmazione) standard dal supporto effettivo dei runtime e dagli strumenti usati per distribuirle.

## Indice

- [Destructuring spread e rest](#destructuring-spread-e-rest)
- [Moduli e strict mode](#moduli-e-strict-mode)
- [Iteratori e generatori](#iteratori-e-generatori)
- [Map Set e WeakMap](#map-set-e-weakmap)
- [Proxy e Reflect](#proxy-e-reflect)
- [Classi e private fields](#classi-e-private-fields)
- [Errori RegExp Intl e Date](#errori-regexp-intl-e-date)
- [Immutabilità e performance](#immutabilità-e-performance)
- [Evoluzione e compatibilità](#evoluzione-e-compatibilità)

## Destructuring spread e rest

Destructuring estrae valori da un iterabile o proprietà da un oggetto; *rest* raccoglie i valori rimanenti, mentre *spread* li espande. In array e chiamate, spread richiede un iterabile; in un oggetto copia le proprietà proprie enumerabili ed è superficiale. Per sintassi e parametri vedere [JavaScript basics](03-javascript-basics.md#parametri-restspread-e-destructuring).

```js
const [first, ...others] = new Set([1, 2, 3]);
first;  // 1
others; // [2, 3]
```

## Moduli e strict mode

### ES Modules e import dinamico

Un ES Module (ESM, modulo ECMAScript) ha scope proprio, esporta binding con `export` e li importa con `import`. Gli import statici sono dichiarazioni al livello superiore: il runtime può analizzare e collegare le dipendenze prima della valutazione. Gli export sono *live bindings*: chi importa osserva gli aggiornamenti del binding esportato, senza poterlo riassegnare direttamente.

Nel browser gli import statici richiedono uno script `type="module"`; `export default` espone un valore predefinito importato senza parentesi graffe. La valutazione di un modulo può fallire se una dipendenza non viene caricata o genera un errore.

```js
// math.mjs
export const double = (value) => value * 2;
// main.mjs
import { double } from "./math.mjs";
double(3); // 6
```

`import(specifier)` è un'espressione: restituisce una Promise che si risolve con il *module namespace object*, o si rifiuta se caricamento o valutazione falliscono. Serve per caricare codice quando occorre, per esempio una funzionalità usata raramente; il comportamento di rete, risoluzione e caching dipende dall'host. L'import statico rimane preferibile per dipendenze necessarie subito e analisi statica. Per Promise e `await` vedere [JavaScript async e DOM](04-javascript-async-dom.md#promise-e-async-await).

```js
async function calculate() {
  const { double } = await import("./math.mjs");
  return double(3); // 6
}
```

### ESM, CommonJS e strict mode

| Aspetto | ESM | CommonJS (CJS) |
| --- | --- | --- |
| Caricamento | `import`/`export` statici; `import()` dinamico restituisce una Promise. | `require()` carica tramite il loader CommonJS di Node.js e, per un modulo CJS, restituisce `module.exports` in modo sincrono. |
| Export | Binding collegati e aggiornabili dal modulo esportatore. | Proprietà del valore `module.exports`; `exports` è inizialmente un suo alias. |
| Ambiente | Standard ECMAScript, supportato in browser e runtime compatibili. | Sistema storico di Node.js, non sintassi ECMAScript per moduli browser. |
| Errore | `import()` rifiuta la Promise; un import statico non può completare se il modulo non si carica o valuta. | `require()` lancia un'eccezione durante la chiamata se il caricamento fallisce. |

```js
// math.cjs
module.exports = { double: (value) => value * 2 };
// main.cjs
const { double } = require("./math.cjs");
double(3); // 6
```

Riassegnare solo `exports` non cambia `module.exports`; destrutturare il risultato di `require()` copia i valori letti, senza creare live bindings. L'interoperabilità tra ESM e CJS dipende dal runtime e dalla sua versione: non assumere che `require()` possa caricare qualsiasi ESM, in particolare uno con top-level `await`. In Node.js l'estensione del file e la configurazione del package contribuiscono a determinare il sistema di moduli. Per la risoluzione specifica di Node.js vedere [Node e npm](../node-tooling/08-node-npm.md#moduli-e-risoluzione).

In *strict mode*, errori che in codice legacy potevano passare inosservati diventano eccezioni e non si possono creare globali assegnando identificatori non dichiarati. Gli ESM e i corpi delle classi sono sempre strict; uno script classico può attivarlo con `"use strict"` all'inizio. Strict mode non introduce controlli statici dei tipi.

## Iteratori e generatori

### Symbol e protocolli di iterazione

Un `Symbol` è un primitivo univoco usabile come chiave di proprietà; `Symbol.for(key)` riusa invece un simbolo del registro condiviso. I *well-known symbols* definiscono protocolli del linguaggio: `Symbol.iterator` identifica il metodo che rende un valore *iterable*. Per la definizione base di `Symbol` vedere [JavaScript basics](03-javascript-basics.md#valori-primitivi-e-oggetti).

Un iterable fornisce `[Symbol.iterator]()` che restituisce un *iterator*. Ogni `next()` dell'iterator restituisce un oggetto `{ value, done }`; `done: true` segnala la fine. `for...of`, destructuring e spread consumano questo protocollo. Un iterator non è necessariamente iterable: per usarlo direttamente con `for...of` deve avere a sua volta `[Symbol.iterator]()`.

```js
const values = ["a", "b"];
const iterator = values[Symbol.iterator]();
iterator.next(); // { value: "a", done: false }
iterator.next(); // { value: "b", done: false }
iterator.next(); // { value: undefined, done: true }
```

### Generator e async iterator

Una funzione `function*` crea un *generator*: la chiamata restituisce subito un oggetto iterator/iterable, senza eseguire il corpo fino a `next()`. `yield` produce un valore e sospende l'esecuzione; `yield*` delega a un altro iterable. La ripresa è sincrona: un generator non rende automaticamente asincrono il lavoro interno. `return()` può chiudere anticipatamente il generator ed eseguire un eventuale `finally` se era sospeso.

```js
function* numbers() {
  yield 1;
  yield 2;
}
const sequence = numbers();
sequence.next(); // { value: 1, done: false }
[...numbers()];  // [1, 2]
```

Un *async iterable* espone `[Symbol.asyncIterator]()`; il suo iterator restituisce da `next()` una Promise per `{ value, done }`. `async function*` combina `await` e `yield`, e `for await...of` attende i valori uno alla volta. È utile per stream o pagine di dati; l'uscita anticipata dal ciclo invoca `return()` dell'iterator se disponibile, ma il rilascio di risorse deve essere implementato dalla sorgente.

```js
async function* pages() {
  yield await Promise.resolve("prima");
  yield "seconda";
}
async function readPages() {
  for await (const page of pages()) console.log(page);
}
```

## Map Set e WeakMap

### Map e object; Set e array

`Map` associa chiavi di qualsiasi tipo a valori e mantiene l'ordine di inserimento nell'iterazione. `set(key, value)` restituisce la Map, `get(key)` restituisce il valore o `undefined`, `has(key)` un booleano e `delete(key)` un booleano che indica se la chiave era presente. `size` dà il numero di voci; `get()` da solo non distingue una chiave assente da una associata a `undefined`.

| Scelta | Punti forti | Limiti o differenze |
| --- | --- | --- |
| `Map` | Chiavi di ogni tipo, iterazione diretta, `size`, inserimento/rimozione frequenti. | Non è un normale record JSON (JavaScript Object Notation); un oggetto chiave è confrontato per identità. |
| Object | Proprietà `string` o `symbol`, sintassi naturale per record con campi noti. | Ha un prototipo salvo creazione esplicita senza prototipo; non offre l'API di Map. |

`Set` contiene valori unici e iterabili in ordine di inserimento. `add(value)` restituisce il Set; `has()` e `delete()` restituiscono booleani. `Map` e `Set` usano *SameValueZero*: `NaN` coincide con sé stesso e `+0` con `-0`, mentre due oggetti distinti restano diversi.

| Scelta | Uso | Limite |
| --- | --- | --- |
| `Set` | Appartenenza e deduplicazione senza indici. | Non conserva duplicati né accesso per posizione. |
| Array | Sequenza indicizzata con duplicati e metodi di trasformazione. | Per deduplicare occorre logica aggiuntiva; la ricerca per valore è lineare. |

```js
const key = { id: 1 };
const map = new Map([[key, "Ada"]]);
map.get(key);         // "Ada"
map.get({ id: 1 });   // undefined: altra identità
const unique = new Set([NaN, NaN, 0, -0]);
unique.size;          // 2
```

### WeakMap, WeakSet e riferimenti deboli

`WeakMap` associa valori a chiavi che possono essere raccolte dal garbage collector: oggetti e, negli standard recenti, simboli non registrati. `set()` restituisce la WeakMap; `get()` restituisce il valore o `undefined`, `has()` e `delete()` booleani. `WeakSet` conserva solo valori raccoglibili e offre `add()` (restituisce il WeakSet), `has()` e `delete()`. Chiavi o valori non validi causano `TypeError` all'inserimento.

| Proprietà | `Map` | `WeakMap` |
| --- | --- | --- |
| Chiavi | Qualunque valore. | Oggetti o simboli non registrati; verificare il supporto del target per questi ultimi. |
| Riferimenti | Forti: una voce resta finché non è rimossa o la Map diventa irraggiungibile. | Deboli rispetto alle chiavi: la raccolta della chiave può rendere eliminabile la voce. |
| Ispezione | Iterabile, `size`, `clear()`. | Non iterabile; nessun `size` o `clear()`. |

```js
const metadata = new WeakMap();
const owner = {};
metadata.set(owner, { selected: true });
metadata.get(owner).selected; // true
metadata.has(owner);          // true
```

`WeakMap` è utile per metadati legati alla vita di un oggetto; `WeakSet` per segnare oggetti senza trattenerli solo per questo. Nessuna delle due strutture garantisce **quando** avverrà la raccolta: non usarle per conteggi o cleanup deterministico. `WeakRef` espone `deref()`, che restituisce il valore referenziato se ancora disponibile oppure `undefined`; `FinalizationRegistry` può notificare una raccolta, ma il callback non è garantito. Preferire cleanup esplicito per risorse importanti.

## Proxy e Reflect

### Intercettazione e reflection

`new Proxy(target, handler)` restituisce un oggetto che intercetta operazioni sul target mediante *trap* come `get`, `set` o `deleteProperty`. È utile per validazione, logging o interfacce virtuali; l'accesso diretto al target aggira il Proxy e le trap devono rispettare gli invarianti del linguaggio, per esempio le proprietà non configurabili. Un Proxy non rende automaticamente reattivi tutti gli oggetti annidati.

`Reflect` raccoglie metodi per operazioni corrispondenti, spesso usati nelle trap per delegare il comportamento normale. `Reflect.get()` restituisce il valore; `Reflect.set()` e `Reflect.deleteProperty()` restituiscono booleani di successo. Una trap `set` deve restituire un booleano: un esito falso può produrre `TypeError` in strict mode. Chiamare `Reflect` su un Proxy può comunque attraversarne le trap; non è un modo per aggirarlo.

```js
const state = { count: 0 };
const guarded = new Proxy(state, {
  set(target, key, value, receiver) {
    if (key === "count" && (!Number.isInteger(value) || value < 0)) {
      throw new RangeError("count non valido");
    }
    return Reflect.set(target, key, value, receiver);
  },
});
guarded.count = 2; // state.count diventa 2
```

### Descriptors e accessors

Una proprietà *data* ha `value` e `writable`; una proprietà *accessor* ha `get` e/o `set`. Entrambe hanno `enumerable` e `configurable`. `Object.getOwnPropertyDescriptor(obj, key)` restituisce il descriptor o `undefined`; `Object.defineProperty(obj, key, descriptor)` restituisce l'oggetto modificato o lancia `TypeError` se la modifica viola i vincoli. Per una nuova proprietà definita così, gli attributi omessi `writable`, `enumerable` e `configurable` valgono `false`.

```js
const record = { name: "Ada" };
Object.defineProperty(record, "id", { value: 7 });
Object.getOwnPropertyDescriptor(record, "id").enumerable; // false
const person = { get label() { return this.name.toUpperCase(); } };
person.name = "Ada";
person.label; // "ADA"
```

Getter e setter eseguono codice all'accesso o assegnazione, quindi una lettura non è necessariamente priva di effetti. Per le basi di proprietà e prototipi vedere [JavaScript basics](03-javascript-basics.md#proprietà-accessors-e-descriptors).

## Classi e private fields

`class` organizza costruttore, metodi ed ereditarietà prototipale; `extends` e `super` sono trattati in [JavaScript basics](03-javascript-basics.md#prototype-chain-inheritance-e-classi). I *private fields* con prefisso `#` sono elementi privati dichiarati nel corpo della classe: il linguaggio ne impedisce l'accesso lessicale dall'esterno, a differenza della convenzione `_name`.

```js
class Counter {
  #count = 0;
  increment() { return ++this.#count; }
  get value() { return this.#count; }
}
const counter = new Counter();
counter.increment(); // 1
counter.value;       // 1
```

Un private field non è una normale proprietà con chiave stringa: non compare in `Object.keys()`, non passa attraverso le trap di Proxy e una sottoclasse non può accedervi direttamente. Chiamare un metodo che usa `#count` con un ricevente non inizializzato da `Counter` produce `TypeError`. Closure e `WeakMap` erano alternative per lo stato privato prima di questa sintassi; la scelta dipende anche dalla compatibilità richiesta.

## Errori RegExp Intl e Date

### Tipi di errore e `Error.cause`

`Error` rappresenta un fallimento con `name` e `message`; `TypeError` segnala un tipo o un'operazione non valida, `ReferenceError` un identificatore non risolto, `RangeError` un valore fuori intervallo e `SyntaxError` codice non analizzabile. `AggregateError` raggruppa più errori, per esempio quando tutte le Promise di `Promise.any()` falliscono. Un errore di parsing dello stesso script impedisce di eseguirlo, quindi non si intercetta con un `try`/`catch` scritto al suo interno.

Una classe derivata da `Error` permette di distinguere errori applicativi con `instanceof`. `new Error(message, { cause })` conserva l'errore originale aggiungendo contesto; `cause` può essere di qualsiasi tipo e non è necessariamente presente. `stack` è utile per diagnosi, ma formato e disponibilità dipendono dal runtime.

```js
class DataError extends Error {
  constructor(message, cause) {
    super(message, { cause });
    this.name = "DataError";
  }
}
const cause = new TypeError("formato inatteso");
new DataError("lettura fallita", cause).cause === cause; // true
```

### Espressioni regolari

`RegExp` descrive un pattern per cercare o trasformare stringhe: un literal `/pattern/flags` o `new RegExp(pattern, flags)` crea l'oggetto. `test(text)` restituisce un booleano, `exec(text)` un match con gruppi o `null`; metodi di `String` come `match`, `replace` e `matchAll` usano lo stesso meccanismo. Le flag `i` e `g` indicano rispettivamente ricerca senza distinzione fra maiuscole e minuscole e ricerca globale.

```js
const pattern = /(?<name>[a-z]+)=(?<value>\d+)/i;
const match = pattern.exec("count=42");
match?.groups.name;  // "count"
match?.groups.value; // "42"
```

Con `g` o `y`, `test()` ed `exec()` aggiornano `lastIndex`: riusare la stessa RegExp può cambiare il risultato delle chiamate successive. Pattern complessi su input non fidato possono consumare molto tempo per backtracking; scegliere espressioni semplici e misurare sui casi reali.

### `Intl` e `Date`

`Intl` (Internationalization) è l'API di internazionalizzazione standardizzata separatamente in ECMA-402: `Intl.NumberFormat` e `Intl.DateTimeFormat` producono stringhe formattate per locale. `format()` restituisce una stringa e l'output può dipendere dai dati locali del runtime; specificare `locale` e `timeZone` quando il risultato deve essere prevedibile. La formattazione non cambia il valore sottostante.

`Date` rappresenta un istante tramite millisecondi dall'epoch UTC (Coordinated Universal Time), non conserva un fuso orario. I metodi locali interpretano l'istante nel fuso del runtime; `toISOString()` restituisce UTC. Per scambiare istanti usare stringhe con offset esplicito o `Z`: il parsing di formati non standard può variare. Una `Date` invalida ha `getTime()` pari a `NaN`, e `toISOString()` lancia `RangeError`.

```js
const instant = new Date("2025-01-02T12:00:00Z");
instant.toISOString(); // "2025-01-02T12:00:00.000Z"
new Intl.DateTimeFormat("it-IT", {
  timeZone: "Europe/Rome", dateStyle: "medium",
}).format(instant); // stringa localizzata
new Intl.NumberFormat("it-IT").format(1234.5); // stringa localizzata
```

## Immutabilità e performance

### Memoria, reachability e leak

Il *garbage collector* (GC) può recuperare oggetti non più raggiungibili dalle radici dell'esecuzione; un ciclo di riferimenti non è di per sé un leak se l'intero ciclo è irraggiungibile. Un *memory leak* si verifica quando riferimenti ancora raggiungibili trattengono dati inutili, per esempio cache senza limite, listener non rimossi o closure conservate a lungo. `WeakMap` aiuta solo se è la chiave a non essere più trattenuta altrove; non sostituisce la rimozione di listener o la chiusura di risorse.

L'immutabilità facilita il ragionamento e il confronto di stati, ma copiare grafi grandi crea allocazioni e lavoro. Spread e `Object.freeze()` agiscono superficialmente; per comportamento e limiti della copia profonda vedere [Immutabilità e copie](03-javascript-basics.md#immutabilità-e-copie). Preferire aggiornamenti mirati e misurare prima di adottare copie profonde sistematiche.

### Responsività, lazy loading e profilazione

Un loop sincrono lungo o una catena illimitata di microtask può causare *event loop starvation*: timer, input e rendering restano in attesa. Suddividere lavoro sul thread principale o spostare calcolo adatto in un Worker; il modello di task e microtask è in [JavaScript async e DOM](04-javascript-async-dom.md#event-loop-e-task).

Il *lazy loading* differisce il caricamento di codice o dati finché servono; `import()` può rimandare il caricamento di un modulo, mentre un bundler può creare chunk separati secondo la sua configurazione. Riduce il lavoro iniziale ma introduce latenza al primo uso e un possibile errore di caricamento. Profilare tempi, allocazioni e heap con gli strumenti del runtime prima di ottimizzare: una misura locale non garantisce lo stesso risultato su altri dispositivi.

### Pattern e limiti del linguaggio dinamico

Il pattern *module* delimita API e stato; una *factory* restituisce oggetti configurati senza imporre una gerarchia; *observer* distribuisce notifiche a più interessati, come negli [eventi DOM](04-javascript-async-dom.md#eventi-e-delega). Sono soluzioni per problemi ricorrenti, non strutture obbligatorie: callback e registrazioni richiedono spesso cleanup, mentre livelli di astrazione inutili rendono più difficile seguire il flusso.

JavaScript lega i tipi ai valori e consente oggetti modificabili: è flessibile ai confini dinamici, ma errori di forma o tipo possono emergere solo a runtime. Validare input esterni, mantenere contratti chiari e usare controlli statici o test quando il rischio lo giustifica; un type checker non sostituisce la validazione di dati non fidati.

## Evoluzione e compatibilità

### TC39 e principali edizioni

TC39 (Technical Committee 39) mantiene ECMA-262, la specifica di ECMAScript, tramite proposte e revisioni periodiche. Uno *stage* descrive la maturità di una proposta, non la disponibilità universale nel runtime; una proposta non incorporata in un'edizione pubblicata non va presentata come API standard stabile.

| Edizione | Evoluzione rilevante |
| --- | --- |
| ES5 (2009) | Strict mode, accessors, property descriptors e supporto JSON standardizzato. |
| ES2015 (ES6) | Moduli, `let`/`const`, classi, Promise, generatori, Map e Set. |
| ES2017 | Funzioni `async` e `await`. |
| ES2020 | `import()`, `BigInt`, `Promise.allSettled()`, optional chaining `?.` e nullish coalescing `??`. |
| ES2022 | Campi privati di classe, top-level `await` ed `Error.cause`. |
| ES2023 | Metodi di copia degli array come `toSorted()` e `toSpliced()`. |
| ES2024 | `Promise.withResolvers()`, `Object.groupBy()` e flag RegExp `/v`. |
| ES2025 | Iterator helpers, metodi di insieme per `Set` e `RegExp.escape()`. |

ES2026 è la 17ª edizione pubblicata di ECMA-262; la tabella seleziona solo alcune tappe. La presenza nella specifica, l'implementazione in un engine e l'attivazione di default sono passaggi distinti. Verificare il supporto dei target reali prima di usare un'API; non dedurlo dal solo numero di edizione.

### Target, transpilation, polyfill e feature detection

I *browser targets* sono le versioni di browser o runtime che un'applicazione decide di supportare. Definirli in base agli utenti e ai vincoli del prodotto permette agli strumenti di scegliere trasformazioni e polyfill necessari; aumentare il supporto legacy può aumentare dimensioni e costo della build. La risoluzione dei moduli e la generazione dei chunk dipendono anche dal bundler: vedere [Build tools](../node-tooling/09-build-tools.md#transpilation-e-polyfill).

| Tecnica | Che cosa fa | Che cosa non fa |
| --- | --- | --- |
| Transpilation, per esempio Babel | Trasforma sintassi in codice comprensibile ai target configurati. | Non aggiunge da sola API runtime mancanti come `Map` o `Promise`. |
| Polyfill | Aggiunge o emula una funzionalità API assente, se tecnicamente riproducibile. | Non permette al parser di leggere sintassi non supportata; può modificare oggetti globali. |
| Feature detection | Controlla una capacità al runtime e sceglie un percorso alternativo. | Non risolve sintassi che il parser del target non sa leggere. |

```js
const numbers = [3, 1, 2];
const sorted = typeof numbers.toSorted === "function"
  ? numbers.toSorted((a, b) => a - b)
  : [...numbers].sort((a, b) => a - b);
sorted; // [1, 2, 3]; numbers resta invariato
```

Un controllo di presenza non prova che ogni dettaglio semantico sia corretto: per funzionalità critiche servono test nei runtime supportati. Le trasformazioni di sintassi devono comunque produrre codice parsabile dai target prima che qualsiasi feature detection possa essere eseguita.
