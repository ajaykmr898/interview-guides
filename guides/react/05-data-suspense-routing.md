# Dati Suspense e routing React

In React, i dati asincroni alimentano la UI dopo un'operazione esterna; `Suspense` coordina l'attesa durante il render, mentre il routing associa URL e viste tramite un router o framework.

## Indice

- [Dati asincroni](#dati-asincroni)
- [Suspense e priorità](#suspense-e-priorità)
- [Routing](#routing)

## Dati asincroni

### Fetch nel client e suoi limiti

Un fetch avviato in `useEffect` parte dopo il commit e richiede stati espliciti di **loading**, **error** e **success**: mostrare un'attesa, gestire il fallimento e rendere i dati solo quando disponibili. Per la semantica di Promise, `fetch` e status HTTP vedi [JavaScript asincrono](../../topics/javascript/04-javascript-async-dom.md#web-api-e-fetch).

- Quando cambia una dipendenza o il componente si smonta, il cleanup deve impedire che una risposta precedente sovrascriva quella nuova: usare un flag che ignora il risultato obsoleto oppure annullare la richiesta con `AbortController` quando possibile. L'annullamento non sostituisce sempre il controllo della risposta già arrivata; vedi [Effect e cleanup](03-effects-refs-hooks.md#useeffect-dipendenze-e-cleanup).
- Il fetch nell'Effect non viene eseguito durante il rendering server e, da solo, non offre cache, deduplicazione o precaricamento: può produrre richieste in cascata tra componenti. Un framework con data fetching integrato o una cache client può gestire questi aspetti; la scelta dipende dall'applicazione e dal [server state](04-context-state-architecture.md#local-state-context-e-server-state).

### use e risorse asincrone

Da React 19, `use(resource)` legge durante il render il valore di una **Promise** risolta o di un **Context**. Una Promise ancora pending sospende il componente e attiva il `Suspense` più vicino; se viene rifiutata, l'errore passa all'[Error Boundary](04-context-state-architecture.md#error-boundary). Con un Context, `use` restituisce il valore del provider più vicino o il default, come `useContext`.

```jsx
import { Suspense, use } from 'react';

function Profilo({ profiloPromise }) {
  const profilo = use(profiloPromise);
  return <p>{profilo.nome}</p>;
}

function Vista({ profiloPromise }) {
  return (
    <Suspense fallback={<p>Caricamento…</p>}>
      <Profilo profiloPromise={profiloPromise} />
    </Suspense>
  );
}
```

Nell'esempio, `profiloPromise` arriva da un framework o da una cache. `use` non avvia il fetch: la Promise deve essere stabile tra i render; crearne una nuova nel render client può causare sospensioni ripetute. A differenza degli Hook, `use` può essere chiamata in condizioni e cicli, ma solo dentro un componente o custom Hook e non in un `try/catch`. La lettura di Context con `use` non è supportata nelle Server Components.

### Aggiornamenti ottimistici

`useOptimistic(valore, reducer?)` (React 19) restituisce lo stato visualizzato e una funzione per aggiornarlo durante un'**Action** o Transition, prima della conferma remota. L'aggiornamento temporaneo termina con l'Action: in caso di errore la UI torna al valore base, ma occorre mostrare il fallimento; in caso di successo il valore base va allineato alla risposta del server o alla cache aggiornata. Non sostituisce la persistenza né la gestione dei conflitti.

## Suspense e priorità

### Boundary Suspense e fallback

`<Suspense fallback={...}>` delimita un sottoalbero: se un figlio **sospende durante il render**, React mostra il fallback finché quel contenuto non è pronto. Boundary annidati permettono di mostrare parti della pagina progressivamente. La sospensione non equivale a un errore: un errore di render o una Promise rifiutata richiede un Error Boundary.

Tra le sorgenti supportate ci sono componenti caricati con `lazy`, Promise lette con `use` e dati integrati da framework compatibili con Suspense. Un `fetch` avviato in `useEffect` o in un event handler **non** attiva da solo il boundary. Per il codice caricato a richiesta vedi [React.lazy e code splitting](06-performance-server-rendering.md#reactlazy-e-code-splitting).

### Transition e deferred value

Gli aggiornamenti **urgenti** (per esempio il testo di un input) devono riflettersi subito; quelli **non urgenti** possono essere renderizzati in background e interrotti da nuove interazioni. Questo migliora la reattività del render, non accelera la richiesta di rete.

| API | Uso e risultato | Limite principale |
| --- | --- | --- |
| `useTransition()` | Restituisce `[isPending, startTransition]`; gli aggiornamenti di stato eseguiti in `startTransition(action)` sono non urgenti. | La callback parte subito; gli aggiornamenti dopo un `await` richiedono un altro `startTransition`. Non usare una Transition per lo stato che controlla un input testuale. |
| `useDeferredValue(value)` | Restituisce una versione differita del valore per renderizzare in background una parte costosa della UI, mantenendo visibile quella precedente mentre sospende. | Non applica un ritardo fisso, non evita da sola richieste di rete e non deve ritardare il valore dell'input controllato. |

Per una ricerca, aggiornare subito il valore dell'input e passare la sua versione differita alla lista dei risultati; se si controlla il setter della lista, una Transition è un'alternativa. Entrambe possono evitare che una UI già visibile venga sostituita subito dal fallback durante un aggiornamento che sospende.

## Routing

### Router come dipendenza esterna

React non include un router. Una libreria di routing o un framework associa l'URL alla vista, gestisce navigazione e parametri, compone route annidate e può caricare dati prima o durante la navigazione. Link, loader e convenzioni delle route sono **API del router scelto**, non di React: verificarne comportamento e versione nella sua documentazione.

Una route può possedere i dati necessari alla vista, evitando fetch dipendenti a cascata negli Effect dei componenti. Il router può coordinare stato pending, cache e errori, ma queste garanzie dipendono dall'integrazione concreta.

### Confini delle route

Una route può caricare il codice della propria vista a richiesta con `lazy` e mostrare un fallback tramite `Suspense`; per il meccanismo vedi [Caricamento](06-performance-server-rendering.md#caricamento). Gli errori di render richiedono un [Error Boundary](04-context-state-architecture.md#error-boundary); eventuali boundary, pagine di errore e retry **per route** appartengono al router/framework adottato.
