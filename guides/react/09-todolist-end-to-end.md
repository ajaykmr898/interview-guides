# Esempio TodoList end-to-end React

Una **TodoList end-to-end** collega un form e una lista React a un'API (Application Programming Interface) remota: il componente gestisce le interazioni, un custom Hook coordina stato e richieste, e un client HTTP (Hypertext Transfer Protocol) isola il trasporto. L'esempio assume un'app renderizzata nel browser, senza router o store globale.

## Indice

- [Modello e contratto REST](#modello-e-contratto-rest)
- [Struttura e responsabilità](#struttura-e-responsabilità)
- [Flusso CRUD e stato](#flusso-crud-e-stato)
- [Form, errori e cleanup](#form-errori-e-cleanup)
- [Test rappresentativo](#test-rappresentativo)

## Modello e contratto REST

Il modello condiviso è `Todo { id: string, title: string, completed: boolean }`. Il contratto REST (Representational State Transfer) scambia JSON (JavaScript Object Notation) sulla stessa origine; il server assegna `id` e imposta `completed` a `false` alla creazione.

| Operazione | Richiesta | Risposta |
| --- | --- | --- |
| Lettura | `GET /todos` | `Todo[]` |
| Creazione | `POST /todos` con `{ "title": "..." }` | `201` con `Todo` |
| Modifica | `PATCH /todos/{id}` con `title` e/o `completed` | `Todo` aggiornato |
| Eliminazione | `DELETE /todos/{id}` | `204` senza corpo |

Il server elimina gli spazi ai bordi e richiede un titolo non vuoto, accetta `completed` solo se booleano e richiede almeno un campo nel `PATCH`. Restituisce `400` per input non valido, `404` per ID sconosciuto e un errore `5xx` per guasti interni. La validazione del client migliora il feedback, ma non sostituisce quella del server; il contratto generale è in [API RESTful](../../topics/api/16-restful-apis.md).

## Struttura e responsabilità

```text
src/
  main.tsx                  # monta l'app
  todos/
    todo.ts                 # modello e tipo del PATCH
    todos-api.ts            # richieste HTTP e status
    useTodos.ts             # stato, caricamento e mutazioni
    TodosPage.tsx           # form e lista JSX
    TodosPage.test.tsx      # interazione con rete controllata
```

I componenti JSX (JavaScript XML) descrivono la vista. Non serve un repository o un service separato: non c'è logica di business riusabile oltre alla normalizzazione del titolo nel Hook. Se la stessa regola diventasse condivisa tra più flussi, potrebbe uscire in un service; per [stato locale e server state](04-context-state-architecture.md#local-state-context-e-server-state) vedere il capitolo dedicato.

### Componenti JSX e API client

Il client trasforma le risposte HTTP in `Todo` o `Todo[]` e non legge JSON dopo il `204`. `fetch` restituisce una `Promise<Response>` anche per `400`, `404` o `500`: il controllo di `ok` converte questi status in errori gestibili; un guasto di rete rifiuta direttamente la Promise. I tipi TypeScript descrivono il contratto atteso, ma non validano il JSON ricevuto a runtime.

```ts
// todos/todo.ts
export type Todo = { id: string; title: string; completed: boolean };
export type TodoPatch = { title: string; completed?: boolean } | { title?: string; completed: boolean };
```

```ts
// todos/todos-api.ts
import type { Todo, TodoPatch } from './todo';

async function send(path: string, init?: RequestInit): Promise<Response> {
  const response = await fetch(`/todos${path}`, init);
  if (!response.ok) throw new Error(`HTTP ${response.status}`);
  return response;
}

export const todosApi = {
  async list(signal: AbortSignal): Promise<Todo[]> {
    return (await send('', { signal })).json() as Promise<Todo[]>;
  },
  async create(title: string): Promise<Todo> {
    return (await send('', { method: 'POST', headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ title }) })).json() as Promise<Todo>;
  },
  async update(id: string, patch: TodoPatch): Promise<Todo> {
    return (await send(`/${encodeURIComponent(id)}`, { method: 'PATCH',
      headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(patch) })).json() as Promise<Todo>;
  },
  async remove(id: string): Promise<void> {
    await send(`/${encodeURIComponent(id)}`, { method: 'DELETE' });
  },
};
```

La pagina possiede solo il testo del form e l'ID in modifica; lista e stato delle richieste appartengono a `useTodos`. Usa una key stabile fornita dal server e aggiorna la vista tramite callback degli handler.

```tsx
// todos/TodosPage.tsx
import { useState, type FormEvent } from 'react';
import type { Todo } from './todo';
import { useTodos } from './useTodos';

export function TodosPage() {
  const { todos, loading, saving, error, add, rename, toggle, remove } = useTodos();
  const [title, setTitle] = useState('');
  const [editingId, setEditingId] = useState<string | null>(null);
  const invalid = title.length > 0 && !title.trim();
  async function submit(event: FormEvent<HTMLFormElement>) {
    event.preventDefault();
    if (!title.trim()) return;
    const ok = editingId ? await rename(editingId, title) : await add(title);
    if (ok) { setTitle(''); setEditingId(null); }
  }
  async function deleteTodo(todo: Todo) {
    if (await remove(todo) && editingId === todo.id) { setTitle(''); setEditingId(null); }
  }
  return <section>
    <h1>TodoList</h1>
    <form onSubmit={submit}>
      <label htmlFor="todo-title">Titolo</label>
      <input id="todo-title" value={title} required aria-invalid={invalid}
        onChange={event => setTitle(event.target.value)} />
      <button type="submit" disabled={loading || saving || !title.trim()}>
        {editingId ? 'Salva' : 'Aggiungi'}
      </button>
      {editingId && <button type="button" onClick={() => { setEditingId(null); setTitle(''); }}>Annulla</button>}
      {invalid && <p role="alert">Titolo obbligatorio</p>}
    </form>
    {loading && <p role="status">Caricamento...</p>}
    {error && <p role="alert">{error}</p>}
    {!loading && todos.length === 0 && <p>Nessun Todo</p>}
    <ul>{todos.map(todo => <li key={todo.id}>
      <label><input type="checkbox" checked={todo.completed} disabled={loading || saving}
        onChange={() => void toggle(todo)} />{todo.title}</label>
      <button type="button" disabled={loading || saving} onClick={() => { setEditingId(todo.id); setTitle(todo.title); }}>Rinomina</button>
      <button type="button" disabled={loading || saving} onClick={() => void deleteTodo(todo)}>Elimina</button>
    </li>)}</ul>
  </section>;
}
```

### Custom Hook e logica applicativa

`useTodos` è chiamato al top level del componente. Tiene la lista come unica fonte dei dati visualizzati; il setter funzionale produce un nuovo array dopo ogni risposta confermata. `loading` riguarda la lettura iniziale, `saving` disabilita le altre mutazioni nella pagina, `error` espone i fallimenti. `add`, `rename`, `toggle` e `remove` restituiscono una `Promise<boolean>`: `true` indica che la risposta è stata applicata, `false` che l'operazione è fallita.

```ts
// todos/useTodos.ts
import { useEffect, useState } from 'react';
import type { Todo } from './todo';
import { todosApi } from './todos-api';

const message = (cause: unknown) => cause instanceof Error ? cause.message : 'Richiesta non riuscita';
function titleOrThrow(title: string) {
  const value = title.trim();
  if (!value) throw new Error('Titolo obbligatorio');
  return value;
}

export function useTodos() {
  const [todos, setTodos] = useState<Todo[]>([]);
  const [loading, setLoading] = useState(true);
  const [saving, setSaving] = useState(false);
  const [error, setError] = useState<string | null>(null);
  useEffect(() => {
    const controller = new AbortController();
    let active = true;
    todosApi.list(controller.signal)
      .then(items => { if (active) setTodos(items); })
      .catch(cause => { if (active) setError(message(cause)); })
      .finally(() => { if (active) setLoading(false); });
    return () => { active = false; controller.abort(); };
  }, []);
  async function run(work: () => Promise<void>): Promise<boolean> {
    setSaving(true); setError(null);
    try { await work(); return true; }
    catch (cause) { setError(message(cause)); return false; }
    finally { setSaving(false); }
  }
  const add = (title: string) => run(async () => {
    const created = await todosApi.create(titleOrThrow(title));
    setTodos(items => [...items, created]);
  });
  const rename = (id: string, title: string) => run(async () => {
    const updated = await todosApi.update(id, { title: titleOrThrow(title) });
    setTodos(items => items.map(item => item.id === id ? updated : item));
  });
  const toggle = (todo: Todo) => run(async () => {
    const updated = await todosApi.update(todo.id, { completed: !todo.completed });
    setTodos(items => items.map(item => item.id === todo.id ? updated : item));
  });
  const remove = (todo: Todo) => run(async () => {
    await todosApi.remove(todo.id);
    setTodos(items => items.filter(item => item.id !== todo.id));
  });
  return { todos, loading, saving, error, add, rename, toggle, remove };
}
```

L'Effect è riservato alla lettura al mount; le mutazioni partono dagli handler, non dal render. Il cleanup abortisce il `GET` e il flag `active` ignora anche una risposta già arrivata dopo lo smontaggio; con una dipendenza di route o contesto, lo stesso cleanup precederebbe il nuovo caricamento. Per [Effect e race](03-effects-refs-hooks.md#useeffect-dipendenze-e-cleanup) vedere il capitolo dedicato.

## Flusso CRUD e stato

CRUD (Create, Read, Update, Delete) segue un flusso unidirezionale: la pagina chiama il Hook, il Hook invoca `todosApi`, e la risposta del server aggiorna `todos`.

| Interazione | Richiesta | Aggiornamento confermato |
| --- | --- | --- |
| Apertura pagina | `GET /todos` | Sostituisce la lista. |
| Invio form nuovo | `POST /todos` | Aggiunge il `Todo` con ID assegnato dal server. |
| Rinomina o checkbox | `PATCH /todos/{id}` | Sostituisce l'elemento con quello restituito dal server. |
| Elimina | `DELETE /todos/{id}` | Rimuove l'elemento dopo una risposta positiva (contratto: `204`). |

Un errore lascia invariata la lista locale: l'esempio usa aggiornamenti **confermati**, quindi non richiede rollback ottimistico. Lo stato del form resta nella pagina; non occorrono Context né cache globale per un'unica vista. Per [ownership e aggiornamenti immutabili](02-state-events-forms.md#oggetti-array-e-stato-derivato) vedere il capitolo sullo stato.

## Form, errori e cleanup

L'input controlled usa `value` e `onChange`; `required` e il pulsante disabilitato segnalano l'assenza di titolo, mentre `titleOrThrow` elimina gli spazi e rifiuta un valore vuoto anche se il Hook viene chiamato da un altro componente. Il server ripete la validazione e controlla i permessi. `aria-invalid`, la label e i messaggi con `role="alert"` rendono il feedback percepibile; per [semantica dei form](../../topics/web/01-html-web.md#form-e-validazione) vedere HTML e Web.

Un `400` o `404` produce `HTTP 400` o `HTTP 404`; un `5xx`, un guasto di rete o un JSON malformato viene catturato dal Hook e mostrato senza rimuovere i Todo già confermati. Durante `loading` o `saving` i controlli di mutazione sono disabilitati. `AbortController` riguarda il caricamento: una mutazione già inviata può comunque essere applicata dal server anche se la pagina viene chiusa, perciò un nuovo mount rilegge la lista. Un fetch in Effect non offre da solo cache, deduplicazione o dati nel rendering server; per app più grandi valutare [data fetching del framework o cache client](05-data-suspense-routing.md#fetch-nel-client-e-suoi-limiti).

## Test rappresentativo

Il test controlla il confine di rete con un mock del client, attende il caricamento e verifica un'interazione osservabile: il titolo inviato è normalizzato e il Todo confermato appare nella pagina. Presuppone Vitest, React Testing Library e `user-event`, già descritti nel [capitolo testing](07-testing-accessibility-security.md#render-e-interazioni-osservabili).

```tsx
// todos/TodosPage.test.tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { test, expect, vi } from 'vitest';
import { todosApi } from './todos-api';
import { TodosPage } from './TodosPage';

vi.mock('./todos-api', () => ({ todosApi: {
  list: vi.fn(), create: vi.fn(), update: vi.fn(), remove: vi.fn(),
} }));

test('crea e mostra il Todo confermato', async () => {
  vi.mocked(todosApi.list).mockResolvedValue([]);
  vi.mocked(todosApi.create).mockResolvedValue({ id: 't1', title: 'Studiare', completed: false });
  const user = userEvent.setup();
  render(<TodosPage />);
  await screen.findByText('Nessun Todo');
  await user.type(screen.getByLabelText('Titolo'), ' Studiare ');
  await user.click(screen.getByRole('button', { name: 'Aggiungi' }));
  expect(todosApi.create).toHaveBeenCalledWith('Studiare');
  expect(await screen.findByText('Studiare')).toBeTruthy();
});
```

Il test non dimostra che il server rispetti il contratto: `400`, `404`, `204` e validazione server richiedono test dell'API. La distinzione tra test di componente e integrazione è in [Testing dei componenti](07-testing-accessibility-security.md#testing-dei-componenti).
