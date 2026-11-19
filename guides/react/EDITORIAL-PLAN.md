# Piano editoriale — React

Requisiti dei capitoli di questa directory. Seguire le regole comuni in [AGENTS.md](../../AGENTS.md). Tutte le voci **Da coprire** sono obbligatorie; P0/P1/P2 indicano la profondità, non l'esclusione.

## Ambito e confini

- Trattare React e `react-dom` come oggetto della guida; distinguere le API core dalle scelte di router, framework, data cache e state library. Verificare lo stato delle API rispetto alla versione documentata prima di completare i capitoli.
- Per linguaggio, event loop e DOM usare [JavaScript](../../topics/javascript/03-javascript-basics.md) e [asincronia e DOM](../../topics/javascript/04-javascript-async-dom.md); per i tipi usare [TypeScript](../../topics/typescript/06-typescript-core.md).
- Per HTML, accessibilità, HTTP e sicurezza browser usare [HTML e Web](../../topics/web/01-html-web.md); per CSS e build usare [CSS e SCSS](../../topics/web/02-css-scss.md) e [Build tools](../../topics/node-tooling/09-build-tools.md).
- Per categorie di test e architettura generale usare [Engineering](../../topics/engineering/15-architecture-devops.md); per il contratto delle API usare [API RESTful](../../topics/api/16-restful-apis.md). Nei capitoli spiegare soltanto l'integrazione specifica di React.

## `01-components-jsx-rendering.md` — Componenti JSX e rendering React

**Da coprire:**
- **P0 — Modello React e ciclo di rendering / Componenti funzione e props:** React come library dichiarativa, componenti funzione, elementi, props, `children`, flusso dati unidirezionale e purezza del render.
- **P0 — Modello React e ciclo di rendering / Root e mount dell'applicazione:** ruolo di `react-dom`, `createRoot` e container; distinguere l'avvio client dal rendering server.
- **P0 — Modello React e ciclo di rendering / Render, commit e riconciliazione:** differenza tra chiamata del componente, commit sul DOM e browser paint; aggiornamento e conservazione dell'identità.
- **P0 — JSX e composizione / Sintassi JSX e attributi:** espressioni, attributi, `className`, `htmlFor`, differenze essenziali da HTML e escaping del testo.
- **P0 — Liste e identità / Rendering condizionale e liste; Key e stato:** condizioni, `map`, key stabili, riuso/reset dello stato e limiti degli indici come key.
- **P1 — JSX e composizione / Children, Fragment e composizione:** composizione delle UI e confini delle props; **Portali e albero React** distingue l'albero React dal posizionamento nel DOM.
- **P2 — Modello React e ciclo di rendering / StrictMode nel render:** controlli di sviluppo e loro effetto sulle funzioni pure; dettagli degli Effect nel capitolo 03.

## `02-state-events-forms.md` — Stato eventi e form React

**Da coprire:**
- **P0 — Stato e aggiornamenti / useState, snapshot e batching:** identità dello stato, setter funzionale, snapshot per render e aggiornamenti raggruppati.
- **P0 — Stato e aggiornamenti / Oggetti, array e stato derivato:** aggiornamenti immutabili, evitare stato ridondante; **Lifting state up e reset** copre ownership e key.
- **P0 — Eventi e interazioni / Event handler e propagazione:** passaggio di callback, eventi React e distinzione tra handler ed esecuzione durante il render.
- **P0 — Form e input / Input controlled e uncontrolled:** `value`/`onChange` e `defaultValue`/ref, scelta e limiti; **Validazione e invio** collega la semantica HTML canonica.
- **P1 — Eventi e interazioni / Comportamento predefinito e stato UI:** `preventDefault`, propagazione e aggiornamenti associati all'interazione.
- **P1 — Form e input / Form Actions e stato pending:** `action` funzione, `FormData`, `useActionState`, `useFormStatus` e differenze rispetto a `onSubmit`, con stato delle API verificato.
- **P2 — Form e input / Casi avanzati dei form:** reset, invii concorrenti e confine tra validazione client e server.

## `03-effects-refs-hooks.md` — Effect ref e Hook React

**Da coprire:**
- **P0 — Regole degli Hook / Ordine di chiamata e custom Hook:** chiamata al top level, identità per posizione, composizione e limiti dei custom Hook.
- **P0 — Effect e sincronizzazione / useEffect, dipendenze e cleanup:** sincronizzazione con sistemi esterni, dipendenze reattive, cleanup, race e stale closure.
- **P0 — Effect e sincronizzazione / Quando evitare un Effect:** calcoli nel render ed eventi negli handler; **StrictMode e riesecuzione** distingue i controlli di sviluppo dal comportamento di produzione.
- **P1 — Ref e escape hatch / useRef e DOM:** valori mutabili senza re-render, accesso al DOM e limiti della lettura durante il render.
- **P1 — Ref e escape hatch / Ref ai componenti e API imperative:** ref prop e `useImperativeHandle`, con differenze tra versioni verificate.
- **P2 — Effect e sincronizzazione / useLayoutEffect:** momento di esecuzione e costo rispetto a `useEffect`; **Ref e escape hatch / Callback ref** per setup e cleanup DOM mirati.

## `04-context-state-architecture.md` — Context e architettura dello stato React

**Da coprire:**
- **P0 — Stato condiviso / Props, Context e Provider:** prop drilling, `createContext`, `useContext`, scope, fallback e re-render dei consumer.
- **P0 — Stato condiviso / useReducer e composizione:** reducer puro, dispatch, scelta rispetto a `useState` e combinazione con Context.
- **P1 — Architettura dello stato / Local state, Context e server state:** ownership, persistenza e differenza tra stato UI, cache remota e store globale.
- **P1 — Architettura dello stato / Store esterni e useSyncExternalStore:** contratto di sottoscrizione, snapshot e quando preferire una libreria dedicata.
- **P1 — Confini dei componenti / Error Boundary:** errori intercettati nel render e limiti rispetto a eventi e operazioni asincrone.
- **P2 — Confini dei componenti / API dei componenti e provider:** composizione, provider annidati e trade-off di astrazioni riusabili.

## `05-data-suspense-routing.md` — Dati Suspense e routing React

**Da coprire:**
- **P0 — Dati asincroni / Fetch nel client e suoi limiti:** loading/error/success, cleanup e race; distinguere `useEffect` da data fetching integrato in framework o cache.
- **P0 — Suspense e priorità / Boundary Suspense e fallback:** significato della sospensione, boundary e limiti delle sorgenti supportate; `Suspense` non abilita da solo il fetch in un Effect.
- **P1 — Dati asincroni / use e risorse asincrone:** lettura di Promise e Context, eccezione alle regole di chiamata degli Hook, vincoli e integrazione con Suspense secondo la versione verificata.
- **P1 — Suspense e priorità / Transition e deferred value:** `useTransition`, `useDeferredValue`, aggiornamenti urgenti e non urgenti; limiti sugli input controllati.
- **P1 — Routing / Router come dipendenza esterna:** navigazione, parametri, route annidate e caricamento dati; distinguere API React da API del router/framework scelto.
- **P2 — Dati asincroni / Aggiornamenti ottimistici:** `useOptimistic`, rollback e coerenza con la risposta server; **Routing / Confini delle route** copre lazy loading e gestione degli errori a livello introduttivo.

## `06-performance-server-rendering.md` — Performance e rendering server React

**Da coprire:**
- **P0 — Misurazione e ottimizzazione / Profiler e cause dei re-render:** misurare prima di ottimizzare, distinguere render React, commit e costo del DOM.
- **P0 — Rendering server / CSR, SSR e hydration:** HTML iniziale, `hydrateRoot`, mismatch e responsabilità del framework; distinguere SSR da React Server Components.
- **P0 — React Server Components / Server e Client Components:** confine server/client, direttiva `use client`, interattività e serializzazione delle props.
- **P1 — Misurazione e ottimizzazione / memo, useMemo e useCallback:** condizioni d'uso e costi; **React Compiler** copre ottimizzazione automatica, adozione e stato corrente verificato.
- **P1 — Caricamento / React.lazy e code splitting:** boundary `Suspense`, fallback e trade-off del bundle.
- **P1 — React Server Components / Server Functions:** `use server`, differenze dalle Server Components, requisiti del framework e autorizzazione sul server.
- **P2 — Rendering server / Streaming e prerendering:** API e limiti del rendering progressivo, con stato/versione verificati.

## `07-testing-accessibility-security.md` — Testing accessibilità e sicurezza React

**Da coprire:**
- **P0 — Testing dei componenti / Render e interazioni osservabili:** React Testing Library e interazioni utente, query semantiche, stato e callback; distinguere test React dalle categorie generali.
- **P0 — Accessibilità nei componenti / Semantica, label e tastiera:** JSX semantico, controlli accessibili e gestione del focus nelle UI dinamiche.
- **P1 — Testing dei componenti / Asincronia, act e isolamento:** aggiornamenti asincroni, mock dei confini esterni, StrictMode e affidabilità dei test.
- **P1 — Testing dei componenti / Routing, form ed error boundary:** verifiche dei flussi e responsabilità di component vs integration test.
- **P1 — Sicurezza React / Escaping e HTML non attendibile:** escaping JSX, `dangerouslySetInnerHTML`, sanitizzazione e limiti della protezione React.
- **P1 — Sicurezza React / Confini server e autorizzazione:** input e permessi nelle Server Functions; nessun controllo solo client garantisce l'accesso.
- **P2 — Accessibilità nei componenti / Annunci e portali:** aggiornamenti dinamici e focus; **Testing dei componenti / E2E** copre i flussi critici senza duplicare la teoria generale.

## `08-evolution-migrations.md` — Evoluzione e migrazioni React

**Da coprire:**
- **P0 — API legacy e moderne / Componenti di classe e Hook:** lifecycle di classe, `setState`, error boundary e migrazione senza equivalenze meccaniche.
- **P0 — API legacy e moderne / Root e rendering moderno:** `createRoot`/`hydrateRoot` e differenze dalle API legacy; indicare introdotto, deprecated o rimosso solo dopo verifica.
- **P1 — Evoluzione delle API / Form, ref e rendering server:** Actions, ref come prop, Server Components e differenze tra versioni/supporto dei framework.
- **P1 — Strategia di migrazione / Compatibility e aggiornamento graduale:** dipendenze, codemod, regole degli Hook, test e rollback; distinguere feature introdotta, stabile e default.
- **P2 — Strategia di migrazione / Casi legacy e interoperabilità:** classi esistenti, librerie non aggiornate e migrazione incrementale senza riscrittura totale.
