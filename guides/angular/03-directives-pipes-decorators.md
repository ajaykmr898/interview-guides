# Direttive pipe e decoratori Angular

Le **direttive** aggiungono comportamento a elementi o controllano la creazione di viste; le **pipe** trasformano valori nelle espressioni del template. I decorator Angular associano a classi e membri i metadata che il compiler usa per componenti, direttive, pipe, service e NgModule.

## Indice

- [Direttive strutturali e di attributo](#direttive-strutturali-e-di-attributo)
- [Pipe pure e impure](#pipe-pure-e-impure)
- [Decoratori Angular](#decoratori-angular)
- [Query view e content](#query-view-e-content)
- [API delle direttive](#api-delle-direttive)

## Direttive strutturali e di attributo

Una **attribute directive** cambia aspetto o comportamento dell'elemento host senza un template proprio. Una **structural directive** controlla se e come creare una vista incorporata a partire da un `<ng-template>`. Un componente è una direttiva specializzata che possiede un template: usarlo per un'unità di interfaccia, una direttiva per comportamento riusabile su elementi esistenti.

| Tipo | Esempi Angular | Uso e limite |
| --- | --- | --- |
| Attributo | `NgClass`, `NgStyle`, `NgOptimizedImage`; direttive custom con `selector: '[appX]'`. | Modifica l'host; per un solo binding di classe, stile o evento spesso basta la [sintassi del template](02-components-templates-syntax.md#template-binding). |
| Strutturale legacy | `*ngIf`, `*ngFor`, `*ngSwitchCase` sotto `[ngSwitch]`. | Le direttive `*` creano/rimuovono viste; `NgSwitch` seleziona i casi. `NgIf`, `NgFor` e `NgSwitch` sono deprecate dalla v20, non ancora rimosse. |
| Control flow attuale | `@if`, `@for`, `@switch`. | Blocchi del template, senza import di direttive; non sono classi `@Directive`. Vedi [confronti e `track`](02-components-templates-syntax.md#control-flow-moderno). |

La **microsyntax** `*` è una scorciatoia compilata in un `<ng-template>`: `*ngIf="ready"` sull'elemento equivale a una vista condizionale gestita dalla direttiva `NgIf`. Espressioni come `let item of items; trackBy: byId` dichiarano variabili locali e legano input della direttiva; la traduzione dei nomi segue convenzioni, non la sintassi JavaScript. Si può applicare una sola direttiva strutturale con `*` per elemento: per annidarle usare un `<ng-container>` o blocchi distinti.

Una direttiva personalizzata usa `@Directive.selector` per scegliere l'host. Qui `[appHighlight]` attiva la direttiva e ne imposta l'input; `host` aggiorna lo stile quando il valore cambia, senza manipolare direttamente il DOM (Document Object Model).

```ts
import { Directive, input } from '@angular/core';

@Directive({
  selector: '[appHighlight]',
  host: { '[style.backgroundColor]': "appHighlight() ? 'gold' : null" },
})
export class Highlight {
  appHighlight = input(false);
}
```

In un componente standalone, importare `Highlight` in `@Component.imports` e usarla come `<p [appHighlight]="selected">Testo</p>`. Per un comportamento riusabile, preferire `host` con binding dichiarativi; accedere a `ElementRef.nativeElement` solo quando necessario e con attenzione al rendering fuori dal browser.

## Pipe pure e impure

Una **pipe** riceve il valore a sinistra di `|`, eventuali parametri separati da `:`, e restituisce il risultato di `transform()`. In un componente standalone va importata in `@Component.imports`. Pipe built-in di `@angular/common` includono `DatePipe`, `CurrencyPipe`, `DecimalPipe` per formattazione locale, `UpperCasePipe`/`TitleCasePipe` per testo e `JsonPipe` per debug; non sostituiscono la trasformazione dei dati di dominio.

Una pipe custom ha `@Pipe({ name: ... })` e implementa `PipeTransform`. Questo esempio concatena un suffisso: `{{ label | suffix:'!' }}` passa `label` e `!` a `transform`, che restituisce una stringa.

```ts
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({ name: 'suffix' })
export class SuffixPipe implements PipeTransform {
  transform(value: string, suffix: string): string {
    return value + suffix;
  }
}
```

| Modalità | Quando esegue `transform()` | Trade-off |
| --- | --- | --- |
| **Pure** (default) | Quando cambia un argomento primitivo o il riferimento di un oggetto passato alla pipe. | Evita lavoro ripetuto; una mutazione interna di array/oggetto a riferimento invariato non la riattiva. |
| **Impure** (`pure: false`) | A ogni controllo di change detection della vista. | Può osservare mutazioni interne, ma il costo cresce con numero di istanze e controlli: usarla solo quando serve. |

Preferire dati immutabili o stato derivato prima di passare a una pipe impure; una pipe pure dovrebbe dipendere solo dai suoi argomenti. `AsyncPipe` è un'eccezione built-in impure: legge `Observable<T>` o `Promise<T>`, rende l'ultimo valore o `null` prima della prima emissione/risoluzione, segnala il componente per un controllo e si sgancia alla distruzione o al cambio di riferimento. Questo cleanup non cancella il lavoro di una `Promise`. Per l'[integrazione RxJS](06-rxjs-http-routing.md#integrazione-con-rxjs) vedi il capitolo dedicato.

## Decoratori Angular

| Decorator | Metadata essenziali | Ruolo |
| --- | --- | --- |
| `@Component` | `selector`, `template`/`templateUrl`, `imports`, `styles`, `encapsulation`. | Definisce una direttiva con vista. Vedi [componenti e template](02-components-templates-syntax.md#metadata-e-selector). |
| `@Directive` | `selector`, `inputs`, `outputs`, `host`, `exportAs`. | Associa comportamento a un host; non definisce una vista propria. |
| `@Pipe` | `name`, `pure`. | Rende `transform()` disponibile nelle espressioni del template. |
| `@Injectable` | `providedIn` opzionale. | Rende una classe utilizzabile nella dependency injection; la disponibilità dipende dalla registrazione dei provider. Vedi [service e provider](04-services-di-lifecycle.md#service-e-provider). |
| `@NgModule` | `declarations`, `imports`, `exports`, `providers`, `bootstrap`. | Raggruppa dichiarazioni e provider nel codice basato su moduli; vedi [standalone e NgModule](01-core-architecture.md#standalone-component-e-ngmodule). |

Questi decorator hanno significato specifico per il compiler Angular, che analizza i metadata per generare definizioni e controllare i template. Un [decorator TypeScript generico](../../topics/typescript/07-typescript-advanced.md#decoratori-e-metadati) non registra automaticamente una classe come componente, direttiva, pipe o service Angular; non confondere le due semantiche.

### Stili del componente

`@Component.styles`/`styleUrl` associano CSS (Cascading Style Sheets) al componente. `ViewEncapsulation` controlla dove si applicano tali regole; non cambia la visibilità delle classi TypeScript o dei provider.

| Modalità | Comportamento | Limite principale |
| --- | --- | --- |
| `Emulated` (default) | Angular riscrive i selettori e aggiunge attributi di scope a host e nodi della vista. | Non crea uno ShadowRoot; gli stili globali possono ancora entrare e gli stili del componente non attraversano normalmente i template dei figli. |
| `ShadowDom` | Usa Shadow DOM nativo, con template e stili in uno ShadowRoot. | Cambia anche propagazione degli eventi, proiezione e selettori; verificarne l'impatto prima di adottarlo. |
| `None` | Gli stili del componente sono globali. | Possibili collisioni: usare nomi o scope espliciti. |

```ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-panel',
  template: '<p>Contenuto</p>',
  styles: ':host { display: block; } :host(.active) p { color: navy; }',
})
export class Panel {}
```

`:host` seleziona l'elemento host; `:host(.active)` applica la regola quando l'host ha la classe. `:host-context(.dark)` permette in modalità `Emulated` di dipendere da un antenato, ma lo pseudo-selettore è deprecato nei browser moderni: preferire classi o CSS custom properties esposte sul componente per il tema. `::ng-deep` disattiva lo scope dei selettori dopo quel punto: Angular lo mantiene per compatibilità legacy e ne scoraggia il nuovo uso. Per stili di elementi proiettati o di librerie, preferire API di styling pubbliche, custom properties o CSS globale mirato; la [specificità CSS](../../topics/web/02-css-scss.md#cascade-specificità-ed-ereditarietà) resta rilevante.

## Query view e content

Una **view query** cerca elementi creati dal template del componente; una **content query** cerca elementi proiettati dal genitore. `viewChild()`/`contentChild()` restituiscono signal aggiornati con la vista, mentre `@ViewChild`/`@ContentChild` scrivono proprietà di classe: la scelta non cambia il confine tra vista e contenuto. I risultati opzionali possono essere `undefined`, specialmente sotto control flow; non leggerli come obbligatori nel costruttore. Esempi, lifecycle e differenze sono nel [capitolo sui template](02-components-templates-syntax.md#content-projection).

## API delle direttive

- **Disponibilità nel template:** una direttiva standalone va importata dal componente che la usa; una direttiva `standalone: false` va dichiarata in un NgModule ed esportata se serve ad altri moduli. `selector: '[appX]'` applica una direttiva di attributo agli elementi che hanno `appX`; il selector non è un provider.
- **Input/output:** `input()`/`output()` o `@Input`/`@Output` espongono configurazione ed eventi al template; `exportAs` permette di leggere l'istanza con `#ref="nome"`. Per la sintassi e la comunicazione vedi [componenti e template](02-components-templates-syntax.md#comunicazione-tra-componenti).
- **Host:** `host` dichiara binding di proprietà, attributi, classi, stili e listener sull'elemento cui la direttiva è applicata. `@HostBinding`/`@HostListener` restano per compatibilità, ma la guida Angular preferisce `host`; evitare collisioni con binding dello stesso elemento nel template.
- **Composizione:** `hostDirectives` applica staticamente direttive standalone all'host di un componente o di un'altra direttiva. Il loro `selector` è ignorato e input/output non diventano pubblici automaticamente: vanno esposti esplicitamente quando servono. È utile per riusare un comportamento senza una gerarchia di componenti, ma aggiunge dipendenze all'host.
