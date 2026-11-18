# RxJS

RxJS (Reactive Extensions for JavaScript) è una libreria per comporre flussi di valori ed eventi nel tempo. Nel modello **reactive/push** è la sorgente a notificare i valori al consumer; le semantiche qui descritte seguono la documentazione della linea stabile RxJS 7.

## Indice

- [Observable e subscription](#observable-e-subscription)
- [Operatori di trasformazione](#operatori-di-trasformazione)
- [Operatori di filtro e controllo](#operatori-di-filtro-e-controllo)
- [Operatori di combinazione](#operatori-di-combinazione)
- [Gestione degli errori](#gestione-degli-errori)
- [Multicasting e Subject](#multicasting-e-subject)
- [Unsubscription e teardown](#unsubscription-e-teardown)
- [Scheduler e test](#scheduler-e-test)

## Observable e subscription

Un **Observable** descrive una sorgente che può inviare zero o più notifiche `next`, seguite al massimo da un `complete` oppure un `error`: dopo una notifica terminale non emette altro. L'**Observer** è il consumer con callback `next`, `error` e `complete`; `subscribe()` restituisce una **Subscription**, la risorsa con cui interrompere l'osservazione. Una sorgente può emettere sincronicamente (`of`) o asincronicamente (`interval`): Observable non significa automaticamente «asincrono».

Una sorgente **cold** crea un produttore per ogni subscription e in genere parte quando ci si iscrive; una sorgente **hot** produce anche indipendentemente dal singolo subscriber, per esempio un evento condiviso. **Unicast** invia ogni esecuzione a un consumer, **multicast** condivide una produzione tra più consumer: hot/cold descrive l'origine del produttore, unicast/multicast come i valori vengono distribuiti. `share()` può rendere condivisa una sorgente cold. `from(promise)` osserva una Promise che può essere già in esecuzione; usare `defer()` se anche la sua creazione deve avvenire alla subscription.

| Creazione | Comportamento essenziale |
| --- | --- |
| `of(a, b)` | Emette gli argomenti, normalmente in modo sincrono, poi completa; `of([a, b])` emette **un** array. |
| `from(input)` | Converte iterable o Promise in Observable; un array emette i suoi elementi, una Promise risolve/rifiuta in modo asincrono. |
| `defer(factory)` | Invoca la factory a ogni subscription e si iscrive alla sorgente restituita; utile per lavoro davvero lazy. |
| `interval(ms)` | Emette `0, 1, …` periodicamente dopo il primo intervallo; non completa da solo. |
| `timer(due, period?)` | Emette `0` dopo `due`; con `period` continua periodicamente, altrimenti completa. |
| `EMPTY` | Completa senza valori; non è `of(undefined)`. |
| `throwError(() => error)` | Invia `error` alla subscription senza valori, creando l'errore quando ci si iscrive. |

```ts
import { interval } from "rxjs";
const subscription = interval(1_000).subscribe({ next: value => console.log(value) });
setTimeout(() => subscription.unsubscribe(), 3_500);
```

Il timer dell'esempio libera la subscription; il numero preciso di emissioni dipende dalla schedulazione. Una [Promise JavaScript](../javascript/04-javascript-async-dom.md#promise-e-async-await) rappresenta un solo esito e non ha cancellazione generica; un Observable può emettere più valori, mentre la Subscription rappresenta la singola esecuzione osservata. `unsubscribe()` non trasforma la sorgente in una Promise annullabile: interrompe il lavoro sottostante solo se il teardown della sorgente lo prevede.

## Operatori di trasformazione

`pipe(op1, op2)` compone operatori nell'ordine indicato e restituisce un nuovo Observable senza avviare da solo la subscription. `map` trasforma ogni valore uno a uno; `scan` accumula uno stato e lo emette a ogni passo; `tap` osserva valori, errori o completion per side effect senza cambiare il valore emesso. Evitare logica che altera lo stato dentro `tap` quando può stare in una trasformazione esplicita.

```ts
import { of, map } from "rxjs";
const doubled$ = of(1, 2, 3).pipe(map(value => value * 2));
doubled$.subscribe(console.log); // 2, 4, 6; poi complete
```

Un **higher-order Observable** emette altri Observable. Gli operatori di flattening mappano ogni valore esterno a una sorgente interna e ne portano i valori nel flusso risultante; scegliere in base a sovrapposizione, ordine e perdita ammessa:

| Operatore | Nuovo valore mentre un inner è attivo | Caso d'uso e limite |
| --- | --- | --- |
| `switchMap` | Si disiscrive dall'inner precedente e passa al nuovo. | Ricerca in cui conta solo l'ultimo input; può scartare risultati e non garantisce l'annullamento del lavoro esterno. |
| `mergeMap` | Mantiene inner concorrenti; l'ordine dei risultati non è garantito. | Scritture indipendenti da conservare tutte; limitare la concorrenza se necessario. |
| `concatMap` | Accoda e avvia il nuovo inner dopo il completamento del precedente. | Operazioni da eseguire in ordine; la coda può crescere se gli inner sono lenti o non completano. |
| `exhaustMap` | Ignora i nuovi valori finché l'inner corrente non completa. | Evitare invii duplicati mentre un'azione è in corso; le richieste ignorate sono perse. |

```ts
import { of, timer, map, switchMap } from "rxjs";
of("a", "b").pipe(
  switchMap(term => timer(10).pipe(map(() => term)))
).subscribe(console.log); // solo "b": "a" è disiscritto prima del timer
```

La disiscrizione di un inner interrompe le sue notifiche; interrompe una richiesta o altro side effect solo se l'implementazione collega il teardown a quell'operazione. Con `mergeMap` e `concatMap`, un inner infinito può impedire completion e trattenere risorse.

## Operatori di filtro e controllo

| Operatore | Comportamento e distinzione |
| --- | --- |
| `filter(predicate)` | Passa i valori che soddisfano il predicato; non completa la sorgente se nessuno passa. |
| `take(n)` | Emette al massimo `n` valori, poi completa e si disiscrive dalla sorgente. |
| `takeUntil(notifier)` | Completa quando il notifier emette un valore; la sola completion del notifier non ferma la sorgente. |
| `first(predicate?)` | Emette il primo valore idoneo e completa; senza match prima della completion produce `EmptyError`, a differenza di `take(1)`. |
| `debounceTime(ms)` | Emette l'ultimo valore dopo un periodo di silenzio; utile per input di ricerca, con latenza aggiunta. |
| `throttleTime(ms)` | Emette di default il primo valore della finestra e sopprime i successivi; utile per limitare eventi frequenti. |
| `auditTime(ms)` | Emette l'ultimo valore alla fine di una finestra avviata da un'emissione; non attende un silenzio come `debounceTime`. |
| `distinctUntilChanged()` | Scarta duplicati **consecutivi**; per default confronta con `===`, quindi oggetti nuovi con gli stessi campi risultano distinti. |

Questi operatori controllano notifiche, non applicano backpressure alla sorgente: un produttore può continuare a lavorare anche se alcuni valori vengono scartati. Per il cleanup di stream infiniti, `take`, `takeUntil` o una disiscrizione esplicita devono raggiungere la subscription effettiva.

## Operatori di combinazione

| Operatore | Quando emette | Limite da ricordare |
| --- | --- | --- |
| `combineLatest([a$, b$])` | Dopo il primo valore di **ogni** sorgente, poi a ogni emissione di una qualsiasi, usando gli ultimi valori. | Se una sorgente completa senza emettere, completa senza valori; con sorgente che non emette resta in attesa. |
| `forkJoin([a$, b$])` | Una volta, con gli ultimi valori, quando **tutte** completano. | Una sorgente infinita impedisce l'emissione; una sorgente vuota fa completare senza risultato. |
| `zip(a$, b$)` | Accoppia primo con primo, secondo con secondo, ecc. | Può accumulare valori della sorgente più veloce; completa quando non può formare altre coppie. |
| `withLatestFrom(b$)` | Solo quando emette la sorgente principale, dopo che `b$` ha emesso almeno una volta. | Le sole emissioni di `b$` non producono output. |
| `startWith(value)` | Inserisce un valore iniziale prima delle emissioni della sorgente. | È utile per avviare `combineLatest`, ma il valore iniziale deve avere significato nel dominio. |

Gli errori delle sorgenti si propagano di norma al risultato e interrompono le altre subscription. Scegliere `forkJoin` per operazioni finite di cui serve il risultato finale, `combineLatest` per stato aggiornato nel tempo e `withLatestFrom` quando esiste un evento principale che decide quando campionare gli altri valori.

## Gestione degli errori

`error` e `complete` sono notifiche terminali alternative; un errore non gestito chiude lo stream. `catchError` sostituisce la sorgente fallita con un nuovo Observable o rilancia l'errore; posto **dentro** un flattening può recuperare un singolo inner lasciando vivo l'outer, posto **dopo** intercetta l'errore dell'intera catena. `retry(n)` consente fino a `n` nuove sottoscrizioni dopo un errore; senza limite può riprovare indefinitamente. Può ripetere side effect e non rende sicura un'operazione non idempotente.

```ts
import { throwError, of, catchError, finalize } from "rxjs";
throwError(() => new Error("offline")).pipe(
  catchError(() => of("fallback")),
  finalize(() => console.log("cleanup"))
).subscribe({ next: console.log, error: console.error });
```

`finalize` esegue il callback quando la subscription termina per completion, errore o `unsubscribe`; non trasforma né nasconde l'errore. Definire fallback solo se è un valore valido per il chiamante, altrimenti propagare l'errore.

## Multicasting e Subject

Un **Subject** è insieme Observable e Observer: si invia un valore con `next()` e lo si distribuisce ai subscriber correnti. È hot e multicast, ma espone un punto di scrittura che va controllato; spesso conviene esporre solo `asObservable()` ai consumer. Un Subject non avvia né conserva automaticamente una subscription a un'altra sorgente.

| Tipo | Cosa riceve un nuovo subscriber | Trade-off |
| --- | --- | --- |
| `Subject` | Solo valori futuri. | Nessun valore iniziale o replay. |
| `BehaviorSubject` | Il valore corrente, a partire da quello iniziale richiesto dal costruttore. | Serve un valore iniziale significativo; dopo completion non ripete l'ultimo valore ai nuovi subscriber. |
| `ReplaySubject` | Gli ultimi valori conservati nel buffer, anche dopo completion. | Limitare `bufferSize`/finestra temporale per non trattenere dati senza limite. |
| `AsyncSubject` | L'ultimo valore **solo alla completion**; anche i subscriber successivi possono riceverlo. | Se non completa non emette; senza valori completa senza `next`. |

`share()` crea una subscription condivisa alla sorgente mentre ci sono consumer, senza replay dei valori passati. `shareReplay({ bufferSize: 1, refCount: true })` condivide e ripropone l'ultimo valore ai nuovi subscriber; con `refCount: true` si disiscrive dalla sorgente quando il conteggio scende a zero. Il default di `shareReplay` è `refCount: false`: su una sorgente infinita può continuare a lavorare senza consumer. Una sorgente completata con successo resta in cache, quindi valutare invalidazione, durata del dato e identità dei parametri prima di usarlo come cache.

## Unsubscription e teardown

Il **teardown** è la funzione o risorsa che una sorgente registra per rilasciare timer, listener e richieste quando la subscription termina. Completion ed errore notificano l'Observer e chiudono la subscription; `unsubscribe()` esegue il teardown **senza** chiamare il callback `complete`. Disiscriversi da un wrapper non garantisce cancellazione esterna se il produttore non ha previsto il cleanup.

Le **nested subscriptions** creano ownership e gestione degli errori difficili: preferire `pipe` con un flattening operator, che collega la vita degli inner alla catena. Un Observable finito completa da sé, ma una subscription a stream di eventi o intervalli rimasta attiva dopo la fine del consumer può trattenere callback e risorse: è un **memory leak**. Con operatori che aprono inner, posizionare il cleanup in modo che raggiunga anche la subscription interna; `takeUntil` prima di `switchMap` può lasciare vivo l'inner. Le API Angular `async` pipe e `takeUntilDestroyed` appartengono al [capitolo di integrazione](../../guides/angular/06-rxjs-http-routing.md#integrazione-con-rxjs); `toSignal`/`toObservable` alla [guida ai signals](../../guides/angular/05-signals-change-detection.md#interoperabilità-con-rxjs).

Per un singolo valore asincrono senza composizione di stream, una Promise o uno stato locale semplice può essere più leggibile. RxJS è utile quando servono più valori, cancellazione tramite teardown, combinazioni temporali o più subscriber; la Subscription non è né il valore né la Promise del risultato.

## Scheduler e test

Uno **scheduler** decide quando eseguire lavoro o consegnare notifiche (`subscribeOn` influenza la subscription, `observeOn` le notifiche); non crea automaticamente un thread parallelo. Operatori temporali usano uno scheduler, di solito quello asincrono, e l'ordine osservato dipende anche dalla sorgente.

I **marble diagrams** rappresentano emissioni e completion/errore lungo una linea temporale (`-` tempo, lettere valori, `|` completion, `#` errore). `TestScheduler` virtualizza il tempo degli operatori RxJS per test deterministici; Promise e altri meccanismi asincroni esterni non vengono virtualizzati automaticamente. Usare questi test quando il comportamento temporale è parte del contratto, non per ogni semplice trasformazione.
