# Evoluzione e migrazioni React

Una migrazione React aggiorna le API (Application Programming Interface), le integrazioni e il comportamento osservabile di un'app senza presumere che una sintassi nuova sia equivalente a quella precedente. Lo stato delle API dipende dalla versione installata di `react`, `react-dom` e delle librerie collegate.

## Indice

- [API legacy e moderne](#api-legacy-e-moderne)
- [Evoluzione delle API](#evoluzione-delle-api)
- [Strategia di migrazione](#strategia-di-migrazione)

## API legacy e moderne

### Componenti di classe e Hook

I **componenti di classe** estendono `Component`, leggono `this.props` e `this.state` e restituiscono nodi React da `render()`. Sono ancora **supportati in React 19**, ma per nuovo codice sono preferiti i componenti funzione con Hook. `render()` deve restare puro in entrambi i modelli.

| Classe | Componente funzione | Distinzione nella migrazione |
| --- | --- | --- |
| `this.setState(nextState)` | Setter di `useState` o `dispatch` di `useReducer` | `setState` accoda un aggiornamento e **fonde superficialmente** l'oggetto nello stato di classe; il setter di `useState` sostituisce il valore. Per valori basati sul precedente usare l'updater funzionale. |
| `componentDidMount`, `componentDidUpdate`, `componentWillUnmount` | Di solito `useEffect` con dipendenze e cleanup | Non esiste una conversione uno a uno: un Effect sincronizza un sistema esterno rispetto ai valori reattivi; la logica di evento resta nell'handler. |
| `componentDidCatch` e `getDerivedStateFromError` | Nessun Hook equivalente diretto | Un [Error Boundary](04-context-state-architecture.md#error-boundary) resta un componente di classe, eventualmente incapsulato attorno a componenti funzione. |

Per esempio, `this.setState(s => ({ count: s.count + 1 }))` aggiorna solo `count` e conserva gli altri campi; `setState(s => ({ count: s.count + 1 }))` di `useState` sostituirebbe l'intero oggetto. Prima di riscrivere una classe, individuare ownership dello stato, side effect e contratti dei figli; per [Hook ed Effect](03-effects-refs-hooks.md#effect-e-sincronizzazione) vedi il capitolo dedicato.

### Root e rendering moderno

| Versione | Avvio client | HTML (HyperText Markup Language) già prodotto dal server |
| --- | --- | --- |
| React 17 | `ReactDOM.render` | `ReactDOM.hydrate` |
| React 18 | `createRoot` introdotta; `ReactDOM.render` deprecata | `hydrateRoot` introdotta; `ReactDOM.hydrate` deprecata |
| React 19 | `createRoot` corrente; `ReactDOM.render` rimossa | `hydrateRoot` corrente; `ReactDOM.hydrate` rimossa |

Le API moderne si importano da `react-dom/client`: `createRoot(container)` restituisce una root su cui chiamare `root.render(<App />)`; `hydrateRoot(container, <App />)` restituisce una root e riusa l'HTML server, che deve corrispondere al primo render client. `root.unmount()` sostituisce la vecchia `unmountComponentAtNode`, rimossa in React 19. Non usare `createRoot` su markup da idratare: per [rendering client, server e hydration](06-performance-server-rendering.md#csr-ssr-e-hydration) vedi il confronto dedicato.

## Evoluzione delle API

### Form, ref e rendering server

- **Form e Actions:** React 19 ha introdotto le Form Actions (`<form action={fn}>`), `useActionState` e `useFormStatus` per invio, risposta e stato pending. Non sostituiscono automaticamente ogni `onSubmit`: cambiano il flusso di invio e richiedono di verificare validazione, errori e integrazione server. Vedi [Form Actions e stato pending](02-state-events-forms.md#form-actions-e-stato-pending).
- **Ref:** da React 19 un componente funzione può ricevere `ref` come prop; in React 18 e precedenti si usa `forwardRef`. `forwardRef` è ancora supportato in React 19 e la sua deprecazione è prevista per una versione futura: non trattarlo come rimosso. Verificare anche il supporto del transform JSX (JavaScript XML) moderno e dei tipi del progetto; vedi [ref ai componenti](03-effects-refs-hooks.md#ref-ai-componenti-e-api-imperative).
- **Rendering server:** React Server Components e Server Functions sono stabili in React 19, ma richiedono un framework/bundler compatibile; le API usate per implementare quel supporto non garantiscono compatibilità tra versioni minori. `'use client'` marca il confine dei moduli client, mentre `'use server'` marca funzioni invocabili dal client, non Server Components. Vedi [confine server/client](06-performance-server-rendering.md#server-e-client-components).

## Strategia di migrazione

### Compatibility e aggiornamento graduale

**Compatibility** significa che React, `react-dom`, tipi, renderer, router, librerie di test e build tool possono funzionare insieme nella versione scelta. Una feature **introdotta** è disponibile da una versione; **stabile** ne descrive il contratto pubblico; **default** significa attiva senza configurazione: non sono sinonimi. Per esempio, React Compiler è stabile ma opzionale, mentre Server Components stabili richiedono l'integrazione del framework.

1. Inventariare versioni e peer dependency; per passare a React 19, React 18.3 può evidenziare avvisi di deprecazione prima dell'upgrade. Controllare anche il transform JSX moderno, tipi e librerie che dipendono da API rimosse.
2. Applicare i codemod ufficiali pertinenti su una modifica isolata, poi rivedere il diff: automatizzano sostituzioni meccaniche, non semantica di stato, Effect o errori.
3. Eseguire test pertinenti su root, hydration, form, routing ed Error Boundary, inclusi i controlli di sviluppo di StrictMode; distribuire gradualmente e osservare errori prima di completare il rollout.

Preparare un rollback della versione applicativa e verificare che cambiamenti di dati o contratto restino compatibili: il rollback del codice non annulla automaticamente dati già scritti. Per [rilascio e rollback](../../topics/engineering/15-architecture-devops.md#container-e-deployment) vedi Engineering.

### Casi legacy e interoperabilità

Una classe funzionante non richiede una riscrittura solo perché esistono gli Hook: si può mantenere e comporre con componenti funzione, migrando un confine alla volta. In librerie meno aggiornate controllare import da `react-dom`, uso di `ReactDOM.findDOMNode` (rimosso in React 19), ref e dipendenze di versione; isolare o aggiornare il componente interessato prima di cambiare tutta l'app. Confermare il comportamento con i [test dei componenti](07-testing-accessibility-security.md#testing-dei-componenti), senza assumere che un codemod risolva incompatibilità di runtime.
