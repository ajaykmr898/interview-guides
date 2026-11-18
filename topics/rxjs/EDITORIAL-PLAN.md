# Piano editoriale — rxjs

Requisiti specifici dei capitoli di questa directory. Per le regole editoriali comuni e i criteri di accettazione, seguire [AGENTS.md](../../AGENTS.md).

## `10-rxjs.md` — RxJS e Reactive Programming

**Da coprire (capitolo di riferimento completo):**
- Reactive programming, stream, push model; Observable, Observer, Subscription, teardown e unsubscribe.
- Cold vs Hot Observable, unicast vs multicast; lazy execution, completion vs unsubscription.
- Subject, BehaviorSubject, ReplaySubject, AsyncSubject, relativi comportamenti e trade-off.
- Creation: `of`, `from`, `defer`, `interval`, `timer`, `EMPTY`, `throwError`.
- Transformation: `map`, `scan`, `tap`, `switchMap`, `mergeMap`, `concatMap`, `exhaustMap`.
- Filtering/control: `filter`, `take`, `takeUntil`, `first`, `debounceTime`, `throttleTime`, `distinctUntilChanged`, `auditTime`.
- Combination: `combineLatest`, `forkJoin`, `zip`, `withLatestFrom`, `startWith`.
- Error/finalization: `catchError`, `retry`, `finalize`, completion/error propagation.
- Sharing/multicasting: `share`, `shareReplay`; caching pitfalls e refCount (quando rilevante).
- `pipe`, operator composition, higher-order Observable, `switchMap`/`mergeMap`/`concatMap`/`exhaustMap` in una tabella comparativa unica.
- Schedulers, marble diagrams/testing come definizioni sintetiche; memory leaks, nested subscriptions e cleanup.
- Promise vs Observable vs Subscription; Subject vs BehaviorSubject; quando preferire RxJS rispetto a uno stato locale semplice.

**Snippet obbligatori:** subscribe/unsubscribe, `pipe(map(...))`, confronto flattening con casi d'uso, semplice error handling.

**Confine:** `async` pipe, `takeUntilDestroyed`, `toSignal`/`toObservable` sono spiegati nel capitolo Angular su interoperabilità RxJS, HTTP e Router.
