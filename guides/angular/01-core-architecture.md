# Angular core e architettura

Angular è un framework per applicazioni web basato su componenti: integra template, dependency injection, routing, form e tooling. A differenza di una library, fornisce convenzioni e servizi coordinati per l'intera applicazione; una library risolve di norma un problema più circoscritto, lasciando al progetto la composizione dell'architettura.

**AngularJS** (la linea 1.x) e Angular sono framework distinti: il primo usa scope e digest cycle, il secondo componenti, template compilati e change detection propri. Nelle applicazioni legacy possono coesistere durante una migrazione tramite `@angular/upgrade/static`, ma il codice AngularJS non diventa automaticamente codice Angular. Una **SPA (Single-Page Application)** carica una pagina iniziale e aggiorna le viste tramite routing client; Angular supporta anche rendering server e prerendering, quindi la SPA non è l'unica modalità di distribuzione. L'interfaccia è una gerarchia di componenti, ciascuno con template, comportamento e dipendenze esplicite.

## Indice

- [Workspace e bootstrap](#workspace-e-bootstrap)
- [Standalone component e NgModule](#standalone-component-e-ngmodule)
- [Struttura dell'applicazione](#struttura-dellapplicazione)
- [Dependency graph](#dependency-graph)
- [Build e configurazione](#build-e-configurazione)

## Workspace e bootstrap

La **CLI (Command-Line Interface)** Angular, invocata con `ng`, crea e gestisce un **workspace** che può contenere una o più applicazioni e librerie. `ng generate` produce artefatti tramite schematics, `ng serve` avvia il server di sviluppo e `ng build` esegue il target di build configurato; comandi e output dipendono dal builder del progetto.

| File | Responsabilità |
| --- | --- |
| `angular.json` | Configurazione del workspace e dei target per progetto (`build`, `serve`, `test`), con builder, opzioni e configurazioni alternative. |
| `src/main.ts` | Entry point dell'applicazione: avvia il componente radice. |
| `src/app/app.config.ts` | `ApplicationConfig` per i provider e le funzionalità disponibili a livello di applicazione; è un file convenzionale delle app standalone generate dalla CLI. |

Il **bootstrap** crea l'applicazione e monta il componente radice nell'elemento host. In una configurazione standalone, `bootstrapApplication` accetta un componente standalone e un `ApplicationConfig` opzionale; restituisce una `Promise<ApplicationRef>`, che può rifiutare se l'avvio fallisce. I provider applicativi, per esempio `provideRouter`, si registrano nella configurazione; provider di route o componente hanno uno scope più ristretto (si veda [Service e DI](04-services-di-lifecycle.md#service-e-provider)).

```ts
// src/app/app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideRouter } from '@angular/router';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [provideRouter(routes)],
};
```

```ts
// src/main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { App } from './app/app';
import { appConfig } from './app/app.config';

bootstrapApplication(App, appConfig).catch(console.error);
```

## Standalone component e NgModule

Un componente **standalone** dichiara in `@Component.imports` i componenti, le direttive, le pipe o gli NgModule necessari al proprio template. `imports` non registra i service nella dependency injection: per quelli servono provider, per esempio in `ApplicationConfig`, su una route o su un componente. Nelle versioni Angular correnti i componenti sono standalone per default; prima della 19 il default era `standalone: false`.

```ts
// src/app/app.ts
import { Component } from '@angular/core';
import { RouterOutlet } from '@angular/router';

@Component({ selector: 'app-root', imports: [RouterOutlet], template: '<router-outlet />' })
export class App {}
```

| Aspetto | Standalone | NgModule |
| --- | --- | --- |
| Dipendenze del template | In `imports` del componente standalone. | In `imports` del modulo; le dichiarazioni rese pubbliche sono in `exports`. |
| Dichiarazioni | Non richiedono un NgModule. | `declarations` contiene componenti, direttive e pipe con `standalone: false`; una dichiarazione appartiene a un solo NgModule. |
| Provider | Nell'applicazione, nella route o nel componente secondo lo scope voluto. | Anche in `providers` del modulo; nel bootstrap via modulo i provider dei moduli importati confluiscono nell'injector applicativo. |
| Bootstrap | `bootstrapApplication(App, appConfig)`. | Un modulo radice con `bootstrap` avviato tramite `bootstrapModule`. |

Le due forme sono interoperabili: un componente standalone può importare un NgModule esistente; un NgModule può importare ed esportare dichiarazioni standalone. Per riusare *provider* di un NgModule nell'injector applicativo o di route si può usare `importProvidersFrom`, che non sostituisce gli import del template. Angular raccomanda standalone per il nuovo codice; per migrare un'app legacy esiste lo schematic `ng generate @angular/core:standalone`, da eseguire per passi con build e test tra una fase e l'altra. Dettagli e compatibilità per versione sono nel capitolo [Evoluzione e migrazioni](11-evolution-migrations.md#migrazione-standalone).

## Struttura dell'applicazione

| Elemento | Responsabilità principale |
| --- | --- |
| **Component** | Presenta una vista e gestisce l'interazione del relativo template; la comunicazione tra componenti è trattata nel [capitolo sui template](02-components-templates-syntax.md#comunicazione-tra-componenti). |
| **Service** | Incapsula logica o accesso ai dati condivisibili, forniti tramite dependency injection; non è un elemento del template. |
| **Directive** | Aggiunge comportamento a elementi esistenti senza una vista propria; vedi [direttive strutturali e di attributo](03-directives-pipes-decorators.md#direttive-strutturali-e-di-attributo). |
| **Pipe** | Trasforma un valore per la presentazione nel template; vedi [pipe pure e impure](03-directives-pipes-decorators.md#pipe-pure-e-impure). |

Organizzare i file per **feature** tiene vicini UI, service, route e test che cambiano insieme. Una struttura minima può essere:

```text
src/app/
  app.ts                 # shell e outlet
  app.routes.ts          # confini di navigazione
  catalog/
    catalog-page.ts      # pagina della feature
    catalog.service.ts   # logica/dati della feature
    catalog.routes.ts    # route della feature, se necessarie
  shared/                # elementi riusabili senza stato della feature
```

`core/` è una convenzione facoltativa per infrastruttura applicativa trasversale, come autenticazione o logging; `shared/` raccoglie solo elementi davvero riusati, evitando un contenitore generico per tutto il codice. Un componente **smart/container** coordina dati, stato e azioni della feature; uno **presentational** riceve dati e comunica eventi con una responsabilità di visualizzazione. È una distinzione progettuale, non un tipo Angular: separarla serve quando riduce accoppiamento e semplifica test e riuso. Le feature dovrebbero dipendere da API pubbliche mirate, non dai dettagli interni di altre feature.

## Dependency graph

Il **dependency graph** statico nasce dagli `import` TypeScript e dalle dipendenze dei template; determina quali moduli possono entrare nei chunk di build. Il grafo dei **provider** descrive invece dove Angular risolve le istanze dei service: lo scope di dependency injection non implica da solo un chunk lazy. Per i principi generali di moduli e code splitting si veda [Build tools](../../topics/node-tooling/09-build-tools.md#bundling-e-code-splitting).

| Strategia | Comportamento | Trade-off |
| --- | --- | --- |
| **Eager** | Una route con `component: Page` importa direttamente il componente nella configurazione. | Disponibile subito, ma il codice entra nel grafo iniziale della configurazione. |
| **Lazy** | `loadComponent` carica un componente quando la route si attiva; `loadChildren` carica le route figlie durante il matching. | Riduce il JavaScript iniziale, ma introduce un caricamento alla prima navigazione. |

```ts
// src/app/app.routes.ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  { path: 'catalog', loadComponent: () => import('./catalog/catalog-page').then(m => m.CatalogPage) },
];
```

Un confine lazy funziona se la feature non è anche importata staticamente da codice eager: altrimenti il vantaggio sul bundle iniziale può svanire. Le dipendenze condivise possono restare in chunk comuni; verificare l'output della build. I provider dichiarati sulla route valgono per quella route e i suoi figli, indipendentemente da come è stato suddiviso il bundle. Per Router, preloading e `loadChildren` si veda [RxJS, HTTP e routing](06-rxjs-http-routing.md#router-e-lazy-loading).

## Build e configurazione

Il builder scelto in `angular.json` determina la pipeline. Nelle app generate dalla CLI corrente il builder applicativo usa esbuild; workspace più vecchi possono usare builder basati su webpack. `ng serve` usa il target di sviluppo, mentre `ng build` produce l'output da distribuire secondo la configurazione scelta: verificare entrambi quando si cambiano builder o opzioni. Per bundling, transpilation e analisi dei chunk si veda [Build tools](../../topics/node-tooling/09-build-tools.md#bundling-e-code-splitting).

Prima di un upgrade controllare la versione Angular del progetto e la matrice ufficiale di compatibilità con Node.js, TypeScript e RxJS; non assumere che le versioni più recenti siano sempre combinabili. CLI, Angular DevTools e Language Service coprono rispettivamente generazione/build, analisi runtime e supporto nell'editor. Versioni, cambiamenti di default e procedure di aggiornamento sono trattati in [Evoluzione e migrazioni](11-evolution-migrations.md#versioni-e-compatibilità).
