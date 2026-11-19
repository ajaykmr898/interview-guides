# Ecosistema librerie e convenzioni React

**Last reviewed:** 2026-11-19 (React 19.3 e documentazione dei progetti citati).

## Indice

- [Criteri di scelta e stato](#criteri-di-scelta-e-stato)
- [Routing e navigazione](#routing-e-navigazione)
- [Stato e data fetching](#stato-e-data-fetching)
- [Form e validazione](#form-e-validazione)
- [Testing](#testing)
- [Performance e sicurezza](#performance-e-sicurezza)
- [Tooling e convenzioni](#tooling-e-convenzioni)

## Criteri di scelta e stato

L'**ecosistema React** comprende le API (Application Programming Interface) di `react` e `react-dom`, strumenti del team React e progetti esterni per routing, dati, form e build. Una libreria diffusa non diventa per questo uno standard React: scegliere in base al problema, al framework già adottato e al costo di manutenzione.

| Stato | Significato |
| --- | --- |
| **Integrato** | API di React o React DOM, senza un'altra libreria applicativa. |
| **Piattaforma Web** | Funzione del browser o standard HTML (HyperText Markup Language), utilizzabile anche senza React. |
| **Ufficiale opzionale** | Strumento del team React da configurare o installare quando serve. |
| **Esterno opzionale** | Progetto indipendente: preferenza della community o del team, non approvazione ufficiale di React. |

Prima di aggiungere un pacchetto, controllare release, dipendenze peer, versione di React/React DOM, supporto del framework e attività di manutenzione. Le opzioni sotto sono una selezione per colloqui, non un catalogo esaustivo; le alternative specialistiche hanno senso solo se il progetto ne richiede le capacità.

## Routing e navigazione

React non include un router: URL (Uniform Resource Locator), route annidate, loader e navigazione sono responsabilità del router o framework scelto.

| Opzione e stato | Scopo e casi d'uso | Alternativa e trade-off |
| --- | --- | --- |
| [React Router](https://reactrouter.com/start/modes) — **esterno opzionale** | Routing dichiarativo, parametri e navigazione; nelle modalità Data e Framework integra anche caricamento dati e stati pending. | [Next.js App Router](https://nextjs.org/docs/app) offre convenzioni full-stack; verificare la modalità e la versione di React Router, perché le API disponibili cambiano. |
| [Next.js App Router](https://nextjs.org/docs/app) — **framework esterno opzionale** | Route basate su file, dati e rendering server/client per applicazioni che richiedono queste capacità. | React Router o una piccola app client danno più controllo sulla composizione dello stack; Next.js aggiunge convenzioni, build e requisiti di deployment propri. |

La [guida React alla creazione di app](https://react.dev/learn/creating-a-react-app) consiglia un framework per nuove app, ma nessuno dei due è un'API core. Per i confini tra route e componenti vedere [Router come dipendenza esterna](05-data-suspense-routing.md#router-come-dipendenza-esterna).

## Stato e data fetching

Distinguere **stato client** (interazioni e preferenze locali) da **server state** (dati remoti, freschezza e revalidazione): Context distribuisce valori, ma non è una cache.

| Opzione e stato | Scopo e casi d'uso | Alternativa e trade-off |
| --- | --- | --- |
| [`useState`, `useReducer` e Context](https://react.dev/learn/managing-state) — **integrati** | Stato locale, transizioni complesse e condivisione nel sottoalbero. | Uno store esterno serve quando lo stato client condiviso e i suoi aggiornamenti superano questi confini; Context non gestisce fetch, cache o persistenza. |
| [`fetch`](https://developer.mozilla.org/en-US/docs/Web/API/Window/fetch) con [`useEffect`](https://react.dev/reference/react/useEffect) — **piattaforma Web + React integrato** | Richieste semplici in viste renderizzate nel browser, con loading, errore e cleanup espliciti. | Loader del framework o data cache evitano parte delle race e richieste ripetute; `fetch` non rifiuta per status HTTP (Hypertext Transfer Protocol) di errore e un Effect non carica dati nell'HTML server. |
| [TanStack Query](https://tanstack.com/query/latest/docs/framework/react/overview) — **esterno opzionale** | Cache e sincronizzazione del server state, con query, mutazioni e politiche di freschezza. | Loader del framework possono già possedere i dati della route; aggiungere una cache client comporta configurazione e regole di invalidazione. |
| [Redux Toolkit](https://redux.js.org/toolkit/introduction/getting-started) — **esterno opzionale, raccomandato dal progetto Redux per Redux** | Store client con azioni, reducer e strumenti prevedibili quando molte parti dell'app condividono transizioni. | Context e reducer bastano per scope limitato; Redux introduce un modello e dipendenze ulteriori. Non è una raccomandazione ufficiale React. |
| [RTK Query](https://redux.js.org/toolkit/rtk-query/overview) — **addon esterno opzionale di Redux Toolkit** | Fetch, cache e invalidazione quando l'app usa già Redux Toolkit. | TanStack Query evita di legare il server state allo store Redux; le capacità dei loader del framework possono rendere superflua un'altra cache. |

Per uno store esterno più leggero, [Zustand](https://github.com/pmndrs/zustand) è un'opzione **esterna facoltativa** con API ridotta e selettori; rispetto a Redux Toolkit offre meno convenzioni per flussi articolati e non sostituisce una strategia di cache remota. Controllarne release e peer dependencies come per ogni store. Per [ownership dello stato](04-context-state-architecture.md#local-state-context-e-server-state) e [limiti del fetch in Effect](05-data-suspense-routing.md#fetch-nel-client-e-suoi-limiti) vedere i capitoli dedicati.

## Form e validazione

| Opzione e stato | Scopo e casi d'uso | Alternativa e trade-off |
| --- | --- | --- |
| [Input React e `<form>`](https://react.dev/reference/react-dom/components/input) — **integrati + piattaforma Web** | Input controlled o uncontrolled, validazione HTML e invio semplice; da React 19 il [`form action` con funzione](https://react.dev/reference/react-dom/components/form) gestisce anche una Action. | `onSubmit` dà controllo diretto dell'evento; una libreria form aiuta quando molti campi, errori e integrazioni rendono ripetitiva la gestione manuale. Le Server Functions dipendono dal framework. |
| [React Hook Form](https://react-hook-form.com/get-started) — **esterno opzionale** | Registrazione dei campi, validazione e stato di invio in form articolati. | Form nativi o stato React sono sufficienti per pochi campi; i componenti controlled personalizzati richiedono integrazione e la versione va verificata con React e le UI library usate. |
| [Zod](https://zod.dev/) — **esterno opzionale** | Schema di validazione a runtime con inferenza TypeScript, utile per input complessi e confini di rete. | Validator HTML o funzioni locali bastano per regole semplici; richiede una versione TypeScript supportata e la validazione server resta obbligatoria. |

Per la distinzione tra [controlled, uncontrolled e Form Actions](02-state-events-forms.md#form-e-input) vedere il capitolo sui form. Zod può integrarsi con React Hook Form, ma non è una dipendenza richiesta da React.

## Testing

| Opzione e stato | Scopo e casi d'uso | Alternativa e trade-off |
| --- | --- | --- |
| [React Testing Library](https://testing-library.com/docs/react-testing-library/intro/) e [`user-event`](https://testing-library.com/docs/user-event/intro/) — **esterni opzionali** | Render di componenti e interazioni tramite ruoli, label e risultati visibili. | Test di funzioni pure non richiedono il Document Object Model (DOM); la libreria non include un test runner e richiede dipendenze compatibili. |
| [Vitest](https://vitest.dev/guide/) — **runner esterno opzionale** | Esecuzione, mock e asserzioni, specialmente in progetti con Vite. | [Jest](https://jestjs.io/docs/getting-started) è un'alternativa consolidata; scegliere in base a build esistente, trasformazioni e ambiente di test. |
| [Playwright](https://playwright.dev/docs/intro) — **esterno opzionale** | Test end-to-end (E2E) nel browser per navigazione e flussi critici con backend. | I test di componente sono più rapidi e isolano meglio i guasti; E2E richiede ambiente e dati riproducibili. |

Per [asincronia, mock e confini dei test](07-testing-accessibility-security.md#testing-dei-componenti) vedere il capitolo dedicato: il runner, la libreria di component test e lo strumento E2E hanno responsabilità diverse.

## Performance e sicurezza

| Opzione e stato | Scopo e casi d'uso | Alternativa e trade-off |
| --- | --- | --- |
| [`<Profiler>`](https://react.dev/reference/react/Profiler), [`memo`](https://react.dev/reference/react/memo) e [`lazy`](https://react.dev/reference/react/lazy) — **integrati** | Misurare render costosi, evitare alcuni re-render e caricare codice a richiesta. | Misurare prima: memoizzazione e chunk aggiungono confronti, richieste e fallback; non risolvono costi di rete o layout del browser. |
| [React Compiler](https://react.dev/learn/react-compiler/introduction) — **ufficiale opzionale, stabile** | Ottimizzazione automatica a build time per ridurre memoizzazione manuale. | `memo` e `useMemo` restano disponibili per casi mirati; il Compiler richiede integrazione nella build e rispetto delle Rules of React, non si attiva installando React. |
| [`react-dom/server`](https://react.dev/reference/react-dom/server) e [`hydrateRoot`](https://react.dev/reference/react-dom/client/hydrateRoot) — **integrati** | Rendering server (SSR, Server-Side Rendering) dell'HTML iniziale e hydration delle parti interattive. | Un framework come Next.js coordina route, dati e deployment; usare direttamente le API dà più controllo ma trasferisce questa integrazione al team. |
| [Escaping JSX (JavaScript XML) e `dangerouslySetInnerHTML`](https://react.dev/reference/react-dom/components/common#dangerously-setting-the-inner-html) — **integrati** | Il testo inserito in JSX non diventa markup eseguibile; HTML grezzo richiede trattamento separato. | Se serve mostrare HTML non attendibile, una sanitizzazione dedicata è necessaria; React non autorizza operazioni server né rende sicuro l'accesso diretto al DOM. |
| [DOMPurify](https://github.com/cure53/DOMPurify) — **esterno opzionale** | Sanitizzazione di HTML non attendibile prima dell'inserimento come markup. | Preferire testo JSX quando il markup non serve; mantenere aggiornato il sanitizer e verificare il contesto DOM in caso di rendering server. |

I dettagli su [profiler, Compiler, SSR e React Server Components](06-performance-server-rendering.md#misurazione-e-ottimizzazione) e su [XSS (Cross-Site Scripting) e autorizzazione](07-testing-accessibility-security.md#sicurezza-react) sono nei capitoli canonici. SSR non equivale a React Server Components: l'integrazione di queste ultime dipende dal framework.

## Tooling e convenzioni

| Opzione e stato | Scopo e casi d'uso | Alternativa e trade-off |
| --- | --- | --- |
| [Vite](https://vite.dev/guide/) — **esterno opzionale** | Dev server e build per app React client, anche con template TypeScript. | Un framework full-stack integra più decisioni su routing e rendering; con Vite il team sceglie queste parti e verifica i requisiti Node.js e browser della versione adottata. |
| [React Developer Tools](https://react.dev/learn/react-developer-tools) — **ufficiale opzionale** | Ispezione di componenti, props e stato, con Profiler durante lo sviluppo. | Le misure del browser e dei dati reali completano il profiling React; l'estensione non sostituisce test o monitoraggio in produzione. |
| [`eslint-plugin-react-hooks`](https://react.dev/reference/eslint-plugin-react-hooks) — **ufficiale opzionale** | Regole per ordine degli Hook, dipendenze degli Effect e problemi rilevati dal Compiler. | ESLint senza il plugin non verifica queste regole React; aggiornare plugin e preset in accordo con la versione del progetto. |

Preferire gli strumenti già integrati nel progetto e verificare le [dipendenze durante una migrazione](08-evolution-migrations.md#compatibility-e-aggiornamento-graduale). Una scelta diffusa nella community resta facoltativa finché il caso d'uso non la giustifica.
