# Piano editoriale — API RESTful

Requisiti del capitolo di questa directory. Per le regole comuni seguire [AGENTS.md](../../AGENTS.md).

## `16-restful-apis.md` — API RESTful

**Da coprire (tutte le priorità sono obbligatorie):**
- **P0 — Vincoli architetturali REST:** definire API RESTful ed espandere API e REST alla prima occorrenza; client-server, stateless, cacheable, layered system e uniform interface, con code-on-demand opzionale; distinguere REST da una generica API HTTP e da endpoint RPC.
- **P0 — Uniform interface:** articolare i suoi quattro vincoli, distinti dall'insieme dei vincoli architetturali REST: identificazione delle risorse; manipolazione delle risorse tramite rappresentazioni; messaggi autodescrittivi; hypermedia as the engine of application state (HATEOAS).
- **P0 — Design delle risorse:** identità e URI, rappresentazioni e link tra risorse; ruolo pratico della hypermedia e limiti delle API che non la espongono.
- **P0 — Operazioni:** semantica delle risorse con `GET`, `POST`, `PUT`, `PATCH` e `DELETE`; safety e idempotenza, inclusa la distinzione fra `PUT` e `PATCH`.
- **P0 — Risposte:** status code, header e `Location` nel contratto delle operazioni; distinguere errori client, conflitti e guasti server senza ripetere il catalogo HTTP generale.
- **P1 — Dati ed errori:** media type, content negotiation, validazione degli input e risposte d'errore coerenti; Problem Details e limiti della sua adozione.
- **P1 — Collezioni:** filtri, ordinamento e paginazione offset/cursor con stabilità dell'ordinamento e trade-off.
- **P1 — Cache e concorrenza:** cache delle rappresentazioni, `ETag` e richieste condizionali; aggiornamenti concorrenti con `If-Match` e risposta a precondizioni fallite.
- **P1 — Evoluzione:** versioning, cambiamenti compatibili, deprecazione e consumer esistenti; evitare versioni inutili per ogni modifica.
- **P1 — Sicurezza del contratto:** requisiti di autenticazione, autorizzazione per risorsa/operazione e distinzione tra risposte `401`/`403`; rate limiting come quota e risposta `429` con eventuale `Retry-After`, senza dettagli di middleware.
- **P1 — Verifica del contratto:** OpenAPI e contract test; distinguere la descrizione del contratto dalla verifica del comportamento.
- **P2 — Affidabilità delle operazioni:** retry sicuri, idempotency key per operazioni non idempotenti e operazioni asincrone con stato consultabile.

**Confronti/snippet per il completamento:** API RESTful vs API HTTP/RPC, `PUT` vs `PATCH`, paginazione offset vs cursor; esempi brevi di risorsa, errore e aggiornamento condizionale solo quando aggiungono valore.

**Confini:** per semantica generale di HTTP, cache e sicurezza browser vedi [HTML e Web](../web/01-html-web.md#http-https-e-cache) e [isolamento e sicurezza](../web/01-html-web.md#isolamento-e-sicurezza); per routing, middleware e implementazione di autenticazione e rate limiting in Node vedi [Node e npm](../node-tooling/08-node-npm.md#backend-http-ed-express); per contratti e categorie di test vedi [Architettura e DevOps](../engineering/15-architecture-devops.md#api-e-contratti). Qui approfondire le scelte specifiche del design delle risorse REST.
