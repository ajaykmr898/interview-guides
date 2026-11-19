# Ecosistema librerie e convenzioni Angular

**Last reviewed:** 2026-11-19 (documentazione Angular 22 e dei progetti citati).

## Indice

- [Criteri di scelta e stato](#criteri-di-scelta-e-stato)
- [Routing e navigazione](#routing-e-navigazione)
- [Stato e data fetching](#stato-e-data-fetching)
- [Form e validazione](#form-e-validazione)
- [Testing](#testing)
- [Performance e sicurezza](#performance-e-sicurezza)
- [Tooling e convenzioni](#tooling-e-convenzioni)

## Criteri di scelta e stato

L'ecosistema Angular comprende API del framework, strumenti mantenuti dal team Angular e librerie esterne: la popolarità di un pacchetto non lo rende uno standard ufficiale. Questa selezione serve a motivare scelte frequenti nei colloqui, non a prescrivere uno stack unico.

| Stato | Significato |
| --- | --- |
| **Integrato** | API inclusa nei pacchetti Angular; non richiede una libreria applicativa separata. |
| **Ufficiale opzionale** | Strumento o pacchetto del team Angular, da adottare quando serve. |
| **Esterno opzionale** | Progetto distinto da Angular: verificare manutenzione, versioni supportate e costo di integrazione. |

Prima di aggiornare, confrontare [versioni Angular, Node.js, TypeScript e RxJS](https://angular.dev/reference/versions) e le dipendenze peer delle librerie esterne. Le alternative citate sotto sono scelte contestuali, non un catalogo esaustivo.

## Routing e navigazione

| Opzione e stato | Scopo e casi d'uso | Alternativa e trade-off |
| --- | --- | --- |
| [Angular Router](https://angular.dev/guide/routing) — **integrato** | URL, navigazione, parametri, guard e caricamento lazy nelle applicazioni Angular. | La navigazione locale senza URL basta per viste semplici; un router esterno aggiunge integrazione da mantenere senza un vantaggio ordinario. |

Per guard, resolver e confini del lazy loading, vedere [Router e lazy loading](06-rxjs-http-routing.md#router-e-lazy-loading).

## Stato e data fetching

| Opzione e stato | Scopo e casi d'uso | Alternativa e trade-off |
| --- | --- | --- |
| [Signals](https://angular.dev/guide/signals) — **integrato** | Stato locale o condiviso in un servizio, con valori derivati tramite `computed`. | Per flussi asincroni complessi restano utili gli Observable; Signals non fornisce da solo una cache remota. |
| [HttpClient](https://angular.dev/guide/http) — **integrato** | Richieste HTTP, interceptor, errori e test del trasporto; adatto anche alle mutazioni. | [`httpResource`](https://angular.dev/api/common/http/httpResource) riduce il codice per letture reattive, ma `HttpClient` offre controllo esplicito del flusso. |
| [`httpResource`](https://angular.dev/api/common/http/httpResource) — **integrato, Angular 22+** | Espone valore e stato di una richiesta HTTP dipendente da Signals; utile per letture legate alla vista. | `HttpClient` resta la scelta diretta per sequenze o mutazioni; verificare la versione Angular prima di adottarlo. |
| [NgRx Store](https://ngrx.io/guide/store) — **esterno opzionale** | Stato condiviso con azioni, reducer e selector; utile quando transizioni e tracciabilità coinvolgono molte parti dell'app. | Servizi con Signals richiedono meno struttura per stato limitato; Store introduce convenzioni e codice aggiuntivo. |
| [NgRx SignalStore](https://ngrx.io/guide/signals/signal-store) — **esterno opzionale** | Store basati su Signals, anche con scope locale; utili quando servono feature e convenzioni NgRx senza il modello a reducer. | Un servizio con Signals è più semplice per pochi valori; controllare la compatibilità della versione NgRx con Angular. |

Per gli Observable e le responsabilità del trasporto, vedere [HttpClient e interceptor](06-rxjs-http-routing.md#httpclient-e-interceptor); per il confine tra stato locale e globale, [Scelta e confini dello stato](08-ngrx-state.md#scelta-e-confini-dello-stato). [NgRx ComponentStore](https://ngrx.io/guide/component-store) rimane un'alternativa per stato locale basato su RxJS, soprattutto in codebase che già lo usano: non aggiungerlo solo per uniformità.

## Form e validazione

| Opzione e stato | Scopo e casi d'uso | Alternativa e trade-off |
| --- | --- | --- |
| [Reactive Forms](https://angular.dev/guide/forms/reactive-forms) — **integrato** | Form con struttura esplicita, controlli tipizzati e validazione sincrona o asincrona; adatto a form complessi o dinamici. | I [template-driven forms](https://angular.dev/guide/forms/template-driven-forms) sono più brevi per form semplici, ma rendono meno esplicito il modello e offrono meno garanzie di tipo. |
| [Signal Forms](https://angular.dev/guide/forms/signals/comparison) — **integrato, Angular 22+** | Modello a Signals e validazione a schema per nuove applicazioni che usano già Signals. | Reactive Forms resta appropriato per codebase esistenti e integrazioni basate su Observable; verificare il supporto della propria versione Angular. |

La scelta tra i tre approcci e i limiti dei validator sono spiegati in [Reactive forms](07-forms.md#reactive-forms) e [Signal Forms e coesistenza](07-forms.md#signal-forms-e-coesistenza). La validazione nel browser non sostituisce quella sul server.

## Testing

| Opzione e stato | Scopo e casi d'uso | Alternativa e trade-off |
| --- | --- | --- |
| [TestBed](https://angular.dev/guide/testing) — **integrato** | Test di componenti e servizi con dependency injection e fixture Angular. | Test di funzioni pure sono più veloci senza TestBed; usarlo quando serve il contesto Angular. |
| [Vitest tramite Angular CLI](https://angular.dev/guide/testing) — **runner esterno, integrazione ufficiale** | Esecuzione dei test nei nuovi progetti CLI. | [Karma](https://angular.dev/guide/testing/karma) resta supportato nei progetti esistenti; verificare la configurazione prima di migrare. |
| [Angular Testing Library](https://testing-library.com/docs/angular-testing-library/intro/) — **esterno opzionale** | Query per ruolo o etichetta e interazioni dal punto di vista dell'utente nei test dei componenti. | TestBed diretto evita una dipendenza; verificare la [matrice di compatibilità](https://testing-library.com/docs/angular-testing-library/version-compatibility/) prima di scegliere la versione del package. |
| [Angular CDK component harnesses](https://angular.dev/guide/testing/component-harnesses-overview) — **ufficiale opzionale** | API di test stabili per componenti UI riutilizzabili, anche al variare del DOM interno. | Per componenti semplici bastano TestBed e query accessibili; gli harness richiedono manutenzione della propria API. |
| [Playwright](https://playwright.dev/docs/intro) — **esterno opzionale** | Test end-to-end nel browser per percorsi critici, routing e integrazione reale. | I test di componente sono più rapidi e isolano meglio i guasti; i test browser richiedono ambiente e dati riproducibili. |

Per mock HTTP e verifica della navigazione vedere [Test HTTP e routing](09-testing.md#test-http-e-routing); i test browser non sostituiscono i test mirati dei singoli componenti.

## Performance e sicurezza

| Opzione e stato | Scopo e casi d'uso | Alternativa e trade-off |
| --- | --- | --- |
| [`@angular/ssr`](https://angular.dev/guide/ssr) — **ufficiale opzionale** | Server-side rendering (SSR), prerendering e rendering ibrido quando contenuto iniziale o indicizzazione lo richiedono. | Client-side rendering (CSR) evita infrastruttura server; SSR aggiunge costo operativo e richiede attenzione alla [hydration](https://angular.dev/guide/hydration). |
| [Angular DevTools](https://angular.dev/tools/devtools) — **ufficiale opzionale** | Ispezione di componenti e profiling per trovare rendering costosi durante lo sviluppo. | Le misure nel browser e i dati reali restano necessari per valutare l'esperienza utente; il profiler da solo non misura l'impatto in produzione. |
| [Sanitizzazione Angular](https://angular.dev/best-practices/security) — **integrata** | Riduce il rischio di cross-site scripting (XSS) nei binding dei template. | L'accesso diretto al Document Object Model (DOM) e il bypass della sanitizzazione richiedono revisione specifica. |
| [Content Security Policy e Trusted Types](https://angular.dev/best-practices/security) — **difese della piattaforma, opzionali** | Limitano l'esecuzione o l'iniezione di codice; utili come difesa aggiuntiva, con configurazione dell'app e del server. | Non sostituiscono i binding sicuri e richiedono verifica delle risorse consentite e del supporto del browser. |

I dettagli di rendering, profiling e superfici XSS sono in [SSR e hydration](10-performance-security-ssr.md#ssr-e-hydration) e [XSS e sanitizzazione](10-performance-security-ssr.md#xss-e-sanitizzazione).

## Tooling e convenzioni

| Opzione e stato | Scopo e casi d'uso | Alternativa e trade-off |
| --- | --- | --- |
| [Angular CLI](https://angular.dev/tools/cli) — **ufficiale** | Generazione, build, test e [`ng update`](https://angular.dev/cli/update) con migrazioni guidate; scelta predefinita per i workspace Angular. | Una toolchain personalizzata dà più controllo, ma trasferisce al team la manutenzione dell'integrazione. |
| [Angular Material](https://material.angular.dev/) e [Angular CDK](https://material.angular.dev/cdk/categories) — **ufficiali opzionali** | Componenti Material Design e primitive riutilizzabili per interazioni UI comuni. | Componenti propri o altri design system danno maggiore libertà visiva, ma richiedono lavoro su accessibilità, comportamento e manutenzione; verificare l'allineamento con la versione Angular. |
| [angular-eslint](https://github.com/angular-eslint/angular-eslint) — **esterno opzionale** | Regole ESLint per TypeScript e template Angular, utili per convenzioni condivise e CI. | ESLint senza regole Angular ha meno vincoli ma non controlla i template allo stesso modo; allineare le major di angular-eslint e Angular secondo la [matrice del progetto](https://github.com/angular-eslint/angular-eslint/blob/main/docs/ANGULAR_VERSION_SUPPORT.md). |

Le convenzioni del workspace e della build sono in [Workspace e bootstrap](01-core-architecture.md#workspace-e-bootstrap). Preferire gli strumenti già adottati dal progetto e aggiungere pacchetti esterni solo per un bisogno misurabile.
