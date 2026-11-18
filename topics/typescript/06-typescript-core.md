# TypeScript core

TypeScript è un superset di JavaScript che aggiunge un sistema di tipi per l'analisi statica: controlla il codice prima dell'esecuzione e normalmente rimuove le annotazioni nell'output JavaScript. L'inferenza deduce molti tipi dai valori; né le annotazioni né il type checking validano input o impediscono errori a runtime. Alcuni costrutti, come `enum`, generano invece codice JavaScript.

## Indice

- [Tipi e inferenza](#tipi-e-inferenza)
- [Union intersection e narrowing](#union-intersection-e-narrowing)
- [Interface e type alias](#interface-e-type-alias)
- [Funzioni e overload](#funzioni-e-overload)
- [Generics](#generics)
- [Moduli e configurazione](#moduli-e-configurazione)

## Tipi e inferenza

I tipi primitivi più usati sono `string`, `number`, `boolean`, `bigint`, `symbol`, `null` e `undefined`; `number` copre sia interi sia numeri a virgola mobile di JavaScript. `T[]`/`Array<T>` descrive un array di elementi omogenei, mentre una **tuple** descrive posizione, tipo e, se presenti, elementi opzionali o rest. Un **literal type** come `"ready"` ammette un solo valore; `const` e il contesto d'uso influenzano se viene inferito il literal o il più ampio `string`.

```ts
const phase = "ready"; // tipo "ready"
let label = "ready";   // tipo string: può cambiare
const scores: number[] = [8, 9];
const entry: readonly [string, number] = ["Ada", 42];
type Profile = { readonly id: number; nickname?: string };
```

Una proprietà `nickname?: string` può mancare: in lettura il suo tipo include `undefined` con `strictNullChecks`. `readonly` impedisce assegnazioni tramite quel riferimento nel type checking, ma non congela l'oggetto a runtime; `readonly T[]` vieta i metodi mutanti in quel riferimento. I controlli delle proprietà extra sono più severi per object literal nuovi assegnati a un tipo esplicito, pur restando la compatibilità principalmente strutturale.

Con `strictNullChecks`, `null` e `undefined` sono valori distinti da dichiarare esplicitamente nelle union (`string | null`, `string | undefined`). `null` rappresenta spesso un'assenza intenzionale; `undefined` anche un valore non inizializzato, una proprietà assente o il risultato di una funzione senza `return`. Disabilitare il controllo rende entrambi assegnabili a molti tipi e nasconde possibili errori di accesso.

| Tipo | Significato e limite |
| --- | --- |
| `any` | Disattiva i controlli sulle operazioni e propaga l'incertezza: una chiamata inesistente può fallire a runtime. |
| `unknown` | Accetta qualunque valore, ma richiede un controllo o un'assertion prima dell'uso; adatto a dati non fidati. |
| `never` | Nessun valore possibile: ramo irraggiungibile o funzione che non termina normalmente. |
| `void` | Risultato da ignorare, tipico delle funzioni senza valore restituito; non equivale a `never`. |

Una funzione di tipo `() => void` può avere un'implementazione che restituisce un valore, ma chi la chiama tramite quel tipo non può usarlo. Preferire `unknown` ad `any` ai confini non tipizzati e restringerlo con verifiche reali.

## Union intersection e narrowing

Una **union** `A | B` ammette un valore di uno dei tipi; senza narrowing sono accessibili solo le operazioni valide per tutti i membri. Un'**intersection** `A & B` richiede entrambe le strutture: non è una fusione runtime e proprietà incompatibili possono diventare impossibili da soddisfare. Una **discriminated union** usa una proprietà literal comune per collegare ogni variante ai suoi campi.

```ts
type Result = { kind: "ok"; value: number } | { kind: "error"; message: string };
function describe(result: Result): string {
  switch (result.kind) {
    case "ok": return String(result.value);
    case "error": return result.message;
    default: const unreachable: never = result; return unreachable;
  }
}
```

Il **narrowing** restringe il tipo lungo i rami del flusso di controllo: `typeof` verifica categorie JavaScript (`"string"`, `"number"` ecc.; `typeof null` è `"object"`), `in` verifica una proprietà e `instanceof` la [catena dei prototipi](../javascript/03-javascript-basics.md#prototype-chain-inheritance-e-classi) di una classe. Con `in`, una proprietà opzionale può comparire in entrambi i rami. Il controllo su `never` nel `default` segnala nuove varianti non gestite.

Un **user-defined type guard** espone un predicato `value is T`; il compilatore si fida della firma, quindi l'implementazione deve verificare i campi necessari. Per input esterni conviene partire da `unknown`:

```ts
type User = { id: number };
function isUser(value: unknown): value is User {
  return typeof value === "object" && value !== null
    && "id" in value && typeof value.id === "number";
}
const payload: unknown = JSON.parse('{"id": 7}');
if (isUser(payload)) console.log(payload.id); // number
const assumed = payload as User; // nessun controllo runtime
```

Una **type assertion** (`as T`) cambia solo il tipo visto dal compilatore e non controlla né converte il valore; `Number(value)` esegue invece una conversione reale e può produrre `NaN`. Per dati non fidati usare un guard o una validazione, non un'assertion. TypeScript usa **structural typing**: un valore è compatibile se possiede i membri richiesti con tipi compatibili, indipendentemente dal nome della sua dichiarazione; le proprietà extra di un oggetto già costruito possono quindi restare presenti anche dopo l'assegnazione.

## Interface e type alias

`interface` e `type` descrivono contratti statici e scompaiono dall'output; `enum` è invece un costrutto che, di norma, crea un valore a runtime.

| Costrutto | Uso e differenza principale |
| --- | --- |
| `interface` | Forma di oggetti e classi, estendibile con `extends`; dichiarazioni omonime compatibili si fondono (**declaration merging**). |
| `type` | Alias di qualsiasi tipo, anche union, tuple, primitivi e intersection; un nome non si riapre con altre dichiarazioni. |
| `enum` | Insieme nominato di valori numerici o stringa, con oggetto JavaScript emesso salvo casi come `const enum`. |

```ts
interface GuideEntity { id: string }
interface GuideEntity { name: string } // merging: entrambi i campi richiesti
type Timestamped = GuideEntity & { createdAt: Date };
type Identifier = string | number;
```

`interface A extends B` segnala subito membri ereditati incompatibili; un'intersection può invece produrre un membro impossibile (`never`). Scegliere `interface` per contratti oggetto estendibili, `type` per composizioni e union. Entrambi seguono il sistema strutturale: il nome non rende nominale il contratto.

Un **numeric enum** assegna numeri ai membri (implicitamente da `0` o da un valore iniziale) e l'oggetto emesso supporta il reverse mapping da numero a nome. Un **string enum** richiede valori stringa espliciti e non ha reverse mapping. L'enum esiste a runtime, ma non valida automaticamente valori ricevuti dall'esterno.

```ts
enum Direction { Up, Down }
Direction.Up;  // 0
Direction[0]; // "Up": reverse mapping del numeric enum
enum LoadState { Ready = "READY" }
LoadState.Ready; // "READY"; nessun reverse mapping
```

Un **`const enum`** sostituisce normalmente gli usi con i valori in fase di compilazione, evitando l'oggetto runtime; il risultato dipende però dal transpiler e da opzioni come `preserveConstEnums`. Pubblicare `const enum` ambient in file `.d.ts` può creare incompatibilità con `isolatedModules` e differenze tra valori inlined e versione caricata: per API pubbliche valutare alternative.

Una **literal union** definisce un insieme di valori senza generare codice; un oggetto `as const` offre anche nomi accessibili a runtime, ma resta un oggetto ordinario e non applica validazione. È spesso sufficiente quando non servono semantiche specifiche di `enum`:

```ts
type Role = "admin" | "reader";
const Roles = { Admin: "admin", Reader: "reader" } as const;
type RoleFromObject = typeof Roles[keyof typeof Roles];
```

## Funzioni e overload

Un **function type** descrive parametri e risultato; una **call signature** descrive una funzione che possiede anche proprietà. Parametri `?` e valori di default permettono argomenti omessi; `...args: T[]` raccoglie i rest parameters. Il compilatore controlla le chiamate, ma non inserisce automaticamente controlli runtime sugli argomenti.

```ts
type Formatter = (value: string) => string;
interface Parser { (text: string): number; format: Formatter }
function greet(name: string, title = "Dr."): string {
  return `${title} ${name}`;
}
```

Gli **overload** pubblicano firme alternative quando gli input determinano risultati diversi; la firma che implementa il corpo non è disponibile ai chiamanti. Per casi con lo stesso risultato è spesso più semplice un parametro union. Esempi e limiti degli overload sono in [TypeScript avanzato](07-typescript-advanced.md#variance-e-compatibilità).

## Generics

Un **generic** parametrizza un tipo o una funzione: lo stesso parametro di tipo collega posizioni diverse e conserva informazioni che `any` perderebbe. TypeScript può inferirlo dagli argomenti; non è un valore disponibile a runtime.

```ts
function first<T>(items: readonly T[]): T | undefined {
  return items[0];
}
const firstScore = first([8, 9]); // number | undefined
```

La `readonly` sull'input indica che la funzione non modifica l'array; `undefined` copre il caso vuoto. Per classi e interfacce generiche, constraints, default e operatori come `keyof`, vedere [Generics e operatori di tipo](07-typescript-advanced.md#generics-e-operatori-di-tipo).

## Moduli e configurazione

`import` ed `export` organizzano il codice in **moduli ECMAScript**; `import type` importa solo dichiarazioni di tipo e viene rimosso dall'output. `tsconfig.json` definisce i file del progetto e le opzioni di controllo ed emissione: attivare `strict` è una buona base per codice nuovo, ma i tipi non sostituiscono test e validazione runtime. `target`, `module` e risoluzione degli import vanno allineati al runtime o al bundler. Per opzioni, file `.d.ts` e source maps, vedere [Configurazione e dichiarazioni](07-typescript-advanced.md#configurazione-e-dichiarazioni).
