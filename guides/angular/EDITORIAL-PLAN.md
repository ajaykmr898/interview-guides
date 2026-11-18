# Piano editoriale — Angular

Requisiti specifici dei capitoli di questa directory. Per le regole editoriali comuni e i criteri di accettazione, seguire [AGENTS.md](../../AGENTS.md).

## Ambito e confini

- Coprire pattern consolidati e moderni: architettura standalone e NgModules; decorator tradizionali e API basate su funzioni; template `#ref` e control flow `@`; Signals, stato derivato ed effetti; Change Detection e zoneless; interoperabilità RxJS; Reactive Forms e Signal Forms; NgRx Store e Signal Store; testing, performance e SSR; breaking changes e migrazioni.
- Usare TypeScript negli snippet Angular.
- Nei capitoli Angular spiegare il contesto locale senza ripetere la teoria generale di JavaScript, TypeScript o RxJS; collegare i dettagli canonici con link Markdown relativi.
- Per l'AI applicata ad Angular, coprire istruzioni di progetto, agenti e skill specifici, Copilot, integrazioni AI della CLI, generazione di codice, migrazioni e review nel capitolo `12-ai-copilot-angular.md`.

| Concetto generale | Integrazione Angular |
| --- | --- |
| HTML, semantica, DOM, Web APIs, HTTP, CORS | Template, binding, Router, HttpClient, sicurezza Angular |
| CSS, SCSS, layout, responsive | ViewEncapsulation, `:host`, component styles |
| JavaScript, Promise, event loop, moduli | Uso nelle API e nel lifecycle Angular |
| TypeScript, generics, `type`, `interface`, `enum`, decorator generici | Decorator Angular, typed forms, signal typing |
| Node.js, npm, bundler, ESM | CLI, builder, workspace e aggiornamenti Angular |
| Observable, Subscription, Subject, operatori RxJS | `async` pipe, HttpClient, `takeUntilDestroyed`, interoperabilità con Signals |
| LLM, VS Code/Copilot, agenti, skill, MCP | Istruzioni, agenti, skill, MCP e CLI per progetti Angular |
| SOLID, pattern, CI/CD, DevOps | Architettura feature-based, performance e testing Angular |

Il programma di ripasso è definito esclusivamente nel [README](README.md).

## `01-core-architecture.md` — Core e Architecture

**Da coprire:**
- Angular vs AngularJS; framework vs library; SPA e component-driven architecture.
- CLI, workspace e responsabilità di `angular.json`, `main.ts`, `app.config.ts`.
- Bootstrap, application configuration e provider registration.
- Standalone architecture vs NgModules: `imports`, providers, bootstrap, migration e legacy interoperability.
- Component/service/directive/pipe overview, separation of concerns.
- Feature folders, shared/core, smart vs presentational components, modularity.
- Eager vs lazy loading, dependency graph, route-level boundaries.
- Angular tooling e versione/compatibility a livello panoramico, con link al capitolo evolution.

**Snippet:** bootstrap/standalone imports e struttura feature minimale.

## `02-components-templates-syntax.md` — Components e Template Syntax

**Da coprire:**
- `@Component` metadata: selector, template/templateUrl, styles/styleUrl, imports, changeDetection.
- Component communication: property/event binding, `@Input`/`@Output`, `input()`, `output()`, `model()`.
- Interpolation `{{ }}`, property `[prop]`, event `(event)`, two-way `[(...)]`; template expressions e statements.
- `#templateRef`, `$event`, template variables e differenza tra reference variable e HTML `id`.
- Modern control flow: `@if`, `@else if`, `@else`, `@for`, `@empty`, `@switch`, `@case`, `@default`, `@let`.
- Contextual variables: `$index`, `$count`, `$first`, `$last`, `$even`, `$odd`; tracking con `track`.
- Legacy structural syntax: `*ngIf`, `*ngFor`, `ngSwitch`, `trackBy`; indicare relazioni e stato aggiornato.
- `ng-template`, `ng-container`, `ng-content`, `ngTemplateOutlet`, content projection e slots.
- Deferrable views: `@defer`, `@placeholder`, `@loading`, `@error`, trigger e prefetch.
- View/content queries: `ViewChild`, `ContentChild`, `viewChild()`, `contentChild()`; differenze e lifecycle.
- Host element bindings/listeners come concetto e confine con le direttive.

**Confronti/snippet:** `#ref` vs `id`, `@if` vs `*ngIf`, `@for` vs `*ngFor`, property vs attribute binding, esempio `@defer`.

## `03-directives-pipes-decorators.md` — Directives, Pipes e Decorators

**Da coprire:**
- Angular decorators `@Component`, `@Directive`, `@Pipe`, `@Injectable`, `@NgModule` (scopo e metadata).
- Structural vs attribute directives, built-in directives, custom directives, selector e host binding.
- Direttive vs componenti; legacy structural microsyntax e nuovo template control flow.
- Pipes built-in, custom, parametrizzazione, pure vs impure; change detection e performance.
- AsyncPipe come concetto con link a RxJS integration.
- Angular styling: ViewEncapsulation Emulated/ShadowDom/None, `:host`, `:host-context`, `::ng-deep`/alternative e stato corrente.
- Angular decorator semantics vs TypeScript decorators generici.

**Snippet:** attribute directive, pure pipe e `:host`/component styles.

## `04-services-di-lifecycle.md` — Services, DI e Lifecycle

**Da coprire:**
- Services, `@Injectable`, dependency inversion, IoC e Angular Dependency Injection.
- Injector hierarchy, providers e scopes: `providedIn:'root'`, component providers, route-level.
- `InjectionToken`, `useClass`, `useValue`, `useFactory`, `useExisting`; optional/self/skip-self.
- Constructor injection vs `inject()`, injection context e relativi limiti.
- Singleton vs nuove istanze; service boundaries, component vs service responsibilities.
- Tutti i lifecycle hooks: `ngOnChanges`, `ngOnInit`, `ngDoCheck`, `ngAfterContentInit`, `ngAfterContentChecked`, `ngAfterViewInit`, `ngAfterViewChecked`, `ngOnDestroy`.
- Render callbacks: `afterNextRender`, `afterEveryRender`; quando sono applicabili.
- `DestroyRef`, cleanup e memory leaks; lifecycle implications con RxJS e DOM APIs.

**Snippet:** service con `inject`, provider con token, cleanup via `DestroyRef`.

## `05-signals-change-detection.md` — Signals e Change Detection

**Da coprire:**
- Change detection: binding updates, Default vs OnPush, `markForCheck`, `ChangeDetectorRef`.
- Zone.js, NgZone, zoneless change detection e trigger di aggiornamento; differenze legacy/current.
- `signal()`, writable/readonly, `set`, `update`, dependency tracking.
- `computed()`: derived state, lazy evaluation, memoization e dinamismo delle dipendenze.
- `effect()`: side effects, cleanup, execution context e pitfalls (evitare propagation impropria dello stato).
- Signal inputs/queries e `model()`, `linkedSignal()`; typing e lifecycle degli input.
- Asynchronous signals: resource API e varianti verificate nella versione documentata.
- `toSignal()`/`toObservable()` overview e link al capitolo interoperabilità.
- Signal vs Observable vs NgRx: scelta in base a local state, async events e global state.
- Immutability, performance, hydration/rendering interaction a livello sintetico.

**Snippet:** `signal` + `computed`, `effect` con cleanup, signal input e aggiornamento OnPush.

## `06-rxjs-http-routing.md` — RxJS Integration, HTTP e Router

**Da coprire:**
- RxJS in Angular: HttpClient Observable, `async` pipe, `takeUntilDestroyed()`, `DestroyRef`.
- `toSignal()`, `toObservable()`, rxResource/resources dove documentato; lifecycle e error handling.
- Services e HttpClient, typed responses, `HttpParams`, headers, functional interceptors.
- API calls, cancellation, retry, cache, loading/error/success states e error handling centralizzato.
- Authentication token handling, refresh flows (overview), client- vs server-side authorization.
- Angular Router: routes, `router-outlet`, `RouterLink`, `RouterLinkActive`, navigation.
- Params vs query params, nested routes, route data, redirects, lazy `loadComponent`/`loadChildren`.
- Guards (`CanActivate`, `CanMatch`, `CanDeactivate`), resolvers, route-level providers e preloading.
- Testing HTTP/router qui solo panoramica; approfondimenti in testing.

**Snippet:** HttpClient service, interceptor funzionale, route con lazy loading, stream-to-signal.

**Confine:** non rispiegare creation operators, Subjects e flattening RxJS: link al capitolo canonico.

## `07-forms.md` — Angular Forms

**Da coprire:**
- Template-driven vs Reactive Forms; `FormsModule`, `ReactiveFormsModule`, `ngModel`.
- `FormControl`, `FormGroup`, `FormArray`, `FormRecord`, `FormBuilder`, NonNullableFormBuilder.
- Typed forms, nullability, `.value` vs `.getRawValue()`.
- `setValue` vs `patchValue`, updateOn, touched/dirty/pristine, disabled/valid/invalid/pending.
- Validators sync/async, custom validators, cross-field validation, error UX.
- `ControlValueAccessor`, custom controls, dynamic forms.
- Signal Forms: model/schema, field bindings, validation, field state e API/version status verificato.
- Signal Forms vs Reactive Forms: trade-off, migrazione e coesistenza.
- Form testing overview e link al capitolo testing.

**Snippet:** Reactive Form tipizzato, custom validator, Signal Form minimale se API stabile/verificata.

## `08-ngrx-state.md` — NgRx e State Management

**Da coprire:**
- State locale vs globale, Redux principles, immutability e predictable updates.
- NgRx Store, Actions, Reducers, Selectors, Effects; unidirectional data flow.
- Feature state, entity management, normalized state, memoized selectors.
- Side effects, error handling, async workflows, DevTools e debugging.
- ComponentStore: ruolo e trade-off; NgRx Signal Store e API attuali verificate.
- Signal Store: computed state, methods, `patchState`, state ownership e lifecycle.
- NgRx Store vs Signal Store vs service with Signals/RxJS; quando NgRx è overengineering.
- NgRx unit testing, selector/effect testing come panoramica.

**Snippet:** action/reducer/selector, Signal Store essenziale.

## `09-testing.md` — Angular Testing

**Da coprire:**
- Tipi di test: unit, component, integration, E2E, contract, smoke/regression (definizioni sintetiche e link generale).
- TestBed, fixtures, component test setup, assertions, DOM queries.
- Dependency injection nei test, providers, mocks/stubs/spies/fakes.
- Testing services, interceptors, HttpClient con strumenti correnti, Router testing.
- Testing Signals, Observable async flows, forms e component inputs/outputs.
- Async testing, fake timers, `fakeAsync`, timing, microtask/macrotask caveat.
- Vitest vs Jasmine/Karma; Angular testing defaults/version status verificati.
- Playwright vs Cypress per E2E; Angular Testing Library, Component Harnesses.
- Test isolation, reliability, coverage e testing strategy.

**Snippet:** unit test service, component fixture, HTTP testing con metodo corrente.

## `10-performance-security-ssr.md` — Performance, Security, SSR

**Da coprire:**
- CSR vs SSR vs SSG, hydration, incremental hydration, SEO e deployment implications.
- Lazy loading, preloading, deferrable views, tree shaking, bundles, build budgets.
- Signals/OnPush/zoneless per performance, list tracking, evitare unnecessary work.
- Browser rendering bottlenecks; Core Web Vitals: LCP, INP, CLS e Angular DevTools.
- Security: XSS, sanitization, `DomSanitizer`, Trusted Types/CSP dove pertinente.
- Auth vs authorization, guards non equivalgono a server-side authorization.
- Error handling, logging, observability, environment config e deploy.
- Accessibility in component design e runtime performance trade-offs.

**Snippet:** `@defer`, lazy route, esempio di performance-oriented component quando utile.

## `11-evolution-migrations.md` — Evoluzione e Breaking Changes

**Da coprire:**
- Timeline sintetica delle major Angular importanti per gli argomenti trattati (versioni e date da verificare).
- Standalone API, functional injectors/guards/interceptors, typed forms.
- Signals, signal input/output/query APIs, resource, linkedSignal, Signal Forms.
- Nuovo control flow `@if`/`@for`/`@switch`, `@defer`, `@let`.
- Zone.js vs zoneless defaults/version boundaries, rendering/hydration changes.
- Angular CLI builders, esbuild/Vite, testing runner changes, compatibility requirements.
- Per ogni cambiamento: *introduced*, *stable*, *default*, *deprecated*, *removed*, *migration guidance*.
- Migration strategy e Angular update tooling: progressive migration, compatibility check, test gates, rollback.
- Tabella legacy vs modern APIs senza chiamare "breaking" ogni novità.

**Confronti:** NgModule vs standalone, decorators vs function APIs, `*ngIf` vs `@if`, Zone.js vs zoneless, Reactive Forms vs Signal Forms.

## `12-ai-copilot-angular.md` — AI e Copilot in un progetto Angular

**Da coprire:**
- Uso di Copilot/Codex/agent workflows in repository Angular enterprise.
- Angular-specific context: architecture, naming, structure, Angular versions, lint/test/build requirements.
- `AGENTS.md` e istruzioni per Angular repository; `.github/instructions/`, agent definitions, skills per component/service/forms/tests.
- Skill per standalone component generation, typed services, Reactive/Signal Forms, tests, migrations.
- Custom agents/subagents: code review, architecture review, migration assistant, testing assistant.
- Angular CLI MCP/AI tools e official Angular Agent Skills: esistenza, comandi e supporto verificati sulla documentazione ufficiale al momento della stesura.
- Documentation context (`llms.txt` dove supportato) e uso corretto di MCP tools.
- Workflow `plan → implement → test → review`, diff inspection, scope limit e manual approval.
- Sicurezza: secrets, prompt injection, untrusted skills, permissions, supply-chain risks.
- Esempi minimi di file Markdown/config realmente supportati dal client scelto.

**Confine:** non ripetere definizioni LLM, MCP, skills/agents generiche. Fare link ai capitoli generali AI.

## `13-quick-reference.md` — Ripasso ultra-rapido Angular

**Da coprire:**
- Glossario Angular A–Z con definizioni brevissime e link ai capitoli completi.
- Tabelle comparative essenziali: standalone vs NgModules; `@Input` vs `input`; `@if` vs `*ngIf`; OnPush vs Default; Zone.js vs zoneless.
- Signals vs Observables vs Subject vs NgRx; Store vs Signal Store; `switchMap` vs altri flattening operators (solo sintesi e link).
- Template syntax cheat sheet: `{{}}`, `[]`, `()`, `[()]`, `#ref`, `$event`, `@if`, `@for`, `@defer`, `@let`.
- Forms: Reactive vs Signal Forms, `setValue` vs `patchValue`, controllo stato/validazione.
- Snippet Angular piccoli: component, service/inject, signal, route, form, test.
- Legacy vs modern APIs e mini timeline con link all'evolution chapter.
- Angular AI files/folders quick reference con link a capitolo AI.
- Best practices, pitfalls e trade-off in formato una-riga, senza duplicare il contenuto integrale.

**Obiettivo:** consultazione veloce nelle ultime 1,5 ore, non un secondo manuale.
