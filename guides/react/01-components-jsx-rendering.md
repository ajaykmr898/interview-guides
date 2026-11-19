# Componenti JSX e rendering React

React è una libreria dichiarativa per costruire interfacce come albero di componenti: descrive la UI desiderata a partire da props e stato, poi aggiorna il Document Object Model (DOM) dove necessario. JSX (JavaScript XML) è la sintassi usata comunemente per esprimere quell'albero.

## Indice

- [Modello React e ciclo di rendering](#modello-react-e-ciclo-di-rendering)
- [JSX e composizione](#jsx-e-composizione)
- [Liste e identità](#liste-e-identità)

## Modello React e ciclo di rendering

### Componenti funzione e props

Un **componente funzione** è una funzione JavaScript che riceve un oggetto di **props** e restituisce nodi React, spesso espressi in JSX. Un **elemento React** è la descrizione del risultato da renderizzare, non un nodo DOM. Le props sono input in sola lettura; `children` contiene i nodi annidati tra i tag del componente.

```jsx
function Card({ titolo, children }) {
  return <section><h2>{titolo}</h2>{children}</section>;
}

function Profilo({ nome }) {
  return <Card titolo={nome}><p>Profilo pubblico</p></Card>;
}
```

I dati scorrono dal genitore ai figli tramite props; per comunicare un'interazione verso l'alto, il genitore passa una callback. Durante il render, il componente deve calcolare il risultato dagli input senza mutare props, stato o valori esterni né produrre side effect: React può richiamarlo più volte o scartare un render. Per lo [stato e gli eventi](02-state-events-forms.md) vedi il capitolo dedicato.

### Root e mount dell'applicazione

`react-dom` collega l'albero React al DOM del browser. `createRoot(container)` crea una root su un elemento DOM esistente e restituisce un oggetto con `render(reactNode)` e `unmount()`; `root.render(<App />)` avvia il rendering client e restituisce `undefined`, senza garantire che il commit sia già terminato alla riga successiva.

```jsx
import { createRoot } from "react-dom/client";
import App from "./App.js";

const root = createRoot(document.getElementById("root"));
root.render(<App />);
```

L'esempio richiede un elemento HTML (HyperText Markup Language) con `id="root"`. In un'app con rendering server, `hydrateRoot` riusa l'HTML già prodotto invece di inizializzare con `createRoot`; il confronto tra [client-side rendering (CSR), server-side rendering (SSR) e hydration](06-performance-server-rendering.md#csr-ssr-e-hydration) è nel capitolo dedicato. Un framework può gestire la root automaticamente.

### Render, commit e riconciliazione

Un aggiornamento può essere richiesto da nuove props o da un cambio di stato. **Render**, **commit** e **paint** sono fasi distinte; la **riconciliazione** confronta il nuovo albero React con il precedente.

| Fase | Che cosa accade |
| --- | --- |
| Render e riconciliazione | React calcola il prossimo albero; un render può essere ripetuto o abbandonato senza modificare il DOM. |
| Commit | React applica le modifiche al DOM; un re-render può non richiedere modifiche. |
| Paint | Il browser disegna i pixel secondo il proprio ciclo di rendering. |

React conserva l'identità e lo stato di un componente quando tipo e posizione nell'albero restano compatibili; le `key` possono precisare l'identità fra fratelli. Un cambio di tipo o `key` può ricreare il sottoalbero e azzerarne lo stato. Per il [ciclo di rendering del browser](../../topics/javascript/04-javascript-async-dom.md#dom-e-rendering) vedi il capitolo JavaScript.

### StrictMode nel render

`<StrictMode>` abilita controlli di sviluppo: React richiama una volta in più le funzioni di render per individuare calcoli impuri. Il risultato di una chiamata viene scartato; questi controlli non aggiungono un secondo render in produzione. Non usare il numero di chiamate del componente come misura dei commit. La riesecuzione degli [Effect](03-effects-refs-hooks.md#strictmode-e-riesecuzione) è trattata separatamente.

## JSX e composizione

### Sintassi JSX e attributi

**JSX** è un'estensione sintattica di JavaScript che descrive elementi React con markup simile a HTML; non è una stringa HTML. Le espressioni JavaScript si inseriscono con `{}`. I tag vanno chiusi, più elementi adiacenti richiedono un unico contenitore o un Fragment e le props DOM seguono in genere nomi JavaScript: `className` per la classe CSS (Cascading Style Sheets), `htmlFor` per l'associazione di una label; `aria-*` e `data-*` mantengono i trattini.

```jsx
function Campo({ id, etichetta }) {
  return <label htmlFor={id} className="campo">
    {etichetta}<input id={id} />
  </label>;
}
```

React tratta il testo inserito con `{}` come testo, senza interpretarlo come markup HTML: una stringa contenente `<script>` non diventa un tag. Questo escaping non sostituisce la verifica dei dati o la gestione dell'[HTML non attendibile](07-testing-accessibility-security.md#escaping-e-html-non-attendibile).

### Children, Fragment e composizione

La **composizione** combina componenti passando nodi tramite `children` o altre props: il contenitore decide dove mostrarli, il chiamante ne decide il contenuto. È utile per layout e componenti riusabili senza imporre una gerarchia rigida. Un `Fragment` (`<>...</>`) raggruppa nodi senza aggiungere un elemento DOM; quando serve una `key`, usare la forma esplicita `<Fragment key={id}>`.

### Portali e albero React

`createPortal(children, domNode)` di `react-dom` restituisce un nodo React renderizzato in un contenitore DOM già esistente fuori dalla posizione DOM del genitore, utile per dialog e tooltip. Il portale resta però nel medesimo **albero React**: riceve il Context del genitore e gli eventi React propagano lungo quell'albero, anche se il DOM è altrove. La gestione di focus e accessibilità del dialog rimane responsabilità del componente; vedi [accessibilità e portali](07-testing-accessibility-security.md#annunci-e-portali).

## Liste e identità

### Rendering condizionale e liste

Un componente sceglie cosa restituire con `if`, espressioni ternarie o `&&`; `null` e `false` non producono nodi visibili. Con `&&`, un numero `0` può invece comparire nell'output: per una condizione numerica usare un confronto esplicito. Gli array si trasformano tipicamente con `map` per produrre una lista di elementi.

```jsx
function Elenco({ persone }) {
  if (persone.length === 0) return <p>Nessun risultato</p>;
  return <ul>{persone.map(p => <li key={p.id}>{p.nome}</li>)}</ul>;
}
```

### Key e stato

La **`key`** identifica un elemento tra i suoi fratelli durante la riconciliazione: permette a React di seguire gli elementi quando una lista viene inserita, riordinata o ridotta e di conservare lo stato del componente corretto. Deve essere stabile e unica fra fratelli; non viene passata al componente come normale prop. Preferire un ID dei dati: l'indice è sicuro solo se ordine e composizione della lista restano fissi, mentre una key casuale a ogni render forza la ricreazione.

Fuori dalle liste, cambiare intenzionalmente `key` su un componente nella stessa posizione ne resetta lo stato insieme al sottoalbero. Per ownership e reset dello [stato React](02-state-events-forms.md#lifting-state-up-e-reset) vedi il capitolo successivo.
