# Esempio TodoList end-to-end Angular

Questo esempio collega un form Angular a un'API (Application Programming Interface) HTTP (Hypertext Transfer Protocol) per leggere, creare, completare ed eliminare attività. Il componente possiede lo stato della vista; un service iniettabile isola l'accesso ai dati. L'esempio usa aggiornamenti dopo la conferma del server, senza uno store globale.

## Indice

- [Modello e contratto REST](#modello-e-contratto-rest)
- [Struttura e responsabilità](#struttura-e-responsabilità)
- [Flusso CRUD e stato](#flusso-crud-e-stato)
- [Form, errori e cleanup](#form-errori-e-cleanup)
- [Test rappresentativo](#test-rappresentativo)

## Modello e contratto REST

Il modello condiviso è `Todo { id: string, title: string, completed: boolean }`. L'esempio assume rendering client e un contratto REST (Representational State Transfer) che scambia JSON (JavaScript Object Notation) sulla stessa origine dell'applicazione; il server assegna `id` e inizializza `completed` a `false`.

| Operazione | Richiesta | Risposta |
| --- | --- | --- |
| Lettura | `GET /todos` | `Todo[]` |
| Creazione | `POST /todos` con `{ "title": "..." }` | `201` con `Todo` |
| Modifica | `PATCH /todos/{id}` con `title` e/o `completed` | `Todo` aggiornato |
| Eliminazione | `DELETE /todos/{id}` | `204` senza corpo |

Il titolo viene ripulito con `trim()` e non può essere vuoto; `completed` deve essere booleano e il `PATCH` deve contenere almeno un campo. Il server verifica questi vincoli e restituisce `400` per input non valido, `404` per un identificatore assente e un errore `5xx` per un guasto. Per la semantica dei [metodi e status HTTP](../../topics/api/16-restful-apis.md#contratto-http-delle-risorse) vedi il capitolo canonico.

```ts
// todos/todo.model.ts
export interface Todo {
  id: string;
  title: string;
  completed: boolean;
}
```

## Struttura e responsabilità

```text
src/app/
  app.config.ts
  todos/todo.model.ts
  todos/todos-api.service.ts
  todos/todos-page.component.ts
  todos/todos-page.component.html
  todos/todos-api.service.spec.ts
```

`app.config.ts` registra `provideHttpClient()` nei provider dell'app; la route o il componente padre renderizza `<app-todos>`. Il service è l'unico punto che conosce gli endpoint; il componente coordina form, richieste e stato visibile. Una facade o un repository aggiungerebbero un livello utile solo se più viste condividessero workflow o cache: qui il service `HttpClient` è sufficiente.

```ts
// app.config.ts (estratto)
import { provideHttpClient } from '@angular/common/http';
import { ApplicationConfig } from '@angular/core';

export const appConfig: ApplicationConfig = { providers: [provideHttpClient()] };
```

### Template e componente

Il template collega un `FormControl` al campo, presenta gli errori e usa `@for` con l'identificatore stabile. I pulsanti inviano le modifiche e restano disabilitati durante un'operazione per evitare invii concorrenti.

```html
<!-- todos/todos-page.component.html -->
<form (submit)="add($event)">
  <label for="todo-title">Attività</label>
  <input id="todo-title" [formControl]="title" [readOnly]="loading() || busy()"
    [attr.aria-invalid]="title.touched && title.invalid"
    [attr.aria-describedby]="title.touched && title.invalid ? 'title-error' : null">
  @if (title.touched && title.invalid) { <p id="title-error">Inserisci un titolo.</p> }
  <button type="submit" [disabled]="loading() || busy()">Aggiungi</button>
</form>
@if (loading()) { <p role="status">Caricamento…</p> }
@if (error()) {
  <p role="alert">{{ error() }}</p>
  <button type="button" (click)="load()" [disabled]="loading() || busy()">Ricarica lista</button>
}
@if (!loading() && !error() && todos().length === 0) { <p>Nessuna attività</p> }
<ul>
  @for (todo of todos(); track todo.id) {
    <li>{{ todo.title }} ({{ todo.completed ? 'completata' : 'da fare' }})
      <button type="button" (click)="toggle(todo)" [disabled]="loading() || busy()">{{ todo.completed ? 'Riapri' : 'Completa' }} {{ todo.title }}</button>
      <button type="button" (click)="remove(todo)" [disabled]="loading() || busy()">Elimina {{ todo.title }}</button></li>
  }
</ul>
```

Il controllo valida anche stringhe di soli spazi, che `required` da solo non rifiuterebbe. La [validazione e l'accessibilità dei form](07-forms.md#stato-e-accessibilità) sono spiegate nel capitolo dedicato.

```ts
// todos/todos-page.component.ts
import { Component, DestroyRef, inject, signal } from '@angular/core';
import { HttpErrorResponse } from '@angular/common/http';
import { FormControl, ReactiveFormsModule } from '@angular/forms';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { Observable, finalize } from 'rxjs';
import { Todo } from './todo.model';
import { TodosApi } from './todos-api.service';

@Component({ selector: 'app-todos', standalone: true, imports: [ReactiveFormsModule], templateUrl: './todos-page.component.html' })
export class TodosPage {
  private readonly api = inject(TodosApi);
  private readonly destroyRef = inject(DestroyRef);
  readonly todos = signal<Todo[]>([]);
  readonly loading = signal(false);
  readonly busy = signal(false);
  readonly error = signal('');
  readonly title = new FormControl('', {
    nonNullable: true,
    validators: control => control.value.trim() ? null : { blank: true },
  });

  constructor() { this.load(); }

  load(): void {
    if (this.loading() || this.busy()) return;
    this.loading.set(true); this.error.set('');
    this.api.list().pipe(takeUntilDestroyed(this.destroyRef), finalize(() => this.loading.set(false)))
      .subscribe({ next: todos => this.todos.set(todos), error: err => this.error.set(this.message(err)) });
  }

  add(event: Event): void {
    event.preventDefault();
    this.title.markAsTouched();
    if (this.title.invalid) return;
    this.mutate(this.api.create(this.title.value.trim()), todo => {
      this.todos.update(todos => [...todos, todo]); this.title.reset();
    });
  }

  toggle(todo: Todo): void {
    this.mutate(this.api.update(todo.id, { completed: !todo.completed }), updated =>
      this.todos.update(todos => todos.map(item => item.id === updated.id ? updated : item)));
  }

  remove(todo: Todo): void {
    this.mutate(this.api.remove(todo.id), () =>
      this.todos.update(todos => todos.filter(item => item.id !== todo.id)));
  }

  private mutate<T>(request: Observable<T>, apply: (value: T) => void): void {
    if (this.loading() || this.busy()) return;
    this.busy.set(true); this.error.set('');
    request.pipe(takeUntilDestroyed(this.destroyRef), finalize(() => this.busy.set(false)))
      .subscribe({ next: apply, error: err => this.error.set(this.message(err)) });
  }

  private message(error: HttpErrorResponse): string {
    return error.status === 400 ? 'Dati non validi.'
      : error.status === 404 ? 'Attività non trovata.' : 'Operazione non riuscita.';
  }
}
```

### Service e accesso HTTP

`HttpClient` restituisce `Observable<T>` cold: la richiesta parte alla subscription, emette la risposta o un `HttpErrorResponse` e normalmente completa. Il tipo generico descrive il payload atteso a TypeScript, non valida il JSON ricevuto a runtime. `delete<void>` rappresenta la risposta `204` senza corpo. Per [HttpClient ed errori](06-rxjs-http-routing.md#gestione-degli-errori-http) vedi il capitolo HTTP Angular.

```ts
// todos/todos-api.service.ts
import { HttpClient } from '@angular/common/http';
import { Injectable, inject } from '@angular/core';
import { Todo } from './todo.model';

@Injectable({ providedIn: 'root' })
export class TodosApi {
  private readonly http = inject(HttpClient);
  list() { return this.http.get<Todo[]>('/todos'); }
  create(title: string) { return this.http.post<Todo>('/todos', { title }); }
  update(id: string, patch: Partial<Pick<Todo, 'title' | 'completed'>>) {
    return this.http.patch<Todo>(`/todos/${encodeURIComponent(id)}`, patch);
  }
  remove(id: string) { return this.http.delete<void>(`/todos/${encodeURIComponent(id)}`); }
}
```

## Flusso CRUD e stato

Il flusso CRUD (Create, Read, Update, Delete) resta unidirezionale: il template invoca un metodo del componente, il service invia la richiesta, poi il componente aggiorna il signal `todos` solo con la risposta confermata. `load()` sostituisce la lista; `add()` aggiunge il `Todo` creato con `id` del server; `toggle()` sostituisce l'elemento aggiornato; `remove()` lo elimina solo dopo il `204`. Un errore lascia invariato lo stato locale, quindi non serve rollback ottimistico.

`loading` riguarda la lettura iniziale o il retry, `busy` una mutazione. I controlli e la guardia in `mutate()` evitano mutazioni contemporanee in questo esempio; un'app con più form o aggiornamenti paralleli richiede stato pending per singola operazione e una politica per la concorrenza. Per [signal e aggiornamenti immutabili](05-signals-change-detection.md#signal-computed-ed-effect) vedi il capitolo stato.

## Form, errori e cleanup

Il validatore sincrono legge il `FormControl<string>` e restituisce `null` per un titolo valido oppure `{ blank: true }`; `markAsTouched()` rende visibile il messaggio dopo un tentativo di invio. `add()` invia la versione trim-mata e svuota il campo soltanto dopo il successo. Questa verifica migliora il feedback, mentre il server resta responsabile di validazione e autorizzazione.

`HttpErrorResponse.status` distingue qui `400` e `404`; errori di rete (`status === 0`) e guasti server usano il messaggio generico, con retry della lettura. `takeUntilDestroyed(this.destroyRef)` chiude le subscription alla distruzione del componente e interrompe le richieste `HttpClient` ancora in corso; `finalize` ripristina i flag anche su errore o cancellazione. L'annullamento lato client non garantisce che una mutazione già elaborata dal server venga annullata. Per [cleanup e testing degli errori](09-testing.md#test-http-e-routing) vedi i capitoli dedicati.

## Test rappresentativo

Un test del service verifica il confine di rete senza server reale: controlla metodo, URL (Uniform Resource Locator), body e risorsa restituita. `provideHttpClientTesting()` deve seguire `provideHttpClient()` quando sono entrambi presenti; `flush()` consegna la risposta simulata e `verify()` segnala richieste inattese.

```ts
// todos/todos-api.service.spec.ts
import { provideHttpClient } from '@angular/common/http';
import { HttpTestingController, provideHttpClientTesting } from '@angular/common/http/testing';
import { TestBed } from '@angular/core/testing';
import { expect, it } from 'vitest';
import { Todo } from './todo.model';
import { TodosApi } from './todos-api.service';

it('crea un Todo con il contratto HTTP', () => {
  TestBed.configureTestingModule({ providers: [provideHttpClient(), provideHttpClientTesting()] });
  const api = TestBed.inject(TodosApi);
  const http = TestBed.inject(HttpTestingController);
  let result: Todo | undefined;
  api.create('Studiare').subscribe(todo => result = todo);
  const request = http.expectOne({ method: 'POST', url: '/todos' });
  expect(request.request.body).toEqual({ title: 'Studiare' });
  const created: Todo = { id: 't1', title: 'Studiare', completed: false };
  request.flush(created, { status: 201, statusText: 'Created' });
  expect(result).toEqual(created);
  http.verify();
});
```
