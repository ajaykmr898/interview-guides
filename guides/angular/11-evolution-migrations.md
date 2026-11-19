# Evoluzione e migrazioni Angular

L'**evoluzione di Angular** aggiunge API (Application Programming Interface) e cambia alcuni default senza imporre la riscrittura immediata delle applicazioni esistenti. In un upgrade occorre distinguere la versione che introduce una funzione da quella in cui diventa stabile, predefinita o deprecata.

## Indice

- [Versioni e compatibilità](#versioni-e-compatibilità)
- [Migrazione standalone](#migrazione-standalone)
- [Migrazione del control flow e dei signals](#migrazione-del-control-flow-e-dei-signals)
- [Deprecazioni e schematics](#deprecazioni-e-schematics)
- [Strategia di upgrade](#strategia-di-upgrade)

## Versioni e compatibilità

**Introdotta** indica la prima disponibilità, anche sperimentale; **stabile** indica un'API supportata per la produzione; **default** descrive il comportamento senza configurazione nella versione indicata, spesso solo nei nuovi progetti. **Deprecata** significa ancora disponibile ma destinata alla rimozione; **rimossa** significa non più utilizzabile. Una nuova alternativa non rende automaticamente deprecata l'API precedente. Angular conserva le API deprecate per almeno una major successiva e le rimuove solo in una major; verificare lo stato nella [politica ufficiale](https://angular.dev/reference/releases).

| Major e data | Passaggi rilevanti |
| --- | --- |
| **v14 · giugno 2022** | Typed Reactive Forms disponibili; standalone in developer preview. |
| **v15 · novembre 2022** | Standalone stabile. |
| **v16 · maggio 2023** | Signals e interoperabilità con RxJS (Reactive Extensions for JavaScript) in developer preview; hydration e build con esbuild avanzano. |
| **v17 · novembre 2023** | `@if`/`@for`/`@switch` e `@defer` in developer preview; build `application` con esbuild e Vite disponibile e predefinita nei nuovi progetti. |
| **v18 · maggio 2024** | Control flow e `@defer` stabili; zoneless sperimentale. `@let` arriva in preview nella v18.1. |
| **v19 · novembre 2024** | Standalone predefinito; `input()`, `output()`, query signal e `@let` stabili. `linkedSignal` e `resource` sperimentali; hydration incrementale in preview. |
| **v20 · maggio 2025** | `linkedSignal`, `effect()` e hydration incrementale stabili; zoneless in developer preview. `NgIf`, `NgFor`, `NgSwitch` deprecate, ancora disponibili. `provideZonelessChangeDetection()` stabile dalla v20.2. |
| **v21 · novembre 2025** | Zoneless e Vitest default nei nuovi progetti; Signal Forms sperimentali. |
| **v22 · giugno 2026** | Signal Forms e `resource` stabili; `OnPush` default nei nuovi componenti. Con `provideClientHydration()` l'hydration incrementale diventa predefinita; lo schematic preserva il comportamento precedente negli upgrade esistenti. |

Le date e gli stati derivano dalle [note di rilascio Angular](https://blog.angular.dev/) e dalla [matrice delle versioni](https://angular.dev/reference/versions). Alla v22, v22 è in supporto attivo, v21 e v20 in Long-Term Support (LTS); le versioni fino alla v19 non sono più supportate. La compatibilità va verificata sulla *minor* effettiva per Node.js, TypeScript e RxJS: per esempio v22 richiede Node.js `^22.22.3 || ^24.15.0 || ^26.0.0`, TypeScript `>=6.0.0 <6.1.0` e RxJS `^6.5.3 || ^7.4.0`. Controllare anche peer dependencies di Angular Material, librerie e builder personalizzati prima di aggiornare.

## Migrazione standalone

Un componente **standalone** dichiara le dipendenze del template in `imports` e può essere avviato con `bootstrapApplication`; un `NgModule` raggruppa `declarations`, `imports`, `exports` e provider. I due modelli convivono: un componente standalone può importare un NgModule e un NgModule può importare/esportare componenti standalone. `importProvidersFrom()` trasporta provider di moduli legacy nell'injector applicativo o di route, non gli import del template. Vedi [architettura standalone](01-core-architecture.md#standalone-component-e-ngmodule).

| Aspetto | NgModule | Standalone |
| --- | --- | --- |
| Componenti e template | Dichiarazioni e dipendenze condivise dal modulo. | Dipendenze esplicite in `imports` del componente. |
| Bootstrap | `bootstrapModule(AppModule)`. | `bootstrapApplication(AppComponent, appConfig)`. |
| Migrazione | Conservabile dove serve; non è stato rimosso. | Preferito per nuovo codice; conversione graduale per feature. |

Lo schematic `ng generate @angular/core:standalone` richiede almeno Angular 15.2 e una build pulita. Eseguirlo in tre passaggi, verificando build e test tra i passaggi: conversione di componenti/direttive/pipe, rimozione degli NgModule non necessari, conversione del bootstrap. L'opzione `--path` limita l'area; riferimenti non analizzabili e test possono richiedere correzioni manuali. Non eliminare un modulo che definisce ancora provider o semantica di inizializzazione. Vedi la [migrazione ufficiale](https://angular.dev/reference/migrations/standalone).

## Migrazione del control flow e dei signals

I blocchi `@if`, `@for` e `@switch` sono sintassi del template integrata nel compiler e non richiedono `CommonModule`. `@for` richiede `track` per identificare gli elementi; scegliere una chiave stabile e verificare i casi in cui il vecchio `trackBy` dipendeva da mutazioni in place. `@defer` può caricare in lazy loading le dipendenze standalone del blocco in base a un trigger; `@let` definisce una variabile locale nel template. Sintassi ed esempi sono nel capitolo [template e control flow](02-components-templates-syntax.md#control-flow-moderno).

| Legacy | API moderna | Stato e decisione |
| --- | --- | --- |
| `*ngIf`, `*ngFor`, `*ngSwitch` | `@if`, `@for`, `@switch` | Blocchi stabili dalla v18; direttive deprecate dalla v20, non rimosse alla v22. Migrare i template verificando `track` e differenze di rendering. |
| `@Input()`, `@Output()`, `@ViewChild()` | `input()`, `output()`, `viewChild()` | API funzionali stabili dalla v19; decoratori ancora utilizzabili. Le letture di `input()` e delle query sono signal; `output()` restituisce un emettitore `OutputEmitterRef`, non un signal. |
| Stato locale ordinario o Observable | `signal()`, `computed()`, `effect()` | Signals introdotti in preview nella v16; primitive reattive stabili. Usare `computed()` per derivazioni, `effect()` per effetti collaterali, mantenendo RxJS per stream asincroni. |
| Stato derivato modificabile o caricamento asincrono manuale | `linkedSignal()`, `resource()` | `linkedSignal` stabile dalla v20; `resource` stabile dalla v22. Il primo resetta un valore scrivibile quando cambiano le dipendenze; il secondo espone caricamento, valore ed errore di una lettura asincrona, non è pensato per mutazioni. |
| Reactive Forms con `FormControl`/`FormGroup` | Signal Forms con `form(model)` | Reactive Forms e typed forms restano stabili; Signal Forms stabili dalla v22. Migrare per feature solo se il modello signal e la validazione a schema semplificano il codice. |

Lo schematic `ng generate @angular/core:control-flow` può convertire i template per directory con `--path`; per gli input esiste `ng generate @angular/core:signal-input-migration`. Controllare diff, tipi, template e test dopo la conversione: gli schematic non dimostrano equivalenza funzionale. Per semantica dei [Signals](05-signals-change-detection.md#signal-computed-ed-effect) e confronto tra [Reactive e Signal Forms](07-forms.md#signal-forms-e-coesistenza), usare i capitoli dedicati.

## Deprecazioni e schematics

Le API funzionali spostano la configurazione verso funzioni e provider espliciti, senza eliminare tutte le classi o i decoratori. `inject()` legge un provider solo in un *injection context*; una guard funzionale o un interceptor funzionale può usarlo durante l'esecuzione sincrona. `@Injectable()` resta l'approccio per i service che dichiarano `providedIn` o dipendenze iniettate. Una guard basata su token di classe è deprecata nelle route; `HttpInterceptor` a classe resta supportato, mentre `HttpInterceptorFn` è l'alternativa funzionale. Dettagli in [dependency injection (DI)](04-services-di-lifecycle.md#injection-context-e-inject) e [HTTP e routing](06-rxjs-http-routing.md#httpclient-e-interceptor).

| Area | Prima / alternativa | Stato e migrazione |
| --- | --- | --- |
| Form | Reactive Forms non tipizzati / typed forms | Typed forms disponibili dalla v14; il codice legacy può usare `UntypedFormControl` e affini. Passare ai tipi per porzioni, gestendo `null` e valori disabilitati. |
| Change detection | Zone.js / zoneless | Zoneless sperimentale v18, default nei nuovi progetti v21; per v20.2+ usare `provideZonelessChangeDetection()`. Un timer o una `Promise` non notifica da solo una vista zoneless: servono signal letti nel template, eventi, input o `markForCheck()`. Zone.js resta configurabile con `provideZoneChangeDetection()`. |
| Rendering server | Hydration completa / incrementale | Hydration incrementale in preview v19, stabile v20 e default con `provideClientHydration()` v22. `withIncrementalHydration()` è deprecato in v22; il migrator può aggiungere `withNoIncrementalHydration()` per conservare il comportamento precedente. Verificare markup server/client ed eventi prima di rimuovere l'opt-out. |
| Build | Builder `browser` basato su webpack / `application` con esbuild e Vite | Nuovo sistema stabile dalla v17 e default per nuove app; `browser` è deprecato, non rimosso. Dalla v18 l'upgrade propone la migrazione: controllare configurazioni webpack, output, stylesheet e codice server CommonJS. |
| Test | Karma/Jasmine / Vitest | Vitest stabile e default nei nuovi progetti dalla v21; Karma/Jasmine ancora supportati. La migrazione di una workspace esistente è sperimentale e richiede il builder `application`: verificare anche `fakeAsync` e dipendenze da Zone.js. |

`OnPush` diventa il default nella v22; il precedente comportamento eager resta selezionabile con `ChangeDetectionStrategy.Eager`. L'alias `Default` è deprecato. Negli upgrade gli schematic preservano le strategie esistenti: non attribuire un cambiamento osservato soltanto al nuovo default. Per [change detection](05-signals-change-detection.md#change-detection), [testing](09-testing.md#testbed-e-fixture) e [server-side rendering (SSR) e hydration](10-performance-security-ssr.md#ssr-e-hydration) vedi i capitoli specifici. Per builder e opzioni di migrazione vedi la [guida alla Angular CLI (Command Line Interface)](https://angular.dev/tools/cli/build-system-migration).

## Strategia di upgrade

1. Rilevare versioni di Angular, CLI, Node.js, TypeScript, RxJS, Material e librerie con peer dependencies; confrontarle con la [matrice ufficiale](https://angular.dev/reference/versions) e leggere la [Update Guide](https://angular.dev/update-guide) per la versione di partenza e di arrivo.
2. Creare un branch con stato Git pulito e una baseline di build, test e flussi critici. Aggiornare una major alla volta; per esempio, da v21 a v22: `ng update @angular/cli@^22 @angular/core@^22`. Ispezionare gli schematic automatici e la configurazione modificata.
3. Superare i gate di compilazione, test unitari, routing/form e, dove presenti, build server, hydration ed end-to-end. Migrare standalone, template, signals, builder o runner in passaggi separati, con diff e test dopo ciascuno.
4. Distribuire con monitoraggio di errori e prestazioni; per il rollback ripristinare commit e artefatto della release precedente. Se sono cambiate API backend o dati persistenti, predisporre compatibilità e rollback anche per quelle dipendenze.

`ng update` applica migrazioni automatiche e aggiorna dipendenze, ma non sostituisce i test delle integrazioni applicative. Evitare di saltare major senza verificare i percorsi supportati e gli schematic intermedi. Nessuna delle alternative moderne impone, da sola, una riscrittura completa.
