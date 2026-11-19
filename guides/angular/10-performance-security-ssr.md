# Performance sicurezza e SSR Angular

La **performance Angular** riguarda tempo di caricamento, risposta alle interazioni e lavoro di rendering; la **sicurezza** riguarda i confini fra dati non attendibili, browser e server. Il server-side rendering (SSR) può migliorare la prima vista, ma cambia deployment, caching e vincoli del codice. Misurare prima di ottimizzare e verificare separatamente i controlli di sicurezza.

## Indice

- [Change detection e rendering](#change-detection-e-rendering)
- [Lazy loading e bundle](#lazy-loading-e-bundle)
- [SSR e hydration](#ssr-e-hydration)
- [XSS e sanitizzazione](#xss-e-sanitizzazione)
- [CSP e Trusted Types](#csp-e-trusted-types)
- [Profiling e Core Web Vitals](#profiling-e-core-web-vitals)

## Change detection e rendering

La **change detection** ricalcola i binding delle viste controllate; non è l'unico costo di rendering: JavaScript lungo, layout, paint e immagini possono dominare. Un signal letto nel template notifica Angular quando cambia; `OnPush` limita le viste controllate e zoneless evita controlli avviati dalla sola intercettazione di task. Dalla v22 `OnPush` è il default, mentre zoneless è il default dalla v21: verificare la versione del progetto e usare i trigger espliciti necessari. La semantica completa è nel capitolo [Signals e change detection](05-signals-change-detection.md#change-detection).

Usare `computed()` per derivazioni riutilizzate, mantenere brevi le espressioni di template e usare `@for (...; track item.id)` con una chiave stabile: senza identità corretta Angular può ricreare nodi e perdere stato o focus. Non mutare in place dati su cui si basano signal o input `OnPush`. Per liste molto grandi valutare virtualizzazione, ma solo dopo aver misurato dimensione del Document Object Model (DOM), costo di change detection e lavoro del browser. `track` è spiegato nel capitolo [template](02-components-templates-syntax.md#control-flow-moderno).

L'ottimizzazione non deve rompere l'accessibilità: mantenere elementi nativi, navigazione da tastiera e focus quando una vista lazy sostituisce contenuto o cambia route. Il Component Dev Kit (CDK) offre strumenti per focus e annunci; un'interfaccia che disabilita controlli o nasconde contenuto per ridurre lavoro deve restare comprensibile alle tecnologie assistive. Per la [semantica del Web](../../topics/web/01-html-web.md#accessibilità) vedi il capitolo dedicato.

## Lazy loading e bundle

Il **lazy loading** scarica codice solo quando serve; `loadComponent`/`loadChildren` creano confini di route, mentre `@defer` differisce dipendenze di una parte del template. **Preloading** anticipa il download dei chunk lazy e riduce l'attesa successiva, ma consuma banda; non precarica automaticamente i dati. Il **tree shaking** elimina export non usati quando il bundler può provarlo; un import eager della stessa dipendenza può annullare il beneficio del chunk lazy. Impostare `budgets` in `angular.json` per avvisi o errori su dimensioni iniziali e altri bundle, senza confondere dimensione del bundle con tempo di esecuzione.

```ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  { path: 'reports', loadComponent: () => import('./reports.page').then(m => m.ReportsPage) },
];
```

```html
<div aria-live="polite" style="min-height: 16rem">
  @defer (on viewport; prefetch on idle) {
    <app-chart />
  } @placeholder {
    <p>Grafico in attesa</p>
  } @loading {
    <p>Caricamento del grafico</p>
  } @error {
    <p>Grafico non disponibile</p>
  }
</div>
```

`<app-chart>` deve essere standalone e usato solo nel blocco differito perché il suo codice finisca nel chunk lazy; il trigger `on viewport` richiede un placeholder o una reference osservabile. `prefetch on idle` scarica il codice prima dell'attivazione senza renderizzarlo. Riservare spazio evita layout shift; `aria-live` annuncia il cambio di stato, ma va verificato con screen reader. Differire contenuto essenziale già visibile può peggiorare la prima vista. Per [trigger e placeholder](02-components-templates-syntax.md#control-flow-moderno) e [route lazy](06-rxjs-http-routing.md#router-e-lazy-loading) vedi i capitoli dedicati.

## SSR e hydration

| Modalità | Quando produce HTML (HyperText Markup Language) | Beneficio e costo |
| --- | --- | --- |
| CSR (Client-Side Rendering) | Il browser costruisce la pagina dopo aver caricato JavaScript. | Hosting statico semplice; contenuto iniziale e indicizzazione dipendono dal client/crawler. |
| SSR (Server-Side Rendering) | Il server produce HTML per ogni richiesta. | Contenuto iniziale aggiornato e leggibile dai crawler; richiede runtime server, capacità e caching corretti. |
| SSG (Static Site Generation, prerender) | La build genera HTML per route note. | Distribuzione statica veloce; dati disponibili alla build, niente contenuto personalizzato per richiesta. |

Angular può assegnare `RenderMode.Client`, `Server` o `Prerender` alle route server tramite `@angular/ssr`. Scegliere SSR per pagine con contenuto pubblico dinamico, SSG per pagine pubbliche stabili e CSR per viste che non richiedono HTML iniziale indicizzabile; Search Engine Optimization (SEO) dipende anche da contenuto, metadati e crawler, non dalla sola modalità. SSR aumenta costo e complessità del deploy; SSG aumenta tempi di build e richiede una strategia per route non prerenderizzate. Nelle navigazioni successive l'app normalmente continua a renderizzare nel browser.

La **hydration** riusa il DOM prodotto dal server e collega gli handler, evitando di ridisegnare l'intera pagina. `provideClientHydration()` la abilita; nel setup CLI con SSR è normalmente già configurata. Server e client devono produrre la stessa struttura: HTML non valido, condizioni basate sul solo browser o mutazioni dirette del DOM prima della hydration causano mismatch. `ngSkipHydration` è una misura temporanea per componenti incompatibili, non una correzione generale. Il codice che usa `window`/`document` deve essere limitato al browser o a render callback appropriati ([lifecycle](04-services-di-lifecycle.md#lifecycle-dei-componenti)).

L'**incremental hydration** lascia sezioni `@defer` renderizzate sul server ma interattive solo quando scatta un trigger come `hydrate on viewport` o `hydrate on interaction`; l'event replay conserva interazioni prima dell'attivazione. In Angular v22 è inclusa per default con `provideClientHydration()`; `withIncrementalHydration()` è deprecata dalla v22 e non va aggiunta nei nuovi setup. Senza incremental hydration, un normale `@defer` sul server rende il placeholder. Il trasferimento delle risposte `HttpClient` dal server al client evita richieste duplicate; escludere dati specifici dell'utente dal transfer cache e non condividere HTML personalizzato in una cache pubblica.

## XSS e sanitizzazione

**XSS (Cross-Site Scripting)** si verifica quando dati non attendibili diventano codice eseguibile nel browser. L'interpolazione `{{ value }}` produce testo escaped; Angular sanitizza binding contestuali come `[innerHTML]` e URL (Uniform Resource Locator) ordinari. I template Angular sono invece codice attendibile: non costruirli concatenando input utente. I **resource URL** che possono caricare codice non sono sanitizzabili in modo affidabile e richiedono una decisione esplicita sulla sorgente.

`DomSanitizer.sanitize(SecurityContext.HTML, value)` restituisce HTML ripulito o `null`; usarlo quando un'integrazione richiede esplicitamente una stringa sanificata, mentre nei template preferire i binding Angular. I metodi `bypassSecurityTrust...` restituiscono valori marcati attendibili e **saltano** la sanitizzazione: usarli soltanto per dati controllati e verificati, mai per testo di utenti o un URL arbitrario. Scritture con `ElementRef.nativeElement.innerHTML` o librerie che manipolano direttamente il DOM non ereditano automaticamente le protezioni dei binding e possono anche rompere la hydration. Per la [teoria XSS](../../topics/web/01-html-web.md#isolamento-e-sicurezza) vedi il capitolo Web.

**Autenticazione** identifica l'utente; **autorizzazione** decide se può accedere a una risorsa o eseguire un'operazione. Una route guard può nascondere o reindirizzare una vista, ma il client è modificabile dall'utente: il server deve verificare identità e permessi su ogni richiesta. In SSR verificare anche che la risposta e le cache non espongano dati di un utente a un altro. Per [guard e limiti](06-rxjs-http-routing.md#guard-e-resolver) vedi il Router.

## CSP e Trusted Types

Una **CSP (Content Security Policy)** è una policy inviata dal server che limita le sorgenti di script, stili e altre risorse; **Trusted Types** restringe l'uso di sink DOM pericolosi come `innerHTML`. Sono difese aggiuntive contro XSS: non sostituiscono sanitizzazione e controllo degli input. La CSP deve permettere le risorse effettivamente usate dall'app e preferire nonce o hash a `unsafe-inline`. Un nonce per gli stili Angular si passa tramite `ngCspNonce` sul root o il token `CSP_NONCE`; deve essere imprevedibile e nuovo per ogni risposta, anche quando si usa una cache. Se la build inserisce inline il CSS (Cascading Style Sheets) critico, preferire `ngCspNonce` o `security.autoCsp` al token `CSP_NONCE`.

Con l'enforcement di Trusted Types, la policy `angular` è necessaria per Angular; `angular#bundler` serve al bundler CLI per chunk lazy, mentre `angular#unsafe-bypass` è richiesta solo se l'app usa i metodi `bypassSecurityTrust...`. Abilitare soltanto le policy necessarie e testare build, caricamento lazy e librerie terze sotto la CSP reale. La compatibilità Trusted Types varia fra browser; Angular continua comunque a sanitizzare i suoi binding. Non mettere secrets nei file `environment.ts`: i file replacement sono scelti alla build e i valori finiscono nel bundle client.

## Profiling e Core Web Vitals

| Metrica | Cosa misura | Soglia indicativa “buona” |
| --- | --- | --- |
| LCP (Largest Contentful Paint) | Quando appare il più grande elemento di contenuto visibile nella viewport. | ≤ 2,5 s |
| INP (Interaction to Next Paint) | Ritardo di risposta alle interazioni fino al paint successivo. | ≤ 200 ms |
| CLS (Cumulative Layout Shift) | Instabilità visiva dovuta a spostamenti inattesi durante la visita. | ≤ 0,1 |

Le **Core Web Vitals** si valutano sui dati reali, di norma al 75° percentile, separando mobile e desktop; un test locale aiuta a diagnosticare ma non sostituisce dati sul campo. **Angular DevTools Profiler** misura change detection e lifecycle; il pannello Performance del browser mostra anche task JavaScript, style/layout, paint e rete. Cercare la causa misurata: immagini e font per LCP/CLS, handler lunghi o ricalcoli per INP, viste e liste che si aggiornano inutilmente per costo Angular. Misurare build di produzione e dispositivi realistici prima e dopo la modifica.

Trattare errori HTTP (Hypertext Transfer Protocol) attesi nel service o nella vista con stati recuperabili; `ErrorHandler` raccoglie quelli non gestiti, senza trasformarli automaticamente in un'esperienza corretta. In produzione correlare errori client/server, fallimenti di hydration e metriche per release, evitando token e dati personali nei log. Il deploy deve servire correttamente route CSR/SSG e avviare un runtime per route SSR; controllare header di cache, CSP e validazione degli host e degli header del proxy. Le configurazioni client non custodiscono secrets, che restano nel server o in un secret manager.
