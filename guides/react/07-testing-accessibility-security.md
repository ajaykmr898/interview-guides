# Testing accessibilità e sicurezza React

I test React verificano l'interfaccia utente (UI) osservabile; l'accessibilità rende le interazioni utilizzabili anche con tastiera e tecnologie assistive, mentre la sicurezza protegge i confini tra dati non attendibili, browser e server.

## Indice

- [Testing dei componenti](#testing-dei-componenti)
- [Accessibilità nei componenti](#accessibilità-nei-componenti)
- [Sicurezza React](#sicurezza-react)

## Testing dei componenti

### Render e interazioni osservabili

**React Testing Library** è una libreria esterna che renderizza componenti nel Document Object Model (DOM) di test e li interroga come farebbe un utente. Preferire `screen.getByRole` con nome accessibile o `getByLabelText` ai selettori di classi e allo stato interno: un refactoring dell'implementazione non dovrebbe rompere un test se il comportamento resta uguale.

```jsx
import { useState } from 'react';
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

function Contatore() {
  const [n, setN] = useState(0);
  return <button onClick={() => setN(n + 1)}>Conteggio: {n}</button>;
}

test('incrementa il conteggio', async () => {
  const user = userEvent.setup();
  render(<Contatore />);
  await user.click(screen.getByRole('button', { name: 'Conteggio: 0' }));
  expect(screen.getByRole('button', { name: 'Conteggio: 1' })).toBeTruthy();
});
```

L'esempio presuppone un test runner con `test` ed `expect`. `render` restituisce anche utility per il componente montato; `screen` interroga il documento. `userEvent` riproduce una sequenza di interazione più vicina all'utente di un singolo evento sintetico. Verificare il risultato nel DOM e, se una callback è il contratto del componente, gli argomenti con cui viene chiamata. Per le [categorie generali di test](../../topics/engineering/15-architecture-devops.md#qualità-e-tradeoff) vedi Engineering.

### Asincronia, act e isolamento

| Query | Quando usarla | Esito se manca l'elemento |
| --- | --- | --- |
| `getBy...` | Deve essere già presente. | Lancia un errore. |
| `queryBy...` | Si vuole verificarne l'assenza. | Restituisce `null`. |
| `findBy...` | Compare dopo un aggiornamento asincrono. | Restituisce una Promise e ritenta; rifiuta dopo il timeout. |

Attendere le interazioni asincrone (`await user.click(...)`) e lo stato osservabile con `findBy...` o `waitFor`; per la scomparsa di un elemento usare `waitForElementToBeRemoved` solo se è inizialmente presente. `act(async () => ...)` applica gli aggiornamenti React prima delle asserzioni; le utility di React Testing Library lo usano già nei casi comuni, mentre chiamate dirette a React o timer controllati possono richiederlo esplicitamente. Evitare attese fisse.

Isolare rete, orologio o interfacce browser al confine del componente; non mockare Hook e logica interna che il test dovrebbe verificare. Con `<StrictMode>` i controlli di sviluppo possono ripetere render ed Effect: testare output e cleanup, non presumere un numero di chiamate valido solo in una configurazione. Per il [comportamento di StrictMode](03-effects-refs-hooks.md#strictmode-e-riesecuzione) vedi il capitolo Effect.

### Routing, form ed error boundary

- **Routing:** renderizzare con il provider del router scelto e un URL (Uniform Resource Locator) iniziale noto; verificare navigazione, parametri e vista risultante. Un test di componente può usare un router in memoria, mentre test di integrazione o nel browser servono per la configurazione reale e il caricamento dati.
- **Form:** interagire tramite label e pulsanti, poi verificare validazione, stato pending, successo ed errore. Le [Form Actions](02-state-events-forms.md#form-actions-e-stato-pending) e i loro effetti server richiedono un test di integrazione del confine di rete oltre al comportamento locale.
- **Error Boundary:** montare un figlio che fallisce durante il render e verificare il fallback; gestire o silenziare solo il log atteso nel test. Il boundary non intercetta normalmente errori in event handler o callback asincrone: vedi i suoi [limiti](04-context-state-architecture.md#error-boundary).

### E2E

Un test **E2E (End-to-End)** attraversa browser, router, componenti e backend come un utente. Riservarlo a flussi critici, per esempio navigazione, invio di un form e recupero da un errore; è più lento e sensibile a dati e ambiente rispetto a un test di componente. Per i trade-off generali vedi [Qualità e tradeoff](../../topics/engineering/15-architecture-devops.md#qualità-e-tradeoff).

## Accessibilità nei componenti

### Semantica, label e tastiera

JSX (JavaScript XML) mantiene la semantica degli elementi HTML (HyperText Markup Language): usare `<button>` per azioni e `<a href>` per navigazione, anziché un `<div>` con click. In React una label si associa all'input con `htmlFor` e un `id` stabile; associare anche i messaggi d'errore con `aria-describedby`, senza affidarsi al solo placeholder o colore. Per le [regole HTML e WCAG (Web Content Accessibility Guidelines)](../../topics/web/01-html-web.md#accessibilità) vedi la guida Web.

Verificare che controlli e stati siano raggiungibili e attivabili da tastiera, con focus visibile e ordine logico. Quando una UI React mostra, sposta o rimuove elementi, gestire il focus se il contesto cambia (per esempio aprendo un dialog o dopo una navigazione); una `ref` permette di chiamare `focus()` dopo il commit quando necessario. Gli attributi **ARIA (Accessible Rich Internet Applications)** descrivono ruolo e stato ma non aggiungono da soli comportamento da tastiera.

### Annunci e portali

Per un messaggio di stato inserito senza spostare il focus, una live region `aria-live="polite"` o `role="status"` può annunciare l'aggiornamento; per errori urgenti valutare `role="alert"` senza produrre annunci ripetuti. Verificare con tecnologie assistive reali quando il flusso è importante.

Un [portale](01-components-jsx-rendering.md#portali-e-albero-react) sposta i nodi nel DOM, non il loro posto nell'albero React; non gestisce automaticamente l'accessibilità. In un dialog modale, dare nome al dialog, portare il focus al suo interno, mantenerlo lì durante l'apertura e restituirlo a un punto logico alla chiusura; testare Tab, Shift+Tab ed Escape.

## Sicurezza React

### Escaping e HTML non attendibile

React inserisce i valori testuali come testo nelle espressioni JSX, evitando che una stringa utente diventi markup eseguibile. `dangerouslySetInnerHTML={{ __html: html }}` invece inserisce **HTML grezzo**: usarlo solo quando necessario e sanitizzare il contenuto non attendibile con una soluzione adatta al contesto. Questa protezione del testo non copre HTML grezzo, manipolazioni dirette del DOM o autorizzazione lato server; per [XSS (Cross-Site Scripting) e difese browser](../../topics/web/01-html-web.md#isolamento-e-sicurezza) vedi Web.

Non considerare un test del solo testo JSX una prova che tutte le sorgenti di contenuto siano sicure. Se l'app visualizza HTML da utenti o servizi esterni, testare il percorso reale di sanitizzazione e i punti in cui il contenuto viene inserito.

### Confini server e autorizzazione

Una [Server Function](06-performance-server-rendering.md#server-functions) è invocabile dal client attraverso la rete: argomenti, `FormData` e identità dichiarate dal browser sono input non attendibili. Sul server validare forma e valori, autenticare l'utente e autorizzare **ogni operazione sulla risorsa richiesta** prima di leggere o mutare dati; non restituire segreti nel risultato serializzato.

Nascondere un pulsante, proteggere una route client o disabilitare un form migliora la UI, ma non impedisce una chiamata costruita direttamente. Testare almeno il rifiuto di input non valido e di un utente senza permessi; per i confini generali di sicurezza web vedi [HTTP e sicurezza browser](../../topics/web/01-html-web.md#isolamento-e-sicurezza).
