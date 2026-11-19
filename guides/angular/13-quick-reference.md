# Angular quick reference

Questa **quick reference Angular** è un indice operativo per ricordare definizioni, sintassi e scelte frequenti nei colloqui; gli snippet assumono Angular v22. Per semantica completa delle API (Application Programming Interface) e compatibilità del progetto seguire i link ai capitoli dedicati.

## Indice

- [Architettura](#architettura)
- [Template](#template)
- [DI e lifecycle](#di-e-lifecycle)
- [Signals e RxJS](#signals-e-rxjs)
- [Form e routing](#form-e-routing)
- [Testing performance e sicurezza](#testing-performance-e-sicurezza)

## Architettura

### Glossario A–Z

| Termine | Definizione breve e approfondimento |
| --- | --- |
| **Angular** | Framework per applicazioni web basate su componenti; [architettura](01-core-architecture.md#workspace-e-bootstrap). |
| **Bootstrap** | Avvio del componente radice e registrazione dei provider; [bootstrap](01-core-architecture.md#workspace-e-bootstrap). |
| **Component** | Classe con template e metadata `@Component`; [componenti](02-components-templates-syntax.md#metadata-e-selector). |
| **Directive** | Comportamento applicato a un elemento, senza template proprio; [direttive](03-directives-pipes-decorators.md#direttive-strutturali-e-di-attributo). |
| **Effect** | Side effect che reagisce ai signal letti; [Signals](05-signals-change-detection.md#signal-computed-ed-effect). |
| **Form** | Albero di valori, validazione e stato dei controlli; [form](07-forms.md#reactive-forms). |
| **Guard** | Decide se una navigazione può proseguire, non autorizza richieste server; [router](06-rxjs-http-routing.md#guard-e-resolver). |
| **Hydration** | Riutilizzo sul client del markup renderizzato dal server; [SSR (Server-Side Rendering)](10-performance-security-ssr.md#ssr-e-hydration). |
| **Injector** | Risolve provider secondo lo scope applicativo, di route o componente; [dependency injection (DI)](04-services-di-lifecycle.md#gerarchia-degli-injector). |
| **Lazy loading** | Carica codice di una route o vista quando serve; [routing](06-rxjs-http-routing.md#router-e-lazy-loading). |
| **NgModule** | Raggruppa dichiarazioni, import/export e provider; [standalone vs NgModule](01-core-architecture.md#standalone-component-e-ngmodule). |
| **NgRx Store** | Stato condiviso con action, reducer e selector; [NgRx](08-ngrx-state.md#store-action-reducer-e-selector). |
| **Observable** | Stream di notifiche sottoscrivibile, con errore/completion; [RxJS (Reactive Extensions for JavaScript)](../../topics/rxjs/10-rxjs.md#observable-e-subscription). |
| **OnPush** | Controlla il sottoalbero quando Angular riceve una notifica rilevante; [change detection](05-signals-change-detection.md#change-detection). |
| **Pipe** | Trasforma un valore nel template; [pipe](03-directives-pipes-decorators.md#pipe-pure-e-impure). |
| **Query** | Legge elementi o direttive della vista o del contenuto proiettato; [query](03-directives-pipes-decorators.md#query-view-e-content). |
| **Router** | Associa URL (Uniform Resource Locator), viste, guard e provider di route; [routing](06-rxjs-http-routing.md#router-e-lazy-loading). |
| **Signal** | Valore corrente leggibile sincronicamente che notifica i consumer; [Signals](05-signals-change-detection.md#signal-computed-ed-effect). |
| **Template** | Vista dichiarativa con binding e blocchi di control flow; [template](02-components-templates-syntax.md#template-binding). |
| **TestBed** | Ambiente di test con injector e fixture Angular; [testing](09-testing.md#testbed-e-fixture). |
| **ViewEncapsulation** | Strategia di isolamento degli stili del componente; [stili](03-directives-pipes-decorators.md#stili-del-componente). |
| **Workspace** | Progetto Angular CLI (Command Line Interface) configurato in `angular.json`; [struttura](01-core-architecture.md#workspace-e-bootstrap). |
| **XSS** (Cross-Site Scripting) | Esecuzione di script ostili tramite contenuti non fidati; [sanitizzazione](10-performance-security-ssr.md#xss-e-sanitizzazione). |
| **Zoneless** | Change detection senza intercettare i task con Zone.js; [zone](05-signals-change-detection.md#onpush-e-zone). |

| Scelta | Ricorda | Approfondimento |
| --- | --- | --- |
| Standalone / NgModule | `imports` sul componente e `bootstrapApplication` / `declarations` nel modulo e `bootstrapModule`; interoperano. | [Architettura](01-core-architecture.md#standalone-component-e-ngmodule) |
| Eager / lazy | Il codice eager entra nel caricamento iniziale; `loadComponent` e `@defer` introducono confini lazy. | [Lazy loading](10-performance-security-ssr.md#lazy-loading-e-bundle) |

**Mini timeline:** v14 typed forms e standalone in preview → v15 standalone stabile → v17 blocchi `@` in preview → v18 blocchi stabili → v19 standalone default e input signal stabili → v20 direttive strutturali legacy deprecate → v21 zoneless e Vitest default nei nuovi progetti → v22 Signal Forms stabili e OnPush default nei nuovi componenti. **Deprecato non significa rimosso**: verificare la [timeline e le migrazioni](11-evolution-migrations.md#versioni-e-compatibilità) prima di applicare regole della v22 a un progetto precedente.

**Componente con input signal**, in `title.component.ts` ([comunicazione](02-components-templates-syntax.md#comunicazione-tra-componenti)):

```ts
import { Component, input } from '@angular/core';

@Component({ selector: 'app-title', template: '<h2>{{ title() }}</h2>' })
export class TitleComponent {
  readonly title = input.required<string>();
}
```

## Template

| Sintassi | Promemoria |
| --- | --- |
| `{{ value }}` | Interpola testo dal componente. |
| `[disabled]="busy"` | Stato → proprietà dell'elemento o input di componente. |
| `(click)="save($event)"` | Evento → statement; `$event` è l'evento o il payload dell'output. |
| `[(value)]="count"` | Due direzioni tramite `value`/`valueChange` o `model()`; non vale per ogni proprietà. |
| `#field` | Riferimento locale di template, distinto dall'attributo HTML (HyperText Markup Language) `id`. |
| `@if (ok) { ... } @else { ... }` | Crea la vista del ramo attivo. |
| `@for (item of items; track item.id) { ... }` | Ripete una vista; `track` è obbligatorio, preferire una chiave stabile. |
| `@defer (on viewport) { ... }` | Rinvia le dipendenze standalone usate solo nel blocco; richiede `@placeholder` con un solo elemento radice o una template reference esplicita all'elemento osservato. |
| `@let name = user.name;` | Variabile locale ricalcolata, non riassegnabile. |

`@for` espone `$index`, `$count`, `$first`, `$last`, `$even`, `$odd`; `@empty` gestisce la lista vuota. `@switch`/`@case`/`@default` selezionano un caso senza fallthrough. Vedi [sintassi completa](02-components-templates-syntax.md#control-flow-moderno).

| API | Differenza da ricordare | Stato |
| --- | --- | --- |
| `@Input()` / `input()` | Il decorator aggiorna una proprietà; `input()` restituisce un `InputSignal` read-only, letto con `nome()`. `input.required<T>()` impone il binding. | Entrambi supportati; API signal stabile dalla v19. |
| `*ngIf` / `@if` | La direttiva richiede import e usa microsintassi; il blocco è integrato nel template. | `NgIf` deprecata dalla v20, non rimossa alla v22. |
| `*ngFor` / `@for` | `trackBy` è una funzione opzionale; nel blocco `track` è un'espressione obbligatoria. | `NgFor` deprecata dalla v20. |

**Pitfall:** un `@for` con chiave instabile può ricreare nodi e perdere stato/focus; un input signal non si modifica con `set()`. Vedi [migrazione dei template](11-evolution-migrations.md#migrazione-del-control-flow-e-dei-signals).

## DI e lifecycle

La **dependency injection** risolve provider dall'injector del componente, della route o dell'app. `inject()` funziona in un injection context (per esempio initializer di campo o factory), non in un callback arbitrario; lo scope del provider decide se l'istanza è condivisa. Vedi [service e gerarchia](04-services-di-lifecycle.md#gerarchia-degli-injector).

**Service con `inject()`**; `HttpClient` è iniettabile per default da Angular v21. `provideHttpClient()` configura le funzionalità HTTP; nelle versioni precedenti serve una registrazione esplicita compatibile con la versione ([HTTP (Hypertext Transfer Protocol)](06-rxjs-http-routing.md#httpclient-e-interceptor)):

```ts
import { HttpClient } from '@angular/common/http';
import { Injectable, inject } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class OrdersService {
  private readonly http = inject(HttpClient);
  list() { return this.http.get<{ id: number }[]>('/api/orders'); }
}
```

`list()` restituisce un `Observable` tipizzato; la richiesta parte alla sottoscrizione, non alla chiamata del metodo.

| Momento | Hook o API da ricordare |
| --- | --- |
| Nuovo input | `ngOnChanges` prima di `ngOnInit`; quest'ultimo gira una volta dopo l'inizializzazione degli input. |
| Vista/contenuto pronti | `ngAfterViewInit` / `ngAfterContentInit`; le query possono cambiare in seguito. |
| Dopo il render browser | `afterNextRender()` per accesso al Document Object Model (DOM); non gira sul server. |
| Distruzione | `DestroyRef.onDestroy()` o `ngOnDestroy`; pulire timer, listener e subscription durature. |

`ngDoCheck` e gli hook `*Checked` girano spesso: evitare lavoro costoso. [Lifecycle completo](04-services-di-lifecycle.md#lifecycle-dei-componenti).

## Signals e RxJS

Un **Signal** espone un valore corrente sincrono; `computed()` restituisce un signal derivato read-only, lazy e memoizzato. `set()` sostituisce, `update()` calcola dal valore precedente; non mutare in place un oggetto mantenendo la stessa reference. Vedi [Signals](05-signals-change-detection.md#signal-computed-ed-effect).

```ts
import { computed, signal } from '@angular/core';

const qty = signal(1);
const price = signal(10);
const total = computed(() => qty() * price());
qty.update(n => n + 1);
console.log(total()); // 20
```

| Scelta | Quando serve | Limite da ricordare |
| --- | --- | --- |
| Signal | Stato corrente locale e derivazioni nel template. | Nessun canale intrinseco di errore o completion. |
| Observable RxJS | Sequenza asincrona di valori, errori e completion; utile per richieste/eventi. | Richiede subscription e gestione di teardown. |
| Subject | Observable multicasting con `next()` imperativo. | Non conserva necessariamente l'ultimo valore per un nuovo subscriber. |
| NgRx Store | Stato condiviso con action, reducer, selector ed Effects. | Struttura e tracing utili nei flussi complessi, eccessivi per un toggle. |

`toSignal()` sottoscrive subito un Observable; `toObservable()` espone un signal come stream, ma le emissioni successive possono essere asincrone e coalescere. Un **SignalStore** incapsula signal, computed e metodi con scope di provider; lo **Store** classico espone transizioni tramite action/reducer e selector (`select()` → Observable, `selectSignal()` → Signal). Vedi [interoperabilità](05-signals-change-detection.md#interoperabilità-con-rxjs) e [Store vs SignalStore](08-ngrx-state.md#scelta-e-confini-dello-stato).

| Operatore RxJS | Con nuovi valori mentre l'inner è attivo | Caso tipico |
| --- | --- | --- |
| `switchMap` | Disiscrive l'inner precedente; il lavoro esterno si annulla solo se supporta teardown. | Ricerca: conta l'ultimo risultato. |
| `mergeMap` | Esegue inner concorrenti, ordine d'arrivo non garantito. | Scritture indipendenti da conservare. |
| `concatMap` | Accoda fino alla completion di ogni inner. | Scritture ordinate. |
| `exhaustMap` | Ignora nuovi valori finché l'inner non completa. | Evitare invii duplicati. |

Per errori, completion e cleanup vedi gli [operatori RxJS](../../topics/rxjs/10-rxjs.md#operatori-di-trasformazione) e l'[integrazione Angular](06-rxjs-http-routing.md#integrazione-con-rxjs).

| Strategia | Trigger e costo |
| --- | --- |
| `OnPush` / `Eager` (`Default` legacy) | `OnPush` controlla input nuovi, eventi nel sottoalbero, modifiche a signal letti o `markForCheck()`; `Eager` ricontrolla quando il traversal raggiunge la vista. `OnPush` è default dalla v22; `Default` è alias deprecato. |
| Zone.js / zoneless | Zone.js intercetta task per avviare possibili controlli; zoneless usa notifiche Angular esplicite. Zoneless è default nei nuovi progetti dalla v21; un timer da solo non aggiorna la vista. |

`OnPush` e zoneless sono scelte distinte: il primo limita quali viste controllare, il secondo come notificare Angular. Vedi [change detection e zone](05-signals-change-detection.md#onpush-e-zone).

## Form e routing

| Form | Fonte di verità | Quando preferirlo |
| --- | --- | --- |
| Reactive Forms | Albero `FormControl`/`FormGroup`, `valueChanges` Observable, validator sui controlli. | Form complessi o codice esistente; typed forms stabili. |
| Signal Forms | Writable signal del modello, `form()` e schema; `[formField]` collega i campi. | App Angular v22+ già orientate ai signal; migrare solo con un beneficio concreto. |

`setValue()` richiede la struttura completa del gruppo e segnala mismatch; `patchValue()` aggiorna campi presenti ma può ignorare chiavi sconosciute. `.value` omette i controlli disabilitati; `.getRawValue()` li include. `pristine/dirty` riguardano modifiche utente, `touched/untouched` l'interazione; `VALID`, `INVALID`, `PENDING`, `DISABLED` sono stati distinti. Un validator sincrono restituisce errori o `null`; quello asincrono restituisce Promise/Observable e lascia il controllo `PENDING` finché termina. Validare comunque sul server. Vedi [typed forms](07-forms.md#typed-forms) e [stato/validazione](07-forms.md#stato-e-accessibilità).

**Reactive Form tipizzato:**

```ts
import { FormControl, FormGroup, Validators } from '@angular/forms';

const login = new FormGroup({
  email: new FormControl('', { nonNullable: true, validators: [Validators.required, Validators.email] }),
});
login.setValue({ email: 'ada@example.com' });
```

**Route lazy:** `loadComponent` restituisce una Promise del componente e crea un confine di caricamento; guard e resolver decidono navigazione e dati, non sostituiscono autorizzazione server ([Router](06-rxjs-http-routing.md#router-e-lazy-loading)).

```ts
import { Routes } from '@angular/router';

export const routes: Routes = [{
  path: 'orders',
  loadComponent: () => import('./orders.component').then(m => m.OrdersComponent),
}];
```

## Testing performance e sicurezza

**Test del componente precedente con Vitest**, in `title.component.spec.ts`: `setInput()` segue il binding Angular; `detectChanges()` aggiorna la vista prima dell'assertion ([TestBed](09-testing.md#testbed-e-fixture)).

```ts
import { TestBed } from '@angular/core/testing';
import { expect, it } from 'vitest';
import { TitleComponent } from './title.component';

it('mostra il titolo ricevuto', () => {
  const fixture = TestBed.createComponent(TitleComponent);
  fixture.componentRef.setInput('title', 'Ordini');
  fixture.detectChanges();
  expect(fixture.nativeElement.textContent).toContain('Ordini');
});
```

| Promemoria | Pitfall / trade-off |
| --- | --- |
| Vitest è default nei nuovi progetti dalla v21; Karma/Jasmine restano supportati. | Usare il runner configurato; `fakeAsync` dipende da Zone.js. [Testing](09-testing.md#test-dei-flussi-reattivi). |
| `provideHttpClientTesting()` sostituisce il backend HTTP. | Registrare prima `provideHttpClient()` e chiudere con `HttpTestingController.verify()`. [Test HTTP](09-testing.md#test-http-e-routing). |
| `@defer` e route lazy riducono il codice iniziale; `@for (item of items; track item.id)` conserva l'identità DOM. | Misurare bundle e Core Web Vitals, non presumere un miglioramento automatico. [Performance](10-performance-security-ssr.md#lazy-loading-e-bundle). |
| SSR e hydration richiedono markup coerente fra server e browser. | Mutazioni dirette del DOM prima della hydration causano mismatch. [Rendering](10-performance-security-ssr.md#ssr-e-hydration). |
| Angular sanitizza molti binding HTML e URL; i bypass richiedono fiducia esplicita. | Non usare `bypassSecurityTrust*` su input non fidato; una guard client non autorizza API server. [Sicurezza](10-performance-security-ssr.md#xss-e-sanitizzazione). |

**File AI (Artificial Intelligence) per Angular:** scegliere il meccanismo più piccolo che descrive convenzioni o workflow realmente ricorrenti; i path dipendono dal client ([AI e Copilot Angular](12-ai-copilot-angular.md#contesto-e-istruzioni-del-progetto)).

| File o cartella | Uso rapido |
| --- | --- |
| `AGENTS.md` | Regole condivise su versione Angular, struttura, lint/build/test e scope. |
| `.github/instructions/*.instructions.md` | Regole Copilot mirate ai file tramite `applyTo`. |
| `.agents/skills/<nome>/SKILL.md` / `.github/skills/` | Workflow ripetibile per componente, form, test o migrazione. |
| `.github/agents/*.agent.md` | Ruolo Copilot dedicato, per esempio reviewer. |
| `.mcp.json` (radice del workspace) | Server MCP (Model Context Protocol) nel Copilot harness: chiave `mcpServers`; verificare tool e permessi. |

**Regola finale:** versione del progetto → API compatibili → modifica circoscritta → test e build pertinenti → diff. Per upgrade, distinguere *stabile*, *default*, *deprecato* e *rimosso*; vedi [evoluzione Angular](11-evolution-migrations.md#strategia-di-upgrade).
