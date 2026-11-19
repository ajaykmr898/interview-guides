# Context e architettura dello stato React

L'**architettura dello stato React** decide chi possiede un dato, chi lo legge e come riceve gli aggiornamenti. Props, Context e store esterni risolvono esigenze diverse: condividere un valore non equivale a gestirne persistenza, cache o sincronizzazione remota.

## Indice

- [Stato condiviso](#stato-condiviso)
- [Architettura dello stato](#architettura-dello-stato)
- [Confini dei componenti](#confini-dei-componenti)

## Stato condiviso

### Props, Context e Provider

Le **props** passano dati dal genitore ai figli; quando molti livelli devono solo inoltrarli si parla di *prop drilling*. `createContext(defaultValue)` crea un oggetto Context, mentre `useContext(Context)` legge il `value` del Provider corrispondente più vicino **sopra** il componente. Se non esiste, restituisce il fallback statico dichiarato con `createContext`; un Provider che passa `undefined` non attiva il fallback.

```jsx
import { createContext, useContext } from "react";

const Tema = createContext("chiaro");
function Etichetta() {
  const tema = useContext(Tema);
  return <span>{tema}</span>;
}
function App() { return <Tema value="scuro"><Etichetta /></Tema>; }
```

Da **React 19** il Context può essere usato direttamente come Provider (`<Tema value=...>`); nelle versioni precedenti si usa `<Tema.Provider>`, ancora supportato. Il Context non conserva dati autonomamente: il componente Provider può possedere stato locale e pubblicarne il valore. Quando `value` cambia secondo [Object.is](../../topics/javascript/03-javascript-basics.md#coercizione-truthy-e-uguaglianza), React aggiorna i consumer, anche se sono avvolti in [`memo`](06-performance-server-rendering.md#memo-usememo-e-usecallback); un oggetto ricreato a ogni render risulta sempre diverso. Preferire props o composizione per pochi passaggi, e Context per dati davvero trasversali come tema o sessione.

### useReducer e composizione

`useReducer(reducer, initialArg, init?)` restituisce `[state, dispatch]`. `dispatch(action)` accoda un aggiornamento e restituisce `undefined`; React chiama `reducer(state, action)` per calcolare il prossimo stato. Il reducer deve essere puro e non mutare lo stato: è utile quando più eventi modificano lo stesso dato secondo regole articolate. Per aggiornamenti semplici, [useState](02-state-events-forms.md#usestate-snapshot-e-batching) richiede meno struttura.

```jsx
import { useReducer } from "react";

function reducer(count, action) {
  return action.type === "increment" ? count + 1 : count;
}
function Contatore() {
  const [count, dispatch] = useReducer(reducer, 0);
  return <button onClick={() => dispatch({ type: "increment" })}>{count}</button>;
}
```

`dispatch` non modifica lo snapshot letto nello stesso handler; `init(initialArg)` serve solo a calcolare lo stato iniziale quando fornito. Combinare reducer e Context permette di distribuire stato e dispatch senza passarli a ogni livello, ma i consumer del Context si aggiornano quando cambia il valore pubblicato. Valutare Context separati per stato e dispatch se la misurazione mostra re-render superflui.

## Architettura dello stato

### Local state, Context e server state

La scelta dipende dalla **ownership** e dal ciclo di vita del dato. Il local state appartiene a un componente e si perde quando quel sottoalbero viene ricreato; il [lifting state up](02-state-events-forms.md#lifting-state-up-e-reset) lo condivide tra fratelli. Context distribuisce un valore lungo un sottoalbero, senza renderlo persistente o trasformarlo in una cache.

| Tipo di dato | Responsabilità | Limite da ricordare |
| --- | --- | --- |
| Local state | Interazioni di un componente o di un piccolo sottoalbero. | Va sollevato solo se altri componenti devono condividerlo. |
| Context | Distribuzione di un valore a consumer lontani. | Cambiamenti del Provider aggiornano i consumer; non gestisce cache o accesso ai dati remoti. |
| Server state | Dati remoti con caricamento, errori, freschezza e revalidazione. | Richiede una strategia di fetch/cache; non è sinonimo di [React Server Components](06-performance-server-rendering.md#react-server-components). |
| Store globale | Stato client condiviso fuori da un singolo sottoalbero. | Aggiunge contratto di sottoscrizione e manutenzione; non è necessario per ogni dato condiviso. |

Una libreria di data cache o un framework possono gestire il server state; React non impone una libreria specifica. Per [fetch, Suspense e dati asincroni](05-data-suspense-routing.md#dati-asincroni) vedi il capitolo dedicato; per i [confini architetturali generali](../../topics/engineering/15-architecture-devops.md#architettura-frontend) vedi Engineering.

### Store esterni e useSyncExternalStore

Uno **store esterno** conserva dati fuori dallo stato React e notifica i subscriber quando cambiano. `useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot?)` restituisce lo snapshot corrente: `subscribe(callback)` deve registrare la notifica e restituire una funzione di unsubscribe; `getSnapshot()` deve restituire lo **stesso valore** finché lo store non cambia. Se restituisce ogni volta un nuovo oggetto, può causare re-render continui; per dati mutabili produrre e riusare uno snapshot immutabile.

React confronta gli snapshot con `Object.is` e risottoscrive se cambia l'identità di `subscribe`. Nel rendering server `getServerSnapshot` è richiesto e deve produrre un valore coerente con quello usato nella hydration, altrimenti React segnala un errore o un mismatch. Usare questo Hook quando si integra uno store o un'interfaccia browser esterna; per cache, selettori e politiche di aggiornamento complesse è spesso più pratico adottare una libreria dedicata.

## Confini dei componenti

### Error Boundary

Un **Error Boundary** isola un sottoalbero che fallisce durante il render e mostra un fallback al posto della sua interfaccia. In React si implementa con un componente di classe: `static getDerivedStateFromError(error)` aggiorna lo stato per il fallback; `componentDidCatch(error, info)` può registrare l'errore. Non cattura errori nei propri metodi, negli event handler, nel rendering server o in callback asincrone esterne come `setTimeout`. Gli errori lanciati nella funzione `startTransition` restituita da [`useTransition`](05-data-suspense-routing.md#transition-e-deferred-value) possono invece raggiungere il boundary.

```jsx
import { Component } from "react";

class ErrorBoundary extends Component {
  state = { failed: false };
  static getDerivedStateFromError() { return { failed: true }; }
  render() {
    return this.state.failed ? <p>Contenuto non disponibile</p> : this.props.children;
  }
}
```

Collocare i boundary attorno a parti recuperabili dell'interfaccia; un fallback non corregge la causa né sostituisce la gestione di errori di rete e validazione. Per il [testing degli error boundary](07-testing-accessibility-security.md#routing-form-ed-error-boundary) vedi il capitolo dedicato.

### API dei componenti e provider

L'**API (Application Programming Interface)** di un componente definisce props, callback e slot `children` che i chiamanti possono usare. Preferire composizione e props esplicite per dipendenze locali; introdurre un Provider quando molti discendenti hanno bisogno dello stesso valore. Provider annidati delimitano lo scope e il più vicino prevale, ma troppi livelli o valori instabili rendono difficile seguire provenienza dei dati e re-render. Estrarre un'astrazione riusabile solo quando riduce una duplicazione reale senza nascondere l'ownership dello stato.
