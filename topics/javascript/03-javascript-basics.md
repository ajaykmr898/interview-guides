# JavaScript basics

JavaScript è un linguaggio dinamico basato su oggetti e funzioni, usato in diversi ambienti di esecuzione. ECMAScript è lo standard che ne definisce sintassi e semantica; un *engine* lo implementa, mentre il *runtime* aggiunge API dell'ambiente, per esempio DOM nel browser o accesso ai file in Node.js. Le API dell'ambiente non fanno parte del linguaggio ECMAScript.

## Indice

- [Tipi e coercizione](#tipi-e-coercizione)
- [Scope e closure](#scope-e-closure)
- [Funzioni e this](#funzioni-e-this)
- [Oggetti e prototipi](#oggetti-e-prototipi)
- [Array e collezioni](#array-e-collezioni)
- [Moduli](#moduli)

## Tipi e coercizione

### Valori primitivi e oggetti

I primitivi sono `string`, `number`, `bigint`, `boolean`, `symbol`, `undefined` e `null`. Un oggetto è un valore con identità e proprietà modificabili; array e funzioni sono oggetti. Le variabili non hanno un tipo fisso: il tipo appartiene al valore assegnato.

- `undefined` indica, tra l'altro, una variabile dichiarata ma non inizializzata o una proprietà assente; `null` rappresenta un'assenza assegnata intenzionalmente. `typeof null` restituisce `"object"` per una particolarità storica, ma `null` è un primitivo.
- `NaN` (*Not a Number*) è un valore del tipo `number` che rappresenta un risultato numerico non valido; `NaN === NaN` è `false`. Per riconoscerlo senza conversioni implicite usare `Number.isNaN()`.
- `BigInt` rappresenta interi di precisione arbitraria, per esempio `42n`; non si mescola direttamente con `number` nelle operazioni aritmetiche e non supporta frazioni.
- `Symbol()` crea un primitivo univoco, utile come chiave di proprietà senza collisioni accidentali. Due simboli creati separatamente non sono uguali anche se hanno la stessa descrizione.

### Coercizione, truthy e uguaglianza

La *type coercion* converte un valore quando un operatore richiede un tipo diverso: `"5" + 1` produce `"51"`, mentre `"5" - 1` produce `4`. Una condizione converte a booleano: sono *falsy* `false`, `0`, `-0`, `0n`, `""`, `null`, `undefined` e `NaN`; gli altri valori, inclusi `[]` e `{}`, sono *truthy*.

| Confronto | Comportamento | Caso significativo |
| --- | --- | --- |
| `==` | Converte i tipi secondo regole specifiche prima del confronto. | `0 == false` è `true`; `null == undefined` è `true`. |
| `===` | Non converte i tipi; confronta gli oggetti per identità. | `0 === false` è `false`; `NaN === NaN` è `false`. |
| `Object.is(a, b)` | Non converte; distingue gli zeri con segno e considera uguale `NaN` a sé stesso. | `Object.is(NaN, NaN)` è `true`; `Object.is(0, -0)` è `false`. |

**Best practice:** preferire `===` per evitare conversioni inattese. `x == null` è un uso intenzionale possibile quando si vogliono accettare sia `null` sia `undefined`, ma va reso esplicito nel contesto.

### Passaggio dei valori e operatori

Gli argomenti sono passati **per valore**: per un primitivo si copia il valore; per un oggetto si copia il valore del riferimento allo stesso oggetto. Mutare una sua proprietà è visibile al chiamante, riassegnare il parametro no. Anche `===` fra oggetti verifica l'identità, non l'uguaglianza strutturale.

- `?.` interrompe l'accesso o la chiamata se il valore a sinistra è `null` o `undefined`, restituendo `undefined`; non nasconde errori generati da una funzione esistente.
- `??` usa il valore a destra solo se quello a sinistra è `null` o `undefined`; `||` lo usa per qualunque valore falsy, compresi `0` e `""`.
- `&&` e `||` restituiscono uno degli operandi e valutano il secondo solo se serve; `!` produce un booleano. `condition ? a : b` sceglie un'espressione in base alla condizione.
- `=` assegna, mentre `+`, `-`, `*`, `/`, `%` e `**` eseguono le comuni operazioni aritmetiche, soggette alle regole di coercizione. `typeof` restituisce una stringa con il tipo; `in` verifica la presenza di una proprietà anche nella catena dei prototipi; `instanceof` controlla la catena rispetto a un costruttore.

```js
const options = { retries: 0 };
const retries = options?.retries ?? 3; // 0, non 3
const label = options?.label ?? "senza nome";
```

## Scope e closure

### Dichiarazioni, scope e hoisting

Lo *scope lessicale* determina la visibilità degli identificatori dalla posizione in cui il codice è scritto: una funzione interna può accedere ai nomi degli scope esterni. Lo scope globale è esterno alle funzioni; uno scope di funzione delimita i suoi parametri e le variabili locali; uno scope di blocco delimita le dichiarazioni `let` e `const` tra `{}`. Il codice di un modulo ha inoltre uno scope proprio.

| Dichiarazione | Scope | Prima della dichiarazione | Riassegnazione |
| --- | --- | --- | --- |
| `var` | Funzione o livello superiore di script/modulo; ignora il blocco. | Accessibile come `undefined` nello stesso scope. | Sì. |
| `let` | Blocco. | `ReferenceError` nella *Temporal Dead Zone* (TDZ). | Sì. |
| `const` | Blocco. | `ReferenceError` nella TDZ. | No; richiede inizializzazione. |

*Hoisting* descrive il comportamento delle dichiarazioni prima della loro posizione testuale: una function declaration è utilizzabile prima della riga in cui appare; `var` è inizializzato a `undefined`; `let` e `const` esistono nello scope ma restano in TDZ fino alla dichiarazione. `const` impedisce di riassegnare il binding, non di mutare l'oggetto referenziato.

```js
function scope() {
  if (true) { var old = 1; let current = 2; const fixed = 3; }
  return old; // 1; current e fixed non sono visibili qui
}
```

**Best practice:** usare `const` quando il binding non cambia, `let` quando serve riassegnarlo; riservare `var` alla lettura o manutenzione di codice legacy.

### Execution context, call stack e lexical environment

Un *execution context* è lo stato in cui viene eseguito il codice globale o una chiamata di funzione: comprende i binding disponibili, il valore di `this` dove applicabile e il collegamento all'ambiente lessicale esterno. Il *lexical environment* associa nomi a binding e mantiene tale collegamento; così la ricerca di un nome procede verso gli scope esterni definiti nel sorgente.

Il *call stack* registra le chiamate attive: una chiamata aggiunge un frame, il ritorno lo rimuove. La ricorsione senza una condizione di arresto può esaurire lo stack; il call stack da solo non descrive l'esecuzione asincrona, trattata in [JavaScript async e DOM](04-javascript-async-dom.md).

### Closure

Una *closure* è una funzione che conserva accesso ai binding del proprio scope lessicale anche dopo il ritorno della funzione esterna. Cattura i binding, non una fotografia dei loro valori: le modifiche successive sono osservabili. È utile per stato privato e callback, ma una closure mantenuta in vita può trattenere anche oggetti non più necessari.

```js
function counter() {
  let count = 0;
  return () => ++count;
}
const next = counter();
next(); // 1
next(); // 2
```

## Funzioni e this

### Forme e composizione delle funzioni

Le funzioni sono valori richiamabili: possono essere passate come argomenti, restituite e assegnate a variabili. Una *function declaration* crea un binding utilizzabile prima della dichiarazione; una *function expression* produce un valore funzione quando l'espressione viene valutata. Una *arrow function* usa sintassi compatta e ha differenze semantiche su `this` e costruzione.

- Un *callback* è una funzione passata a un'altra funzione per essere invocata in un momento o punto stabilito da quest'ultima. Una *higher-order function* accetta o restituisce funzioni; `map` ne è un esempio.
- La *ricorsione* richiama la stessa funzione, direttamente o indirettamente; richiede un caso base e può esaurire il call stack con troppi livelli.
- Il *currying* trasforma una funzione con più argomenti in una sequenza di funzioni a un argomento; facilita l'applicazione parziale, ma può rendere meno leggibili chiamate semplici.
- Un'IIFE (*Immediately Invoked Function Expression*) è un'espressione funzione eseguita subito. Era comune per isolare variabili prima degli scope di blocco e dei moduli; resta utile se serve eseguire subito una funzione senza esporre un nome.

```js
function add(a, b) { return a + b; }          // declaration
const multiply = function (a, b) { return a * b; }; // expression
const double = (value) => multiply(value, 2); // arrow e callback possibile
const curriedAdd = (a) => (b) => add(a, b);
curriedAdd(2)(3); // 5
```

### `this`, `bind`, `call` e `apply`

In una funzione regolare, `this` dipende dalla **chiamata**: `obj.method()` usa `obj`; una chiamata isolata usa `undefined` in strict mode e normalmente l'oggetto globale fuori da strict mode. `call(thisArg, ...args)` e `apply(thisArg, argsArray)` invocano subito con un `this` esplicito; `bind(thisArg, ...args)` restituisce una nuova funzione con `this` fissato.

Una arrow function non crea un proprio `this` né un proprio `arguments`: usa quelli lessicali esterni. `call`, `apply` e `bind` non possono cambiarne il `this`; inoltre una arrow non è un costruttore e non va usata come metodo quando serve il ricevente della chiamata.

```js
const person = { name: "Ada", nameOf() { return this.name; } };
const detached = person.nameOf;
detached.call(person);           // "Ada"
detached.apply(person, []);      // "Ada"
const bound = detached.bind(person);
bound();                         // "Ada"
```

Se una funzione che usa `this` viene passata come callback senza il suo oggetto, `bind` può mantenerne il ricevente. Una arrow creata all'interno di un metodo può invece catturare il `this` del metodo.

```js
const obj = {
  value: 1,
  reader() { return () => this.value; },
};
obj.reader().call({ value: 2 }); // 1: la arrow conserva this
```

### Parametri, rest/spread e destructuring

Un parametro con default, come `function f(x = 1)`, usa il default quando l'argomento è `undefined`, non quando è `null`. Il parametro *rest* `...items` raccoglie gli argomenti rimanenti in un array; la sintassi *spread* `...items` espande un iterabile in una chiamata o in un array, oppure le proprietà proprie enumerabili in un oggetto. La *destructuring assignment* estrae elementi di array o proprietà di oggetto in binding; i default seguono la stessa regola di `undefined`.

```js
function sum(first = 0, ...rest) {
  return [first, ...rest].reduce((total, value) => total + value, 0);
}
const { name, age = 18 } = { name: "Ada" };
sum(1, 2, 3); // 6
```

### Funzioni pure, memoization, debounce e throttle

Una funzione *pure* restituisce lo stesso risultato per gli stessi input e non produce effetti osservabili esterni: ciò permette la *referential transparency*, cioè sostituire una chiamata con il suo risultato senza cambiare il comportamento. La *memoization* memorizza risultati per riusarli con gli stessi input; è utile per calcoli ripetuti e costosi, ma introduce costo di memoria e richiede chiavi coerenti con l'uguaglianza desiderata. Con input mutabili o effetti collaterali il risultato memorizzato può diventare scorretto.

| Tecnica | Comportamento | Uso e limite |
| --- | --- | --- |
| Debounce | Esegue dopo un intervallo senza nuove chiamate. | Utile per ricerca mentre si digita; una sequenza continua può rinviare l'esecuzione indefinitamente. |
| Throttle | Limita l'esecuzione a una frequenza massima. | Utile per aggiornamenti durante scroll o resize; può perdere chiamate intermedie. |

Le due tecniche dipendono da timer dell'ambiente: vedere [event loop e task](04-javascript-async-dom.md#event-loop-e-task).

## Oggetti e prototipi

### Proprietà, accessors e descriptors

Un oggetto associa chiavi `string` o `symbol` a proprietà. Una *data property* contiene un valore; una *accessor property* definisce `get` e/o `set`, invocati all'accesso o assegnazione. Un *property descriptor* descrive `value`/`writable` oppure `get`/`set`, più `enumerable` e `configurable`; `Object.getOwnPropertyDescriptor()` lo legge e `Object.defineProperty()` lo imposta. Le due forme di proprietà sono alternative.

```js
const user = {
  first: "Ada",
  get label() { return this.first.toUpperCase(); },
};
Object.defineProperty(user, "id", { value: 1, enumerable: false });
user.label; // "ADA"
```

### Prototype chain, inheritance e classi

Ogni oggetto ordinario può avere un prototipo: quando una proprietà manca sull'oggetto, la ricerca prosegue lungo la *prototype chain* fino a `null`. La *prototypal inheritance* permette quindi di condividere metodi senza copiarli su ogni istanza; una proprietà propria può nascondere quella ereditata.

La sintassi `class` organizza costruttore e metodi sopra questo meccanismo. `extends` collega la sottoclasse alla superclasse; `super()` inizializza la parte base nel costruttore derivato prima di usare `this`, mentre `super.method()` richiama un metodo della classe base. L'ereditarietà è utile per un rapporto stabile “è un”; introduce però accoppiamento alla gerarchia.

```js
class Animal {
  speak() { return "suono"; }
}
class Dog extends Animal {
  speak() { return `${super.speak()} del cane`; }
}
new Dog().speak(); // "suono del cane"
```

### Encapsulation, polymorphism e composition

L'*encapsulation* limita l'accesso allo stato interno tramite un'interfaccia: una closure o un campo privato `#name` possono nascondere dati; una normale proprietà con prefisso `_` indica solo una convenzione. Il *polymorphism* consente a oggetti diversi di rispondere alla stessa operazione con comportamenti propri, per esempio ridefinendo `speak()`; in JavaScript può basarsi anche su metodi condivisi senza una classe base.

La *composition* costruisce un oggetto combinando comportamenti o collaboratori; è spesso più flessibile dell'*inheritance* quando non esiste una relazione stabile di specializzazione. Richiede però di coordinare esplicitamente i componenti. Preferire l'ereditarietà quando il contratto della classe base è davvero valido per le sottoclassi.

### Immutabilità e copie

Un valore è *immutabile* se non cambia dopo la creazione; i primitivi lo sono, gli oggetti normalmente no. Una *shallow copy* crea un nuovo contenitore ma condivide gli oggetti annidati; spread, `Object.assign()` e `slice()` copiano superficialmente. Una *deep copy* duplica anche i dati annidati supportati.

```js
const original = { profile: { name: "Ada" } };
const shallow = { ...original };
const deep = structuredClone(original);
shallow.profile.name = "Grace";
original.profile.name; // "Grace"
deep.profile.name;     // "Ada"
```

`structuredClone()` è un'API dell'ambiente: gestisce anche riferimenti ciclici e diversi tipi incorporati, ma non clona funzioni o nodi DOM e non preserva tutti i metadati, come descriptors e prototipi personalizzati; può lanciare `DataCloneError`. `Object.freeze()` impedisce aggiunta e rimozione di proprietà proprie e rende non scrivibili quelle che contengono dati; un eventuale setter può ancora avere effetti. Agisce solo sul primo livello: gli oggetti annidati restano mutabili se non congelati separatamente. Copiare o congelare grafi grandi ha un costo; scegliere la profondità necessaria.

## Array e collezioni

Un array è un oggetto indicizzato e ordinato, adatto a sequenze di valori. I metodi seguenti ricevono spesso callback; quelli che producono un nuovo array fanno una copia superficiale degli elementi e non clonano gli oggetti contenuti.

| Metodo | Risultato e uso principale | Muta l'array originale? |
| --- | --- | --- |
| `map` | Nuovo array con un risultato per elemento; usare per trasformare. | No. |
| `filter` | Nuovo array con gli elementi che soddisfano un predicato. | No. |
| `reduce` | Un accumulatore; con array vuoto senza valore iniziale lancia `TypeError`. | No, salvo mutazioni nel callback. |
| `find` | Primo elemento che soddisfa un predicato, oppure `undefined`. | No. |
| `some` / `every` | Booleano: almeno uno / tutti soddisfano il predicato; si fermano appena il risultato è noto. | No. |
| `forEach` | `undefined`; usa il callback per effetti, non per costruire un array. | No, salvo mutazioni nel callback. |
| `sort` | Ordina e restituisce lo stesso array; senza comparatore confronta stringhe. | Sì. |
| `slice` | Nuovo array con una porzione, senza includere l'indice finale. | No. |
| `splice` | Array degli elementi rimossi; inserisce o rimuove nell'originale. | Sì. |
| `flatMap` | Mappa e appiattisce di un solo livello il risultato. | No. |

**Differenze:** `map` è adatto a una trasformazione e restituisce un array; `forEach` è adatto a effetti collaterali e ignora i valori restituiti dal callback. `sort((a, b) => a - b)` serve per numeri crescenti; se l'originale deve restare intatto, copiare prima l'array. Per collezioni diverse dagli array, vedere [Map, Set e WeakMap](05-javascript-advanced-ecmascript.md#map-set-e-weakmap).

```js
const values = [1, 2, 3];
const doubled = values.map((value) => value * 2); // [2, 4, 6]
values.forEach((value) => console.log(value));     // ritorna undefined
```

## Moduli

Un modulo isola i propri binding e dichiara esplicitamente le dipendenze. Sintassi `import`/`export`, caricamento dinamico e confronto con CommonJS sono trattati in [JavaScript avanzato ed ECMAScript](05-javascript-advanced-ecmascript.md).
