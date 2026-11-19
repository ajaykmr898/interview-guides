# Esempio TodoList end-to-end Express

## Indice

- [Modello e contratto REST](#modello-e-contratto-rest)
- [Struttura e responsabilità](#struttura-e-responsabilità)
- [Flusso CRUD e persistenza](#flusso-crud-e-persistenza)
- [Validazione, errori e cleanup](#validazione-errori-e-cleanup)
- [Test rappresentativo](#test-rappresentativo)

## Modello e contratto REST

Una TodoList espone operazioni CRUD (Create, Read, Update, Delete) su `Todo { id: string, title: string, completed: boolean }`. Il server genera `id`, imposta `completed: false` alla creazione e restituisce oggetti con le stesse proprietà in ogni risposta. Il client invia e riceve JSON (JavaScript Object Notation).

| Operazione | Input | Risposta |
| --- | --- | --- |
| `GET /todos` | Nessuno | `200` con `Todo[]` |
| `POST /todos` | `{ "title": string }` | `201` con il nuovo `Todo` |
| `PATCH /todos/{id}` | `{ "title"?: string, "completed"?: boolean }`; almeno un campo | `200` con il `Todo` aggiornato |
| `DELETE /todos/{id}` | ID nel percorso | `204` senza corpo |

Input non valido produce `400`, ID inesistente `404`, guasti interni `500`. Per la semantica HTTP (Hypertext Transfer Protocol) del contratto, vedi [RESTful APIs](../../topics/api/16-restful-apis.md#contratto-http-delle-risorse).

## Struttura e responsabilità

Esempio CommonJS per Express 5, con `pg` per PostgreSQL e Supertest per il test HTTP.

```text
db/001_todos.sql
src/todos/{router,controller,service,repository}.cjs
src/{app,server}.cjs
test/todos.test.cjs
```

Il router associa metodo e percorso al controller; il controller traduce tra HTTP e service; il service applica le regole del dominio; il repository esegue le query. Questi layer sono una scelta applicativa, non parti obbligatorie di Express. Qui il repository serve a isolare PostgreSQL e a sostituire la persistenza nel test; per responsabilità e alternative vedi [Architettura e persistenza](05-data-architecture-persistence.md#responsabilità-dei-layer).

### Router e controller

**`src/todos/router.cjs`** — il parser JSON è applicato alle sole richieste con corpo:

```js
const express = require('express');
const { makeController } = require('./controller.cjs');

function makeRouter(service) {
  const router = express.Router();
  const c = makeController(service);
  router.get('/', c.list);
  router.post('/', express.json({ limit: '16kb' }), c.create);
  router.patch('/:id', express.json({ limit: '16kb' }), c.update);
  router.delete('/:id', c.remove);
  return router;
}
module.exports = { makeRouter };
```

**`src/todos/controller.cjs`** — gli handler restituiscono la Promise, così Express 5 inoltra le rejection al middleware degli errori:

```js
function makeController(service) {
  return {
    list: async (_req, res) => res.json(await service.list()),
    create: async (req, res) => res.status(201).json(await service.create(req.body)),
    update: async (req, res) => res.json(await service.update(req.params.id, req.body)),
    remove: async (req, res) => {
      await service.remove(req.params.id);
      res.status(204).end();
    },
  };
}
module.exports = { makeController };
```

### Service e repository

**`src/todos/service.cjs`** — valida l'input al confine del dominio; `false` è un valore valido per `completed`. Gli errori applicativi restano indipendenti da Express:

```js
const { randomUUID } = require('node:crypto');
const invalid = () => Object.assign(new Error('Input non valido'), { code: 'INVALID' });
const missing = () => Object.assign(new Error('Todo inesistente'), { code: 'NOT_FOUND' });
const object = value => value !== null && typeof value === 'object' && !Array.isArray(value);
function title(value) {
  if (typeof value !== 'string' || !value.trim()) throw invalid();
  return value.trim();
}
function makeService(repo) {
  return {
    list: () => repo.list(),
    create: body => {
      if (!object(body) || Object.keys(body).some(k => k !== 'title')) throw invalid();
      return repo.create(randomUUID(), title(body.title));
    },
    update: async (id, body) => {
      if (!object(body) || !Object.keys(body).length ||
          Object.keys(body).some(k => !['title', 'completed'].includes(k))) throw invalid();
      const patch = {};
      if (Object.hasOwn(body, 'title')) patch.title = title(body.title);
      if (Object.hasOwn(body, 'completed')) {
        if (typeof body.completed !== 'boolean') throw invalid();
        patch.completed = body.completed;
      }
      const todo = await repo.update(id, patch);
      if (!todo) throw missing();
      return todo;
    },
    remove: async id => { if (!(await repo.remove(id))) throw missing(); },
  };
}
module.exports = { makeService };
```

Il repository usa query parametrizzate: i valori del client non entrano nel testo SQL (Structured Query Language). Il driver e il pool sono dipendenze dell'applicazione, non di Express.

## Flusso CRUD e persistenza

La tabella conserva il modello condiviso. Applicare lo schema prima di avviare il server; il vincolo `NOT NULL` e il default rafforzano le regole lato server:

**`db/001_todos.sql`**

```sql
CREATE TABLE todos (
  id text PRIMARY KEY,
  title text NOT NULL,
  completed boolean NOT NULL DEFAULT false
);
```

**`src/todos/repository.cjs`** — ogni operazione usa una query singola del pool `pg`; `RETURNING` evita una seconda lettura. L'assenza di righe distingue l'ID sconosciuto:

```js
function makeRepository(pool) {
  return {
    list: async () => (await pool.query(
      'SELECT id, title, completed FROM todos ORDER BY id')).rows,
    create: async (id, title) => (await pool.query(
      'INSERT INTO todos (id, title) VALUES ($1, $2) RETURNING id, title, completed',
      [id, title])).rows[0],
    update: async (id, patch) => (await pool.query(
      `UPDATE todos SET title = COALESCE($2, title),
       completed = COALESCE($3, completed)
       WHERE id = $1 RETURNING id, title, completed`,
      [id, patch.title ?? null, patch.completed ?? null])).rows[0] ?? null,
    remove: async id => (await pool.query(
      'DELETE FROM todos WHERE id = $1', [id])).rowCount > 0,
  };
}
module.exports = { makeRepository };
```

Il flusso è `richiesta → router → controller → service → repository → PostgreSQL → risposta`. `POST` genera un ID e il database applica il default `false`; `PATCH` aggiorna atomicamente i soli campi ricevuti; `DELETE` verifica l'esistenza prima di emettere `204`. `COALESCE` qui distingue i campi omessi perché il service rifiuta esplicitamente `null`.

## Validazione, errori e cleanup

Il service rimuove gli spazi esterni da `title`, rifiuta titoli vuoti, campi inattesi, `completed` non booleano e PATCH senza campi. Il controller non espone dettagli del database. Per il funzionamento generale di validazione e pipeline degli errori vedi [Request e validazione](02-request-response-validation.md#validazione) e [Errori asincroni](03-async-errors-lifecycle.md#pipeline-degli-errori).

**`src/app.cjs`** — l'handler finale mappa gli errori del dominio e del parser JSON; i guasti inattesi sono registrati ma non restituiti al client:

```js
const express = require('express');
const { makeRouter } = require('./todos/router.cjs');
function createApp(service) {
  const app = express();
  app.use('/todos', makeRouter(service));
  app.use((err, _req, res, next) => {
    if (res.headersSent) return next(err);
    const status = err.code === 'INVALID' ? 400 : err.code === 'NOT_FOUND' ? 404
      : Number.isInteger(err.status) && err.status >= 400 && err.status < 500
        ? err.status : 500;
    if (status === 500) console.error(err);
    const message = status === 500 ? 'Errore interno' : status === 404
      ? 'Todo inesistente' : status === 413 ? 'Corpo troppo grande' : 'Input non valido';
    res.status(status).json({ error: message });
  });
  return app;
}
module.exports = { createApp };
```

**`src/server.cjs`** — una sola istanza del pool è condivisa; allo stop il server termina le richieste attive prima di chiudere il pool. `pool.query()` rilascia il client dopo ciascuna query, anche se fallisce; il listener registra gli errori dei client inattivi:

```js
const { Pool } = require('pg');
const { createApp } = require('./app.cjs');
const { makeService } = require('./todos/service.cjs');
const { makeRepository } = require('./todos/repository.cjs');
if (!process.env.DATABASE_URL) throw new Error('DATABASE_URL mancante');
const pool = new Pool({ connectionString: process.env.DATABASE_URL });
pool.on('error', err => console.error(err));
const server = createApp(makeService(makeRepository(pool))).listen(3000);
for (const signal of ['SIGINT', 'SIGTERM']) process.once(signal, () => {
  server.close(() => pool.end().catch(err => { console.error(err); process.exitCode = 1; }));
});
```

## Test rappresentativo

Il test HTTP attraversa router, controller e service. Un repository controllato verifica il contratto senza dipendere da PostgreSQL; un test separato con database reale può coprire le query. Vedi [Test Express](06-testing-operations-migrations.md#test).

**`test/todos.test.cjs`** — con `node:test`, `node:assert` e Supertest:

```js
const { test } = require('node:test');
const assert = require('node:assert/strict');
const request = require('supertest');
const { createApp } = require('../src/app.cjs');
const { makeService } = require('../src/todos/service.cjs');

test('POST /todos valida e crea', async () => {
  const repo = { create: async (id, title) => ({ id, title, completed: false }) };
  const app = createApp(makeService(repo));
  const created = await request(app).post('/todos').send({ title: ' Studiare ' });
  assert.equal(created.status, 201);
  assert.equal(typeof created.body.id, 'string');
  assert.equal(created.body.title, 'Studiare');
  assert.equal(created.body.completed, false);
  const invalid = await request(app).post('/todos').send({ title: '   ' });
  assert.equal(invalid.status, 400);
});
```
