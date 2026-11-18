# TypeScript avanzato

TypeScript avanzato combina tipi parametrizzati, trasformazioni di tipi e configurazione del compilatore per descrivere contratti riutilizzabili. I tipi vengono controllati prima dell'esecuzione e non validano i dati a runtime.

## Indice

- [Generics e operatori di tipo](#generics-e-operatori-di-tipo)
- [Conditional e mapped types](#conditional-e-mapped-types)
- [Utility types](#utility-types)
- [Template literal types](#template-literal-types)
- [Type guards e assertion](#type-guards-e-assertion)
- [Variance e compatibilità](#variance-e-compatibilità)
- [Classi e astrazione](#classi-e-astrazione)
- [Decoratori e metadati](#decoratori-e-metadati)
- [Configurazione e dichiarazioni](#configurazione-e-dichiarazioni)

## Generics e operatori di tipo

Un **type parameter** (`T`, `K`) rappresenta un tipo scelto dal chiamante o inferito dal compilatore. Funzioni, interfacce e classi generiche conservano relazioni tra input e output senza `any`; i parametri di tipo scompaiono dal JavaScript emesso. Un constraint `extends` limita i tipi ammessi, mentre un default viene usato quando non si specifica né si riesce a inferire un tipo.

```ts
function getProp<T, K extends keyof T>(object: T, key: K): T[K] {
  return object[key];
}
interface Box<T> { value: T }
class Store<T = string> implements Box<T> {
  constructor(public value: T) {}
}
const selectedName = getProp({ id: 1, name: "Ada" }, "name"); // string
```

Qui `keyof T` produce l'unione delle chiavi di `T`; `T[K]` è un **indexed access type**, cioè il tipo della proprietà selezionata. Il constraint evita chiavi inesistenti. Un default come `T = string` rende opzionale l'argomento di tipo, ma deve soddisfare un eventuale constraint. Preferire l'inferenza, esplicitando gli argomenti di tipo quando serve fissare un contratto che il compilatore non può dedurre.

`typeof` in una posizione di tipo interroga il tipo di un **valore** esistente, senza eseguire il `typeof` JavaScript. È utile con `keyof` e indexed access per derivare tipi da un oggetto invece di duplicarne la forma:

```ts
const statusCodes = { ok: 200, missing: 404 } as const;
type StatusName = keyof typeof statusCodes;          // "ok" | "missing"
type StatusCode = (typeof statusCodes)[StatusName]; // 200 | 404
```

## Conditional e mapped types

Un **mapped type** percorre le chiavi di un tipo e produce nuove proprietà. Può modificare `readonly` e `?` (`-?` rimuove l'opzionalità) o rimappare le chiavi con `as`; opera sui tipi, non copia oggetti a runtime.

```ts
type MutableRequired<T> = { -readonly [K in keyof T]-?: T[K] };
type Getters<T> = { [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K] };
type User = { readonly id?: number; name: string };
type EditableUser = MutableRequired<User>; // id e name richiesti e modificabili
```

Un **conditional type** `T extends U ? X : Y` sceglie un tipo secondo la compatibilità assegnabile. `infer` introduce un parametro di tipo ricavato dalla parte confrontata; serve, per esempio, a estrarre il valore di un array.

```ts
type ElementOf<T> = T extends readonly (infer U)[] ? U : T;
type A = ElementOf<readonly string[]>; // string
type B = ElementOf<number>;            // number
```

Se il lato sinistro del conditional è un parametro di tipo «nudo», l'operazione si **distribuisce** su ogni membro di una union: `ElementOf<string[] | number[]>` diventa `string | number`. Racchiudere entrambi i lati in tuple, `[T] extends [U]`, disattiva la distribuzione quando va valutata la union intera; trasformazioni su union grandi possono rallentare il type checking.

## Utility types

Gli utility types predefiniti trasformano tipi esistenti senza modificare i valori. Sono composizioni di mapped e conditional types; non implicano controlli a runtime.

| Utility | Risultato e impiego |
| --- | --- |
| `Partial<T>` | Tutte le proprietà opzionali, utile per aggiornamenti parziali. |
| `Required<T>` | Tutte le proprietà richieste; non agisce ricorsivamente sui campi annidati. |
| `Readonly<T>` | Proprietà non riassegnabili nel type checking; non congela oggetto o valori annidati. |
| `Pick<T, K>` | Tipo con le sole chiavi `K` di `T`. |
| `Omit<T, K>` | Tipo senza le chiavi `K`; non le rimuove da un oggetto a runtime. |
| `Record<K, V>` | Tipo di oggetto con chiavi `K` e valori `V`. |
| `Exclude<T, U>` | Membri di `T` assegnabili a `U` esclusi dalla union. |
| `Extract<T, U>` | Membri di `T` assegnabili a `U` mantenuti nella union. |
| `ReturnType<F>` | Tipo restituito da una funzione; per overload usa l'ultima firma. |
| `Parameters<F>` | Tupla dei parametri di una funzione; per overload usa l'ultima firma. |
| `Awaited<T>` | Tipo del valore dopo l'attesa di una `Promise` o di un thenable, anche annidati. |

```ts
type Patch = Partial<{ id: number; name: string }>;
type PublicUser = Pick<{ id: number; secret: string }, "id">;
type Ids = Exclude<"new" | "saved", "new">; // "saved"
type Result = Awaited<Promise<Promise<number>>>; // number
```

`Pick`/`Omit` descrivono una forma, ma non sanificano dati sensibili: costruire esplicitamente l'oggetto da esporre. `Readonly` e `as const` sono vincoli di tipo, non equivalenti a `Object.freeze`.

## Template literal types

I **template literal types** costruiscono string literal types combinando segmenti fissi e union; possono rappresentare nomi di eventi, route o chiavi correlate a proprietà. Le interpolazioni di union formano tutte le combinazioni, quindi conviene limitare i domini grandi.

```ts
type Field = "name" | "age";
type EventName = `${Field}Changed`; // "nameChanged" | "ageChanged"
type FieldFromEvent<E> = E extends `${infer F}Changed` ? F : never;
type Selected = FieldFromEvent<"nameChanged">; // "name"
```

I tipi intrinseci `Uppercase`, `Lowercase`, `Capitalize` e `Uncapitalize` trasformano le parti testuali durante il controllo dei tipi; non trasformano le stringhe a runtime.

## Type guards e assertion

Un **type guard** restringe un tipo dopo un controllo eseguito a runtime; un'**assertion** (`as T`) cambia la prospettiva del compilatore senza verificare il valore. Per il narrowing di base si veda [TypeScript core](06-typescript-core.md#union-intersection-e-narrowing). Quando i dati arrivano da API o input esterni, il guard deve controllare davvero la forma richiesta.

`as const` evita il widening dei literal e rende `readonly` proprietà e tuple inferite, anche annidate nell'espressione; non congela i dati a runtime. `satisfies T` verifica che un'espressione sia assegnabile a `T` preservandone un tipo risultante più specifico rispetto a un'annotazione `: T`. L'assertion `as T` può invece far passare una descrizione non verificata.

```ts
type Route = { path: string; secure: boolean };
const route = { path: "/", secure: false } satisfies Route;
route.secure; // tipo specifico: false
const assumed = { path: "/" } as Route; // secure non verificato
```

`satisfies` non è un validatore runtime né garantisce l'assenza di proprietà extra su valori già costruiti; su object literal nuovi si applicano i normali controlli delle proprietà in eccesso. La contestualizzazione del tipo può influire sull'inferenza: non equivale sempre al tipo ottenuto eliminando del tutto `satisfies`.

## Variance e compatibilità

TypeScript usa principalmente **compatibilità strutturale**: conta la forma dei membri, non il nome del tipo. La **variance** descrive come cambia l'assegnabilità di un contenitore quando cambia il tipo contenuto: un produttore di `T` tende a essere covariante, un consumatore di `T` controvariante. I parametri dei callback sono una posizione di consumo; con `strictFunctionTypes` le funzioni dichiarate come proprietà vengono controllate in modo controvariante, mentre i metodi conservano un'eccezione bivariante per compatibilità. Anche altri compromessi, come array mutabili, impediscono di trattare il sistema come completamente sound.

```ts
type Animal = { name: string };
type Dog = Animal & { bark(): void };
let handle: (value: Dog) => void = dog => dog.bark();
// Con strictFunctionTypes, handle non può sostituire (value: Animal) => void:
// il chiamante potrebbe passare un Animal senza bark.
```

I **typed callbacks** esprimono parametri e risultato come contratto. Nell'esempio, l'inferenza generica collega l'elemento dell'array al parametro del callback e il suo risultato al tipo restituito; `readonly T[]` accetta anche array immutabili senza promettere di modificarli.

```ts
function mapValues<T, U>(items: readonly T[], f: (item: T) => U): U[] {
  return items.map(f);
}
const lengths = mapValues(["a", "bb"], text => text.length); // number[]
```

Gli **overload** espongono più firme di chiamata con risultati correlati agli input; la firma d'implementazione non è invocabile dall'esterno. Preferire un parametro union quando i casi condividono lo stesso risultato, perché un valore già di tipo union può non corrispondere a nessun overload singolo.

```ts
function parse(value: string): number;
function parse(value: number): string;
function parse(value: string | number): string | number {
  return typeof value === "string" ? Number(value) : String(value);
}
function lengthOf(value: string | unknown[]): number { return value.length; }
```

## Classi e astrazione

Una classe TypeScript definisce una classe JavaScript con controlli statici aggiuntivi. Una **classe astratta** può condividere stato e implementazioni, ma non si può istanziare direttamente; le sottoclassi concrete devono implementare i membri astratti. Un'`interface` descrive soltanto un contratto strutturale, viene rimossa in compilazione e una classe può implementarne più di una, mentre può estendere una sola classe base.

```ts
interface Named { name: string }
abstract class Repository<T> {
  protected constructor(protected readonly items: T[]) {}
  abstract find(id: string): T | undefined;
}
class Users extends Repository<Named> {
  constructor() { super([]); }
  find(id: string) { return this.items.find(item => item.name === id); }
}
```

`public` (predefinito), `protected` e `private` limitano l'accesso nel controllo statico: rispettivamente ovunque, nelle sottoclassi o nella sola classe. Non sono barriere runtime; i [private fields JavaScript](../javascript/05-javascript-advanced-ecmascript.md#classi-e-private-fields) con `#` lo sono. `readonly` vieta riassegnazioni dopo l'inizializzazione dal punto di vista dei tipi, ma non rende immutabili gli oggetti referenziati. Scegliere un'interfaccia per un contratto senza implementazione condivisa e una classe astratta quando occorrono comportamento o stato comuni.

## Decoratori e metadati

Un **decorator** è una funzione applicata a una classe o ai suoi membri per aggiungere o sostituire comportamento durante la definizione della classe. TypeScript supporta i decorator della proposta ECMAScript moderna (Stage 3) senza `experimentalDecorators`: ricevono il valore decorato e un oggetto `context`; il comportamento dipende dal tipo di membro. Non forniscono validazione automatica dei tipi TypeScript a runtime.

Il modello **legacy** abilitato da `experimentalDecorators` segue una proposta precedente e usa firme diverse, per esempio `(target, propertyKey, descriptor)` per un metodo. I due modelli non sono intercambiabili: solo il legacy supporta parameter decorators ed è compatibile con `emitDecoratorMetadata`. Questa opzione emette alcuni metadati di design type, che possono essere letti a runtime tramite una libreria come `reflect-metadata`; i metadati sono sperimentali e non rappresentano tutti i tipi TypeScript.

I decorator Angular, per esempio `@Component`, comunicano metadati al compiler/framework Angular per configurare componenti e dependency injection. Sono API Angular, non una conseguenza automatica dei decorator standard: consultare la documentazione Angular e le opzioni del progetto prima di migrare il modello di decorator.

## Configurazione e dichiarazioni

`tsconfig.json` definisce il progetto TypeScript e come controllare, risolvere ed emettere i file. Le opzioni vanno scelte insieme al runtime o al bundler che consumerà il JavaScript.

| Opzione | Effetto principale |
| --- | --- |
| `target` | Livello della sintassi JavaScript emessa; influenza anche le librerie predefinite se `lib` non è impostato. Non aggiunge polyfill. |
| `module` | Forma dei moduli emessi o preservati; deve corrispondere all'ambiente di esecuzione. |
| `lib` | Dichiarazioni dei built-in disponibili al type checker, per esempio `DOM` o versioni ECMAScript; non aggiunge API al runtime. |
| `strict` | Abilita una famiglia di controlli rigorosi, incluso `strictNullChecks`; non è una validazione runtime. |
| `paths` | Associa specifier di import a percorsi per la risoluzione TypeScript; non riscrive gli import emessi. |
| `moduleResolution` | Determina come trovare i moduli: usare una modalità coerente con Node.js o il bundler (`node16`/`nodenext` o `bundler`). |
| `declaration` | Emette file `.d.ts` per pubblicare le firme di tipi; `declarationMap` aggiunge mappe verso le sorgenti. |

`.ts` contiene sorgente TypeScript; `.tsx` consente sintassi **JSX (JavaScript XML)** con annotazioni TypeScript. `.d.ts` contiene dichiarazioni di tipo senza implementazioni JavaScript: descrive API esterne o pacchetti e non crea valori a runtime. Una dichiarazione **ambient** (`declare`) afferma che un valore o modulo esiste altrove; dichiararlo non lo installa né lo carica.

```ts
// globals.d.ts: descrive un valore già fornito dall'ambiente.
declare const BUILD_VERSION: string;
// app.ts: il bundler/runtime deve davvero definire BUILD_VERSION.
console.log(BUILD_VERSION);
```

Le **source maps** (`sourceMap`) collegano JavaScript emesso e sorgenti TypeScript nel debugger; non cambiano i tipi. `isolatedModules` segnala costrutti che un transpiler per singolo file potrebbe tradurre male, ma non sostituisce il type checking. Se il bundler traspila i file, eseguire comunque il controllo TypeScript separatamente e configurare alias, risoluzione dei moduli, JSX e output in modo coerente nei due strumenti. Evitare di pubblicare `.d.ts` che descrivono import risolvibili solo con alias locali del bundler.
