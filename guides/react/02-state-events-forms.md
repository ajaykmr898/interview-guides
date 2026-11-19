# Stato eventi e form React

Lo **stato React** conserva dati tra i render dei componenti dell'interfaccia utente (UI); gli **event handler** reagiscono alle interazioni e i **form** collegano input del browser e stato applicativo. Scegliere dove conservare i dati evita copie incoerenti e aggiornamenti inattesi.

## Indice

- [Stato e aggiornamenti](#stato-e-aggiornamenti)
- [Eventi e interazioni](#eventi-e-interazioni)
- [Form e input](#form-e-input)

## Stato e aggiornamenti

### useState, snapshot e batching

`useState(initialState)` restituisce `[state, setState]`: il valore appartiene al componente nella sua posizione nell'albero React. Il setter accetta un nuovo valore o una funzione pura che riceve lo stato in coda e restituisce il successivo; restituisce `undefined` e richiede un aggiornamento. Leggere `state` subito dopo il setter restituisce ancora lo **snapshot** del render corrente, non il valore futuro.

```jsx
import { useState } from "react";

function Contatore() {
  const [count, setCount] = useState(0);
  return <button onClick={() => {
    setCount(n => n + 1);
    setCount(n => n + 1);
  }}>{count}</button>;
}
```

Ogni click aggiunge due incrementi in coda: il prossimo valore è `count + 2`. React raggruppa gli aggiornamenti (**batching**) prima di aggiornare lo schermo; due `setCount(count + 1)` nello stesso handler userebbero invece lo stesso snapshot. Usare il setter funzionale quando il nuovo valore dipende dal precedente. React può evitare il re-render se il valore finale è uguale al precedente secondo [Object.is](../../topics/javascript/03-javascript-basics.md#coercizione-truthy-e-uguaglianza).

### Oggetti, array e stato derivato

Oggetti e array nello stato sono snapshot: mutarli in-place non comunica a React un nuovo valore e può alterare render precedenti. Creare una nuova struttura per le parti modificate, per esempio `setProfilo(p => ({ ...p, nome }))` o `setElementi(xs => [...xs, nuovo])`; lo spread è superficiale, quindi anche i livelli annidati modificati richiedono copie.

Lo **stato derivato** è calcolabile da props o altro stato durante il render: non conservarne una copia separata se può divergere. Per esempio, da `prezzi` si può calcolare `const totale = prezzi.reduce((sum, n) => sum + n, 0)`; memorizzare anche `totale` richiederebbe sincronizzarlo a ogni modifica. Conservare invece nello stato i dati che l'utente può cambiare indipendentemente.

### Lifting state up e reset

Il **lifting state up** sposta lo stato al più vicino antenato comune dei componenti che devono leggerlo o modificarlo; il proprietario passa valori e callback ai figli. Serve a mantenere una sola fonte di verità, ma spostare tutto alla root amplia inutilmente i re-render e le responsabilità: tenere locale ciò che non è condiviso.

React conserva lo stato finché tipo, posizione e `key` del componente identificano la stessa istanza. Se un editor deve ripartire da zero quando cambia documento, usare `<Editor key={documento.id} documento={documento} />`; cambiare `key` ricrea anche il sottoalbero. Vedi [key e identità](01-components-jsx-rendering.md#key-e-stato).

## Eventi e interazioni

### Event handler e propagazione

Un **event handler** è una callback passata a una prop come `onClick`; React la richiama durante l'interazione. Passare `onClick={handleClick}`, non `onClick={handleClick()}`: la seconda forma esegue la funzione durante il render. Un componente figlio può ricevere dal genitore una callback per comunicare un'intenzione senza possedere il suo stato.

```jsx
function Pulsante({ onSave }) {
  return <button type="button" onClick={onSave}>Salva</button>;
}

function Pannello() {
  return <Pulsante onSave={() => console.log("Salvato")} />;
}
```

Gli eventi React espongono `target` (origine) e `currentTarget` (elemento che gestisce l'evento). Nel bubbling gli handler degli antenati nell'albero React possono essere chiamati dopo quello del figlio; `stopPropagation()` lo interrompe. Un portale mantiene questa propagazione React anche se il suo Document Object Model (DOM) è altrove. Per le [fasi degli eventi DOM](../../topics/javascript/04-javascript-async-dom.md#eventi-e-delega) vedi il capitolo JavaScript.

### Comportamento predefinito e stato UI

`preventDefault()` blocca l'azione predefinita cancellabile, per esempio l'invio HTML (HyperText Markup Language) del form; non ferma il bubbling. `stopPropagation()` blocca la propagazione, non l'azione predefinita. Negli handler si possono aggiornare stato e servizi esterni; dopo `setState`, lo snapshot letto nello stesso handler resta quello corrente. Derivare dal valore dell'evento ciò che serve subito, e usare lo stato per ciò che deve apparire nel render successivo.

## Form e input

### Input controlled e uncontrolled

Un input **controlled** riceve `value` (o `checked` per checkbox e radio) dallo stato React e un `onChange` che lo aggiorna sincronicamente. È utile per validazione immediata, campi dipendenti e interfacce che riflettono ogni modifica; senza aggiornare il valore nell'handler, l'input torna al valore passato. Inizializzare un campo testuale controlled con una stringa, non con `null` o `undefined`.

```jsx
import { useState } from "react";

function Nome() {
  const [nome, setNome] = useState("");
  return <input value={nome} onChange={e => setNome(e.target.value)} />;
}
```

Un input **uncontrolled** lascia il valore corrente al DOM: `defaultValue` o `defaultChecked` imposta solo quello iniziale; una ref o `FormData` legge il valore quando serve. È spesso sufficiente per un form letto all'invio e non richiede un render a ogni battuta. Evitare di passare lo stesso input da uncontrolled a controlled durante la sua vita.

| Scelta | Fonte del valore corrente | Limite principale |
| --- | --- | --- |
| Controlled | Stato React tramite `value`/`checked` e `onChange`. | Aggiorna lo stato a ogni modifica; richiede coerenza tra prop e handler. |
| Uncontrolled | DOM; `defaultValue`/`defaultChecked` sono iniziali. | Per reagire a ogni modifica occorre leggere il DOM o aggiungere stato. |

Per [ref e accesso al DOM](03-effects-refs-hooks.md#useref-e-dom) vedi il capitolo successivo.

### Validazione e invio

`onSubmit` riceve l'evento di invio: chiamare `event.preventDefault()` se il codice React gestisce l'invio senza la navigazione HTML. `new FormData(event.currentTarget)` legge i controlli con `name`; per i campi controlled si può invece usare lo stato corrente. Attributi nativi come `required` e `type` forniscono una prima validazione, ma gli input e i permessi vanno verificati anche sul server. La [semantica dei form HTML e della validazione](../../topics/web/01-html-web.md#form-e-validazione) è nel capitolo Web.

### Form Actions e stato pending

Da **React 19**, `<form action={fn}>` accetta una funzione client o una Server Function: React le passa un `FormData` con i controlli dotati di `name` e gestisce l'invio come Action in una Transition. A differenza di `onSubmit`, non serve chiamare `preventDefault()` per affidare l'invio alla funzione. Una funzione in `action` può essere asincrona; il form usa il metodo `POST` indipendentemente da `method`, ma una funzione client non invia da sola una richiesta HTTP (Hypertext Transfer Protocol). L'uso di [Server Functions](06-performance-server-rendering.md#server-functions) dipende dal framework.

```jsx
function FormDati() {
  function invia(formData) {
    console.log(formData.get("nome"));
  }
  return <form action={invia}>
    <input name="nome" /><button type="submit">Invia</button>
  </form>;
}
```

`useActionState(action, initialState)` restituisce `[state, dispatchAction, isPending]`. Usando `dispatchAction` come `action` del form, React chiama `action(previousState, formData)` e mette nello stato il valore restituito: utile per risposta o errori di validazione. `isPending` indica l'Action in corso; un errore lanciato raggiunge l'[Error Boundary](04-context-state-architecture.md#error-boundary), mentre un errore atteso può essere restituito nello stato.

`useFormStatus()` di `react-dom` restituisce fra l'altro `pending` e i dati dell'invio in corso (o `null`) del form **antenato**. Va chiamato in un componente figlio del form, per esempio nel pulsante di invio: chiamarlo nel componente che crea il form non ne legge lo stato. `onSubmit` resta disponibile quando serve gestire direttamente l'evento; per ottenere pending con questo approccio occorre avviare una Transition.

### Casi avanzati dei form

Quando una funzione `action` termina senza lanciare errori, React resetta i campi uncontrolled ai valori di default; anche restituire un errore di validazione conta come completamento, quindi conservare esplicitamente i valori da mostrare ancora. I campi controlled seguono invece il loro stato React. Con `useActionState`, invii ripetuti vengono messi in coda e ricevono lo stato restituito dall'Action precedente; per altri invii asincroni considerare duplicati, ordine delle risposte e stato pending prima di consentire nuove operazioni. Annullare una richiesta non annulla necessariamente una modifica già eseguita dal server.

La validazione client migliora il feedback, ma non garantisce integrità o autorizzazione: il server deve verificare dati e permessi. Per la [sicurezza ai confini server](07-testing-accessibility-security.md#confini-server-e-autorizzazione) vedi il capitolo dedicato.
