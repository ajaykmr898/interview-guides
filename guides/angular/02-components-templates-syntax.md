# Componenti template e sintassi Angular

Un componente Angular associa una classe a un template: Angular valuta i binding, crea la vista e la aggiorna quando cambiano i dati rilevanti. Il template usa sintassi Angular oltre a HTML (HyperText Markup Language); la classe espone stato e metodi, mentre il metadata `@Component` definisce come il componente è usato.

## Indice

- [Metadata e selector](#metadata-e-selector)
- [Template binding](#template-binding)
- [Control flow moderno](#control-flow-moderno)
- [Content projection](#content-projection)
- [Host binding ed eventi](#host-binding-ed-eventi)
- [Comunicazione tra componenti](#comunicazione-tra-componenti)

## Metadata e selector

`@Component` identifica una classe come componente. Il `selector` individua l'elemento host nel template del genitore; ogni occorrenza crea un'istanza. Le dipendenze del template standalone vanno in `imports`, distinto dai `providers` della dependency injection (vedi [architettura standalone](01-core-architecture.md#standalone-component-e-ngmodule)).

| Metadata | Funzione |
| --- | --- |
| `selector` | Selettore dell'host, per esempio `app-profile`; sceglierlo senza collisioni con elementi esistenti. |
| `template` / `templateUrl` | Template inline / file esterno, relativo alla directory del componente. |
| `styles` / `styleUrl` | Stili inline / file esterno; l'incapsulamento degli stili è trattato nel [capitolo su direttive e pipe](03-directives-pipes-decorators.md). |
| `imports` | Componenti, direttive, pipe standalone e NgModule usati nel template standalone. |
| `changeDetection` | Strategia di controllo dei binding, per esempio `ChangeDetectionStrategy.OnPush` invece di `Default`; vedi [Change Detection](05-signals-change-detection.md#change-detection). |

```ts
import { ChangeDetectionStrategy, Component } from '@angular/core';

@Component({
  selector: 'app-greeting',
  template: '<p>Ciao, {{ name }}!</p>',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class Greeting { name = 'Ada'; }
```

## Template binding

Un **binding** collega un'espressione del template a un valore della classe o di una variabile locale; Angular lo rivaluta durante la change detection. Le espressioni sono un sottoinsieme di JavaScript, con operatori Angular come le pipe: operano nel contesto del componente e del template, senza accesso arbitrario ai globali JavaScript. Per la semantica generale di HTML e DOM (Document Object Model) vedi [HTML e Web](../../topics/web/01-html-web.md#parsing-e-rendering).

| Sintassi | Direzione e uso | Esempio |
| --- | --- | --- |
| `{{ expr }}` | Stato → testo interpolato. | `<p>{{ title }}</p>` |
| `[prop]="expr"` | Stato → proprietà DOM o input di componente/direttiva. | `<button [disabled]="busy">Salva</button>` |
| `[attr.name]="expr"` | Stato → attributo HTML quando serve l'attributo, per esempio ARIA (Accessible Rich Internet Applications) o SVG (Scalable Vector Graphics); `null`/`undefined` lo rimuovono. | `<button [attr.aria-label]="label">⋯</button>` |
| `(event)="statement"` | Evento DOM o output → metodo/aggiornamento del componente. | `<button (click)="save()">Salva</button>` |
| `[(value)]="state"` | Due direzioni: input `value` più output `valueChange`, oppure un `model()`; non è un binding valido per qualsiasi proprietà. | `<app-counter [(value)]="count" />` |

`[disabled]` imposta la proprietà DOM `disabled`, non la stringa dell'attributo; `[attr.aria-label]` chiama invece la gestione dell'attributo. Le proprietà riflettono lo stato runtime degli elementi, mentre gli attributi rappresentano i valori nel markup: scegliere il binding richiesto dall'API (Application Programming Interface) target. `[(ngModel)]` è il caso dei controlli form e richiede `FormsModule` (vedi [Form Angular](07-forms.md)).

Nei binding e nelle interpolazioni si usano **espressioni** che producono un valore, per esempio `user?.name ?? 'Ospite'`; nei listener si usano **statement**, che possono assegnare (`count = count + 1`) e accedere a `$event` (evento DOM o payload dell'output). I listener non supportano pipe. Spostare logica complessa nei metodi o nello stato derivato del componente, evitando lavoro costoso ripetuto durante i controlli.

Una **template reference variable** `#ref` punta all'elemento DOM, all'istanza di un componente oppure, su `<ng-template>`, a un `TemplateRef`; `#ref="exportAs"` seleziona una direttiva che espone quel nome. A differenza di `id`, `#ref` è visibile solo nel proprio scope di template, non crea un identificatore nel DOM e non serve come target di URL (Uniform Resource Locator) o selettore CSS.

```html
<input #searchField id="catalog-search" (input)="search(searchField.value)" />
<button (click)="search(searchField.value)">Cerca</button>
<!-- #searchField è locale al template; id identifica l'elemento nel DOM. -->
```

## Control flow moderno

I blocchi `@` controllano quali viste Angular crea, senza importare direttive strutturali. `@if` valuta una condizione truthy; `@else if` e `@else` coprono i rami alternativi. `@if (user; as currentUser)` conserva il risultato per il blocco. `@let name = user.name;` definisce invece una variabile locale ricalcolata con la sua espressione: non è riassegnabile e vale nella vista corrente e nelle viste discendenti.

```html
@if (loading) {
  <p>Caricamento…</p>
} @else if (error) {
  <p>Errore: {{ error }}</p>
} @else {
  <p>Pronto</p>
}
```

`@for` ripete una vista per ogni elemento di un iterable; `track` è obbligatorio e associa dati e nodi DOM tramite una chiave stabile. Preferire un ID univoco; `track $index` è adatto a liste statiche, mentre `track item` usa l'identità dell'oggetto e può aumentare il lavoro quando le istanze cambiano. `@empty` mostra una vista quando non ci sono elementi. Nel blocco sono disponibili `$index` (indice), `$count` (totale), `$first`/`$last` (estremi), `$even`/`$odd` (parità); `let i = $index` crea un alias utile nei cicli annidati.

```html
<ul>
  @for (item of items; track item.id; let i = $index) {
    <li>{{ i + 1 }}. {{ item.name }}</li>
  } @empty {
    <li>Nessun elemento</li>
  }
</ul>
```

`@switch` confronta con `===` il valore con ogni `@case`; mostra il caso corrispondente, senza fallthrough, oppure `@default` se nessun caso corrisponde. Se manca `@default` e non c'è una corrispondenza, non renderizza nulla. Più `@case` consecutivi possono condividere un blocco.

```html
@switch (status) {
  @case ('ready') { <p>Pronto</p> }
  @case ('pending') { <p>In attesa</p> }
  @default { <p>Stato sconosciuto</p> }
}
```

| Sintassi attuale | Sintassi strutturale legacy | Differenza da ricordare |
| --- | --- | --- |
| `@if (ok) { ... } @else { ... }` | `*ngIf="ok; else other"` con `<ng-template #other>` | Il blocco è sintassi del template; la direttiva va importata e crea un template implicito. |
| `@for (item of items; track item.id) { ... }` | `*ngFor="let item of items; trackBy: byId"` | `track` è un'espressione obbligatoria; `trackBy` è una funzione opzionale, altrimenti conta l'identità dell'oggetto. |
| `@switch` / `@case` / `@default` | `[ngSwitch]` / `*ngSwitchCase` / `*ngSwitchDefault` | `@switch` non ha fallthrough; `ngSwitch` può renderizzare più casi corrispondenti. |

`NgIf`, `NgFor` e `NgSwitch` sono deprecate dalla v20 ma ancora documentate; usare i blocchi nel nuovo codice e consultare [evoluzione e migrazioni](11-evolution-migrations.md#migrazione-del-control-flow-e-dei-signals) prima di aggiornare template legacy.

Una **deferrable view** `@defer` rinvia il caricamento delle dipendenze standalone usate solo nel blocco. Il trigger predefinito è `on idle`; `on viewport`, `on interaction`, `on hover`, `on timer(...)`, `on immediate` e `when condizione` permettono altri tempi. `prefetch on ...` scarica il codice prima, senza renderizzarlo; `@placeholder`, `@loading` e `@error` mostrano rispettivamente attesa iniziale, caricamento ed errore. Questi sottoblocchi sono eager; dipendenze non standalone o usate anche fuori da `@defer` non vengono differite.

```html
@defer (on interaction; prefetch on idle) {
  <app-chart />
} @placeholder {
  <button>Mostra grafico</button>
} @loading {
  <p>Caricamento…</p>
} @error {
  <p>Grafico non disponibile</p>
}
```

Nell'esempio `app-chart` è un componente standalone usato solo nel blocco; il bottone del placeholder è il trigger di interazione. Per non peggiorare accessibilità e layout, dare al contenuto differito spazio e feedback adeguati. Il prefetch non anticipa il rendering.

## Content projection

**Content projection** inserisce nel template di un componente il contenuto dichiarato dal suo genitore. `<ng-content>` è un placeholder compilato da Angular, simile per scopo a uno slot: `select` distribuisce i figli tra slot secondo un selettore CSS (Cascading Style Sheets); un `<ng-content>` senza `select` riceve i figli rimasti. Il contenuto proiettato resta nella vista del genitore, non nella view del componente ricevente.

```ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-card',
  template: '<h2><ng-content select="[cardTitle]" /></h2><ng-content />',
})
export class Card {}
```

Il genitore può usare `<app-card><span cardTitle>Titolo</span><p>Corpo</p></app-card>`. I selettori di proiezione sono statici; non mettere `<ng-content>` sotto `@if` per cercare di creare condizionalmente i figli, perché Angular crea comunque il contenuto proiettato.

| Costrutto | Uso e limite |
| --- | --- |
| `<ng-template>` | Dichiara un frammento non renderizzato finché una direttiva o `ViewContainerRef` non ne crea una vista; `#fragment` è un `TemplateRef`. |
| `<ng-container>` | Raggruppa nodi o ospita direttive senza aggiungere un elemento DOM. |
| `NgTemplateOutlet` | Renderizza un `TemplateRef` nella posizione dell'outlet, anche passando un `context`; in un componente standalone va importato da `@angular/common`. |
| `<ng-content>` | Proietta figli del componente in slot; non è un frammento renderizzabile a richiesta. |

```ts
import { NgTemplateOutlet } from '@angular/common';
import { Component } from '@angular/core';

@Component({
  imports: [NgTemplateOutlet],
  template: `<ng-template #row let-name><p>{{ name }}</p></ng-template>
             <ng-container *ngTemplateOutlet="row; context: { $implicit: 'Ada' }" />`,
})
export class RowExample {}
```

Le **query** leggono riferimenti a elementi, componenti o direttive. `viewChild()` cerca nella vista definita dal template del componente; `contentChild()` cerca nel contenuto proiettato. Restituiscono signal che si aggiornano quando cambia la vista: una query senza risultato legge `undefined`, per esempio se un `@if` rimuove il nodo; la variante `.required()` fallisce se manca il risultato. Le query non attraversano i confini di altri componenti.

Le alternative `@ViewChild` e `@ContentChild` rimangono supportate e aggiornano proprietà di classe; i risultati sono disponibili rispettivamente entro `ngAfterViewInit` e `ngAfterContentInit`, non nel costruttore. Le query signal possono essere lette dopo che la rispettiva vista è stata creata, ma non presumere un risultato obbligatorio prima del rendering. Per il [lifecycle](04-services-di-lifecycle.md#lifecycle-dei-componenti) e i [signal](05-signals-change-detection.md#signal-computed-ed-effect) vedi i capitoli dedicati.

## Host binding ed eventi

L'**host element** è l'elemento che istanzia il componente. Il metadata `host` può associargli proprietà, attributi, classi, stili e listener, con la stessa sintassi dei binding del template: per esempio `host: { '[attr.aria-expanded]': 'open()', '(keydown.enter)': 'toggle()' }`. Si usa quando il componente deve esporre stato e comportamento sul proprio elemento, senza aggiungere un wrapper. Evitare binding dinamici concorrenti alla stessa proprietà tra host, direttive e template del genitore.

`@HostBinding` e `@HostListener` sono alternative legacy ancora supportate; la guida Angular preferisce `host`. Una [direttiva di attributo](03-directives-pipes-decorators.md#direttive-strutturali-e-di-attributo) usa lo stesso meccanismo per aggiungere comportamento a un elemento esistente, mentre un componente possiede anche un template.

## Comunicazione tra componenti

Il genitore passa dati al figlio con `[input]="expr"`; il figlio comunica eventi con `(output)="handler($event)"`. Gli output Angular non fanno bubbling nel DOM. Preferire questo flusso esplicito quando il figlio presenta dati e segnala intenzioni, lasciando al proprietario dello stato la decisione di aggiornarlo.

| API nel figlio | Lettura o emissione | Stato |
| --- | --- | --- |
| `@Input() value: T` | Proprietà aggiornata da Angular; può essere richiesta con `@Input({ required: true })`. | Decorator tradizionale, supportato. |
| `input<T>()` / `input.required<T>()` | `InputSignal<T \| undefined>` / `InputSignal<T>` read-only, letti con `value()`; `required` è verificato nel template. | API signal consigliata per nuovo codice. |
| `@Output() changed = new EventEmitter<T>()` | `changed.emit(value)` invia un evento al genitore. | Decorator tradizionale, supportato. |
| `output<T>()` | `OutputEmitterRef<T>` con `emit(value)`; non è un Signal. | API funzionale consigliata per nuovo codice. |
| `model<T>(initial)` | `ModelSignal<T>` scrivibile con `set`/`update`; genera anche l'output `valueChange` per una proprietà chiamata `value`. | Per componenti che modificano direttamente il proprio valore. |

```ts
import { Component, input, output } from '@angular/core';

@Component({ selector: 'app-save', template: '<button (click)="saved.emit()">{{ label() }}</button>' })
export class SaveButton {
  label = input.required<string>();
  saved = output<void>();
}
```

Il genitore importa `SaveButton` e usa `<app-save [label]="name" (saved)="save()" />`: `label` è un input in sola lettura nel figlio, `saved` emette un evento senza modificare implicitamente lo stato del genitore. Per un controllo che modifica il proprio valore, `model()` permette invece `[(value)]="count"`; se `count` è un writable signal, nel binding si passa il signal, non `count()`.
