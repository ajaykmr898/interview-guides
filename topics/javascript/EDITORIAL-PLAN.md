# Piano editoriale — javascript

Requisiti specifici dei capitoli di questa directory. Per le regole editoriali comuni e i criteri di accettazione, seguire [AGENTS.md](../../AGENTS.md).

## `03-javascript-basics.md` — JavaScript fondamentali

**Da coprire:**
- JS vs ECMAScript; engine/runtime; primitive vs object; `undefined`, `null`, `NaN`, `BigInt`, `Symbol`.
- `var`, `let`, `const`; lexical scope, global/function/block scope; hoisting, Temporal Dead Zone.
- Type coercion, truthy/falsy, `==`, `===`, `Object.is`, passaggio per valore e riferimenti a oggetti.
- Execution context, call stack, lexical environment, closure e relative implicazioni di memoria.
- Function declaration/expression, arrow function, callback, higher-order function, recursion, currying, IIFE.
- `this`, `bind`, `call`, `apply`; differenze tra arrow e regular functions; rest/spread, destructuring.
- Objects: proprietà, accessors, descriptors, prototype chain, prototypal inheritance, `class`, `extends`, `super`.
- Encapsulation, polymorphism, composition vs inheritance; immutability, shallow/deep copy, `structuredClone`, `Object.freeze`.
- Arrays: `map`, `filter`, `reduce`, `find`, `some`, `every`, `forEach`, `sort`, `slice`, `splice`, `flatMap`.
- Functional programming: pure functions, memoization, referential transparency; debounce vs throttle.
- Optional chaining `?.`, nullish coalescing `??`, default parameters e operatori fondamentali.

**Confronti/snippet:** `var`/`let`/`const`, `==`/`===`/`Object.is`, `map`/`forEach`, closure, `this` e copia superficiale/profonda.

**Confine:** event loop e Promise in `04`; moduli, collezioni avanzate, generatori e ECMAScript timeline in `05`.

## `04-javascript-async-dom.md` — Asincronia e DOM

**Da coprire:**
- Synchronous vs asynchronous; concurrency vs parallelism; callback e callback hell.
- Event loop, call stack, task queue/macrotasks, microtask queue, rendering opportunities; `setTimeout`, `setInterval`, `queueMicrotask`.
- Promise: pending/fulfilled/rejected, thenable, chaining, error propagation; `.then`, `.catch`, `.finally`.
- `Promise.all`, `allSettled`, `race`, `any`, `async`/`await`, `try`/`catch`; race conditions.
- AbortController e AbortSignal, cancellation di Fetch vs Promise genericamente.
- DOM tree, node vs element; `querySelector`, traversal, `createElement`, `append`, `remove`, `replaceChildren`.
- `textContent` vs `innerHTML` e relative implicazioni di sicurezza.
- Event handling: `addEventListener`, `removeEventListener`, bubbling, capturing, delegation, `target` vs `currentTarget`, `preventDefault` vs `stopPropagation`.
- CustomEvent, Shadow DOM (overview), event listeners e possibili memory leaks.
- Fetch API: richiesta, response, headers, JSON, status handling; nozione di `response.ok` e di errori di rete.
- Web APIs e loro relazione con il runtime browser, reflow/repaint e performance DOM.
- Confronto introduttivo **Promise vs Observable vs Subscription**, precisando che Observable/Subscription di RxJS non sono primitive native JS.

**Confronti/snippet:** ordine event loop, Promise combinators, `async`/`await`, listener con cleanup, Fetch con AbortController.

**Confine:** approfondimento RxJS nel capitolo generale RxJS.

## `05-javascript-advanced-ecmascript.md` — Avanzato ed ECMAScript

**Da coprire:**
- ES Modules (`import`/`export`, dynamic import) vs CommonJS; strict mode.
- `Map`, `Set`, `WeakMap`, `WeakSet`; `Symbol`, iterables, iterators, generators, `yield`, async iterators.
- `Proxy` e `Reflect`; property descriptors, getter/setter; garbage collection, reachability e weak references.
- JavaScript error types, custom errors, `Error.cause`, regular expressions, `Intl`, Date e gestione del tempo (overview).
- Memory/performance: event loop starvation, leaks, immutability, lazy loading e profilazione (overview).
- Design patterns rilevanti e limiti del linguaggio dinamico.
- TC39 ed evoluzione ECMAScript: ES5, ES2015 e principali funzionalità successive (senza presentare proposte come standard).
- Runtime compatibility, transpilation vs polyfill, feature detection, browser targets.

**Confronti/snippet:** `Map` vs object, `Set` vs array, ESM vs CJS, `WeakMap` vs `Map`, Babel/transpilation vs polyfill.
