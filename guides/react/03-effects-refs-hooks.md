# Effect ref e Hook React

Gli **Hook** aggiungono stato e altre capacità React ai componenti funzione. Un **Effect** sincronizza il componente con un sistema esterno dopo il commit; una **ref** conserva un valore mutabile tra render senza richiedere un aggiornamento visivo.

## Indice

- [Regole degli Hook](#regole-degli-hook)
- [Effect e sincronizzazione](#effect-e-sincronizzazione)
- [Ref e escape hatch](#ref-e-escape-hatch)

## Regole degli Hook

### Ordine di chiamata e custom Hook

React associa lo stato degli Hook alla posizione delle chiamate nel componente. Chiamarli al **top level**, prima di eventuali `return` anticipati: non in condizioni, cicli o funzioni annidate. Gli Hook possono essere chiamati solo da componenti React o da altri Hook; cambiare l'ordine tra render assocerebbe lo stato alla chiamata sbagliata.

Un **custom Hook** è una funzione con nome `use...` che combina altri Hook per riusare logica stateful. Ogni chiamata ha stato ed Effect propri: non condivide automaticamente i dati tra componenti. Se la funzione non usa Hook, basta una normale funzione JavaScript.

```jsx
import { useState } from "react";

function useToggle(initial = false) {
  const [on, setOn] = useState(initial);
  return [on, () => setOn(value => !value)];
}
```

Per condividere **lo stesso stato** tra componenti, usare un antenato comune o il [Context](04-context-state-architecture.md#props-context-e-provider), non due chiamate allo stesso custom Hook.

## Effect e sincronizzazione

### useEffect, dipendenze e cleanup

`useEffect(setup, dependencies?)` dichiara una sincronizzazione con timer, listener, connessioni o altri sistemi esterni e restituisce `undefined`. Dopo il commit React esegue `setup`; se questo restituisce una funzione, React la usa come **cleanup** prima del successivo setup con dipendenze cambiate e quando il componente viene rimosso. Il cleanup deve annullare ciò che il setup ha avviato; gli Effect non vengono eseguiti durante il rendering server.

```jsx
import { useEffect } from "react";

function Timer({ onTick }) {
  useEffect(() => {
    const id = setInterval(onTick, 1000);
    return () => clearInterval(id);
  }, [onTick]);
  return null;
}
```

Le **dipendenze** includono props, stato e valori o funzioni dichiarati nel componente usati dall'Effect; React le confronta con [Object.is](../../topics/javascript/03-javascript-basics.md#coercizione-truthy-e-uguaglianza). `[]` esegue normalmente il setup al mount e il cleanup all'unmount (salvo controlli di sviluppo); senza array l'Effect viene rieseguito dopo ogni commit. Una funzione o un oggetto ricreati a ogni render possono causare risincronizzazioni inutili: semplificare la logica prima di omettere una dipendenza richiesta.

Una **stale closure** legge i valori catturati da un render precedente, spesso perché manca una dipendenza. Per operazioni asincrone, il cleanup deve anche impedire che un risultato ormai obsoleto aggiorni lo stato: ignorare il risultato o annullare la richiesta quando possibile. L'annullamento non garantisce il rollback di un'azione già eseguita dal server; per [Fetch e race nel client](05-data-suspense-routing.md#fetch-nel-client-e-suoi-limiti) vedi il capitolo dati. Da React 19.2, `useEffectEvent` separa la logica non reattiva chiamata **dentro** un Effect che deve leggere i valori più recenti; non è un modo per nascondere dipendenze dell'Effect.

### Quando evitare un Effect

Se un valore deriva da props o stato, calcolarlo durante il render: copiarlo in uno stato aggiornato da un Effect crea un passaggio e un render aggiuntivi. Un'azione causata da un click o da un submit appartiene all'**event handler**, dove è nota l'interazione che l'ha provocata. Usare un Effect quando l'esistenza o i valori del componente devono mantenere sincronizzato un sistema esterno; non per orchestrare il normale flusso dei dati React. Vedi [stato derivato ed eventi](02-state-events-forms.md#oggetti-array-e-stato-derivato).

### StrictMode e riesecuzione

Con `<StrictMode>` alla root, in sviluppo React esegue un ciclo supplementare **setup → cleanup → setup** al mount per individuare cleanup mancanti. In produzione questo controllo non avviene; non va aggirato con una ref che blocca il secondo setup. Se StrictMode copre solo parte dell'albero, React non ripete gli Effect al mount iniziale di quella parte. I controlli del [render puro](01-components-jsx-rendering.md#strictmode-nel-render) sono distinti.

### useLayoutEffect

`useLayoutEffect(setup, dependencies?)` ha dipendenze e cleanup analoghi a `useEffect` e restituisce `undefined`.

| Hook | Momento e uso | Limite |
| --- | --- | --- |
| `useEffect` | Dopo il commit; adatto alla maggior parte delle sincronizzazioni esterne. | Il momento rispetto al paint dipende anche dal tipo di aggiornamento. |
| `useLayoutEffect` | Dopo il commit, prima del repaint; utile per misurare e correggere il layout senza un frame visibile errato. | Blocca il paint e può peggiorare le prestazioni. |

Preferire `useEffect` quando non serve il vincolo prima del paint. Sul server non esiste layout da misurare e `useLayoutEffect` non vi esegue il setup.

## Ref e escape hatch

### useRef e DOM

`useRef(initialValue)` restituisce lo stesso oggetto `{ current }` nei render successivi; assegnare `current` è immediato ma **non** avvia un re-render. È adatto a identificatori di timer o nodi del Document Object Model (DOM) usati in handler ed Effect, non a valori che devono apparire nella UI (User Interface). Evitare letture e scritture di `current` durante il render, salvo inizializzazioni prevedibili.

```jsx
import { useRef } from "react";

function Cerca() {
  const inputRef = useRef(null);
  return <>
    <input ref={inputRef} />
    <button type="button" onClick={() => inputRef.current?.focus()}>Focus</button>
  </>;
}
```

React assegna il nodo a `current` durante il commit e lo riporta a `null` quando lo rimuove; al primo render il nodo non è ancora disponibile. Preferire props e stato per comportamento dichiarativo, usando la ref per operazioni imperative come focus. Per i [nodi DOM e la loro gestione](../../topics/javascript/04-javascript-async-dom.md#dom-e-rendering) vedi JavaScript.

### Ref ai componenti e API imperative

Da **React 19**, un componente funzione può ricevere `ref` come prop e inoltrarla a un elemento DOM. In React 18 e precedenti serviva `forwardRef`, ancora disponibile ma destinato a una futura deprecazione. Un'interfaccia di programmazione (API, Application Programming Interface) imperativa può esporre meno del nodo: `useImperativeHandle(ref, createHandle, dependencies?)` restituisce `undefined` e assegna alla ref del genitore l'handle creato, per esempio un metodo `focus`.

```jsx
import { useImperativeHandle, useRef } from "react";

function Campo({ ref }) {
  const inputRef = useRef(null);
  useImperativeHandle(ref, () => ({
    focus: () => inputRef.current?.focus()
  }), []);
  return <input ref={inputRef} />;
}
```

L'handle viene ricreato se cambiano le dipendenze reattive di `createHandle`; mantenerlo minimo evita di vincolare il genitore alla struttura DOM interna. Per la [migrazione delle ref](08-evolution-migrations.md#form-ref-e-rendering-server) vedi il capitolo dedicato.

### Callback ref

Una **callback ref** è una funzione passata a `ref` che riceve il nodo quando React lo collega; è utile per setup e cleanup mirati, per esempio registrare nodi dinamici in una `Map`. Da React 19 può restituire una funzione di cleanup chiamata alla rimozione o al cambio della ref. Senza cleanup restituito React richiama ancora la callback con `null` per compatibilità, comportamento destinato a sparire: non restituire accidentalmente il nodo. Come per [useRef](#useref-e-dom), l'assegnazione avviene nel commit; StrictMode può ripetere setup e cleanup in sviluppo.
