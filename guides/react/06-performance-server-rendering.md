# Performance e rendering server React

La performance React dipende dal lavoro di render, dal commit e dal browser; il rendering server produce contenuto prima dell'esecuzione nel browser. Misurazione, caricamento del codice e confine server/client risolvono costi diversi.

## Indice

- [Misurazione e ottimizzazione](#misurazione-e-ottimizzazione)
- [Caricamento](#caricamento)
- [Rendering server](#rendering-server)
- [React Server Components](#react-server-components)

## Misurazione e ottimizzazione

### Profiler e cause dei re-render

Un **re-render** ricalcola un sottoalbero React quando cambiano stato, props o Context; può propagarsi dal render del genitore, salvo ottimizzazioni. Non implica necessariamente una modifica del Document Object Model (DOM). Per la distinzione tra [render, commit e paint](01-components-jsx-rendering.md#render-commit-e-riconciliazione) vedi il capitolo introduttivo.

- Prima di ottimizzare, usare React DevTools Profiler per individuare componenti e commit costosi. Il componente `<Profiler>` misura programmaticamente il tempo di render del sottoalbero tramite le props `id` e `onRender`: la callback riceve fase, `actualDuration` e tempi del commit; in produzione richiede una build speciale con profiling.
- Un render React economico può comunque precedere lavoro costoso del browser (layout, paint) o di rete: misurare questi costi con gli strumenti del browser. Controllare anche props instabili, Context troppo ampio e `key` che ricreano componenti, prima di aggiungere memoizzazione.

### memo, useMemo e useCallback

| API | Cosa conserva | Quando aiuta e limite |
| --- | --- | --- |
| `memo(Componente)` | Restituisce un componente memoizzato; confronta le props con `Object.is` per prop, salvo comparatore personalizzato. | Può evitare un render dovuto al genitore se le props restano uguali; stato o Context del componente possono comunque causarlo. Il confronto ha un costo e non garantisce che React salti sempre il render. |
| `useMemo(calcolo, dipendenze)` | Restituisce il risultato del calcolo puro, riusato finché le dipendenze non cambiano. | Utile per un calcolo misurato come costoso o per stabilizzare un valore passato a un figlio memoizzato; non usarlo come deposito di stato. |
| `useCallback(fn, dipendenze)` | Restituisce la funzione `fn` con identità stabile finché le dipendenze non cambiano; non la esegue. | Utile se l'identità della callback conta per un figlio memoizzato o per un Effect; non rende più veloce la funzione stessa. |

Le dipendenze reattive vanno dichiarate correttamente e sono confrontate con `Object.is`. Oggetti o callback ricreati a ogni render possono annullare il beneficio di `memo`; misurare il guadagno e considerare il costo di lettura e manutenzione della memoizzazione.

### React Compiler

**React Compiler** è un'ottimizzazione automatica a build time: memoizza componenti e valori, riducendo la necessità di `memo`, `useMemo` e `useCallback` manuali. La versione 1.0 è stabile, ma il compiler resta **opzionale** e richiede integrazione nella build; non è attivato semplicemente installando React.

L'adozione può essere graduale: verificare le Rules of React, usare lint e test e misurare l'effetto sulle schermate rilevanti. Il compiler non corregge logica impura né elimina costi di rete, DOM o browser; la memoizzazione manuale resta possibile nei casi misurati in cui serve.

## Caricamento

### React.lazy e code splitting

`lazy(load)` restituisce un componente React il cui modulo è caricato quando viene renderizzato per la prima volta. Di norma `load` usa `import()` e restituisce una Promise che risolve a un modulo con `default` export del componente; React memorizza Promise e risultato. Finché il codice manca, il componente sospende e un [`Suspense` boundary](05-data-suspense-routing.md#boundary-suspense-e-fallback) mostra il fallback; un errore di caricamento raggiunge un [Error Boundary](04-context-state-architecture.md#error-boundary).

```jsx
import { lazy, Suspense } from 'react';

const Dettagli = lazy(() => import('./Dettagli.jsx'));

export function Pagina() {
  return <Suspense fallback={<p>Caricamento…</p>}><Dettagli /></Suspense>;
}
```

L'esempio presuppone che `Dettagli.jsx` esporti il componente come `default`. Il bundler/framework deve produrre i chunk: dividere il codice riduce il JavaScript iniziale, ma aggiunge richieste e fallback. Scegliere confini utili, per esempio una route o una vista poco frequente; per il meccanismo generale vedi [Bundling e code splitting](../../topics/node-tooling/09-build-tools.md#bundling-e-code-splitting).

## Rendering server

### CSR, SSR e hydration

| Strategia | HTML iniziale e interattività | Costo o limite |
| --- | --- | --- |
| **CSR (Client-Side Rendering)** | `createRoot` costruisce la UI nel browser. | Il contenuto dipende dal caricamento ed esecuzione del JavaScript client; un fetch in Effect arriva dopo il primo render. |
| **SSR (Server-Side Rendering)** | Il server produce HTML iniziale; il browser può mostrarlo prima che sia interattivo. | Richiede lavoro server e JavaScript client per idratare le parti interattive. |

L'**hydration** collega la logica React all'HTML prodotto dal server: `hydrateRoot(container, <App />)` restituisce una root con `render` e `unmount`, mentre per un contenitore vuoto si usa `createRoot`. Il primo render client deve corrispondere all'HTML server; dati divergenti, valori casuali o markup non valido possono causare un *hydration mismatch*, con warning e possibile rigenerazione client. Un framework normalmente coordina output, dati iniziali e hydration.

SSR descrive **come si produce l'HTML iniziale**; React Server Components (RSC) descrive **dove vengono eseguiti alcuni componenti e quali moduli arrivano al client**. Si può usare SSR senza RSC; una soluzione RSC può anche produrre HTML con SSR. Per il ciclo del browser vedi [Parsing e rendering](../../topics/web/01-html-web.md#parsing-e-rendering).

### Streaming e prerendering

Lo **streaming SSR** invia l'HTML a pezzi: con `Suspense` il server può inviare la shell e i fallback, poi il contenuto quando è pronto. `react-dom/server` espone `renderToPipeableStream` (stream Node.js, restituisce `pipe`/`abort`) e `renderToReadableStream` (Promise di Web Stream). Il vantaggio dipende da server, rete, boundary e capacità del framework di orchestrare errori e hydration.

Il **prerendering** produce HTML statico prima della richiesta: `prerender` di `react-dom/static` (React 19, Web Streams) attende che il contenuto sospeso sia pronto e restituisce una Promise con lo stream `prelude`; non invia progressivamente il contenuto mentre arriva. Per l'ambiente Node.js esiste `prerenderToNodeStream`. La scelta tra statico, streaming e rendering per richiesta dipende dalla freschezza dei dati.

## React Server Components

### Server e Client Components

Le **React Server Components (RSC)** sono componenti eseguiti in un ambiente server o durante la build: possono leggere dati con `await` nel render e il loro codice non entra nel bundle client. Non gestiscono interazioni browser con `useState`, Effect o event handler; compongono Client Components quando serve interattività. RSC sono stabili in React 19, ma l'integrazione di bundler/framework usa API ancora sensibili alle versioni: serve un framework compatibile.

La direttiva `'use client'` all'inizio di un modulo marca il confine client e include le sue dipendenze nel relativo bundle; le props che attraversano il confine devono usare i tipi serializzabili supportati. Non significa necessariamente «solo render nel browser»: un framework può includere Client Components nell'HTML iniziale SSR e poi idratarli. Non esiste una direttiva `'use server'` per definire Server Components: quella riguarda le Server Functions.

### Server Functions

Una **Server Function** è una funzione asincrona eseguita sul server ma invocabile dal client tramite una richiesta gestita dal framework. `'use server'` si mette all'inizio della funzione asincrona in un file server, oppure all'inizio di un modulo server per marcarne gli export; il client riceve un riferimento, non il codice della funzione. Argomenti e risultato devono essere serializzabili secondo i tipi supportati.

Le Server Functions sono stabili in React 19 ma richiedono un framework che implementi il protocollo. Sono pensate soprattutto per mutazioni, non come sostituto generico del [data fetching](05-data-suspense-routing.md#dati-asincroni). Ogni chiamata è un confine di rete: validare gli input e verificare l'autorizzazione sul server, anche se il bottone client è nascosto o disabilitato.
