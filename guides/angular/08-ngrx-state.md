# NgRx e stato Angular

**NgRx** è una famiglia di librerie per gestire lo stato nelle applicazioni Angular. `@ngrx/store` applica un flusso di eventi e transizioni ispirato a Redux; `@ngrx/signals` offre store basati su Angular Signals. La scelta dipende da chi deve condividere lo stato, dalla durata e dalla necessità di tracciare le transizioni.

## Indice

- [Store action reducer e selector](#store-action-reducer-e-selector)
- [Effects e side effects](#effects-e-side-effects)
- [Entity e normalizzazione](#entity-e-normalizzazione)
- [ComponentStore e SignalStore](#componentstore-e-signalstore)
- [Scelta e confini dello stato](#scelta-e-confini-dello-stato)

## Store action reducer e selector

Lo **stato locale** appartiene a un componente o a una feature e termina con il relativo scope; lo **stato globale** è condiviso tra parti distanti dell'app. Evitare di copiare gli stessi dati in entrambi: decidere una sorgente autorevole e derivare le viste. Nel modello Redux di NgRx Store, i componenti leggono tramite selector e inviano **action**; i **reducer** calcolano il nuovo stato. Il flusso unidirezionale rende esplicito perché cambia un valore.

| Elemento | Contratto e responsabilità |
| --- | --- |
| `Store` | Contenitore di stato iniettabile; `dispatch(action)` invia un evento, `select(selector)` restituisce un `Observable`, `selectSignal(selector)` un `Signal`. |
| Action | Oggetto con `type` che identifica l'evento e, se serve, payload tipizzato; `createAction`/`props` creano l'action creator. Descrive *cosa è accaduto*, non esegue lavoro. |
| Reducer | Funzione pura `(state, action) => nextState`, senza richieste o mutazioni in place; `createReducer` e `on` associano action e transizioni. |
| Selector | Funzione pura che legge o deriva stato; `createSelector` compone input e memoizza il risultato finché restano uguali (per gli oggetti, per reference). |

```ts
import { createAction, createFeature, createReducer, createSelector, on, props } from '@ngrx/store';

export const add = createAction('[Cart] Add', props<{ amount: number }>());
const reducer = createReducer({ count: 0 },
  on(add, (state, { amount }) => ({ ...state, count: state.count + amount })));
export const cartFeature = createFeature({ name: 'cart', reducer });
export const selectIsEmpty = createSelector(cartFeature.selectCount, count => count === 0);
```

`createFeature` raggruppa reducer, nome della slice e selector generati. Registrare lo Store con `provideStore()` e la feature con `provideState(cartFeature)` nei provider dell'app o della route; un componente può fare `store.dispatch(add({ amount: 1 }))` e leggere `store.selectSignal(selectIsEmpty)`. Gli equivalenti `StoreModule`/`EffectsModule` restano utili nei progetti basati su NgModule. Una transizione deve creare nuove reference per i rami cambiati: mutare un array o oggetto esistente può lasciare selector e viste con valori obsoleti. Per [immutabilità e Signals](05-signals-change-detection.md#signal-computed-ed-effect) vedi il capitolo dedicato.

## Effects e side effects

Un **Effect** di `@ngrx/effects` ascolta lo stream `Actions` dopo i reducer, filtra con `ofType` e coordina effetti esterni, come richieste HTTP (Hypertext Transfer Protocol). `createEffect` restituisce un effect registrabile; di norma lo stream emette action di successo o fallimento, poi elaborate dai reducer. Gli effect possono essere classi o funzioni (`{ functional: true }`); `provideEffects(...)` li registra a livello root o di feature. Con `{ dispatch: false }` un effect esegue un'operazione senza emettere action.

Per un caricamento, distinguere `load`, `loadSuccess` e `loadFailure` nello stato (`loading`, dati, errore). Gestire `catchError` **dentro** `switchMap`/`exhaustMap`/`concatMap`/`mergeMap`: un errore lasciato uscire dallo stream esterno può interrompere l'ascolto delle action. L'operatore determina la concorrenza: `switchMap` annulla la richiesta precedente, `exhaustMap` ignora nuovi trigger mentre lavora, `concatMap` accoda, `mergeMap` consente sovrapposizione. Per la [semantica RxJS (Reactive Extensions for JavaScript) e HTTP](06-rxjs-http-routing.md#gestione-degli-errori-http) vedi i capitoli dedicati; scegliere in base a cancellabilità e ordine richiesto dall'operazione.

`provideStoreDevtools()` collega lo Store alla Redux DevTools Extension per ispezionare action, stato e sequenza delle transizioni; configurare limiti e uso in produzione, evitando dati sensibili nei payload. Il replay aiuta a diagnosticare transizioni pure, ma non ripete automaticamente gli effetti esterni come una richiesta al server. I runtime checks di NgRx possono segnalare mutazioni e valori non serializzabili durante lo sviluppo; log e DevTools non sostituiscono gestione esplicita di errori e stati asincroni.

## Entity e normalizzazione

**Normalizzare** una collezione significa conservarla per ID (identificatore), invece di duplicare oggetti in più rami dello stato. `@ngrx/entity` fornisce `EntityState<T>` con `ids` e `entities`, più `createEntityAdapter<T>()` per operazioni immutabili (`addOne`, `updateOne`, `removeOne`, `setAll`) e selector (`selectAll`, `selectEntities`, `selectTotal`). L'adapter assume `id` come chiave se non si specifica `selectId`; `sortComparer` è opzionale e mantiene ordinati gli ID, con costo sugli aggiornamenti.

Nella feature conviene conservare `selectedId` e derivare l'entità selezionata da `entities`, evitando copie incoerenti. La normalizzazione facilita aggiornamenti e lookup di raccolte condivise, ma aggiunge struttura inutile per piccole liste temporanee. In Signal Store l'equivalente è `withEntities` da `@ngrx/signals/entities`, con helper propri: non usare l'adapter di `@ngrx/entity` come se avesse la stessa interfaccia.

## ComponentStore e SignalStore

`ComponentStore` (`@ngrx/component-store`) gestisce stato di una feature o di un componente con `setState`/`patchState`, `updater` puri, `select` Observable, `selectSignal` e `effect` basati su RxJS. Lo si fornisce nello scope che deve possedere lo stato; gli effect del ComponentStore vivono con la sua istanza. È utile in codice esistente costruito su Observable, senza richiedere action, reducer e selector globali.

**NgRx SignalStore** (`@ngrx/signals`, qui secondo la documentazione v22) crea un service iniettabile con `signalStore`. `withState` espone signal di stato (anche annidati), `withComputed` aggiunge stato derivato e `withMethods` racchiude gli aggiornamenti. `patchState` applica un aggiornamento parziale immutabile; per un oggetto annidato va costruito il nuovo oggetto. Lo stato è protetto da scritture esterne per default, così i consumer usano i metodi pubblici.

```ts
import { computed } from '@angular/core';
import { patchState, signalStore, withComputed, withMethods, withState } from '@ngrx/signals';

export const CounterStore = signalStore(
  withState({ count: 0 }),
  withComputed(({ count }) => ({ doubled: computed(() => count() * 2) })),
  withMethods(store => ({
    increment(): void { patchState(store, { count: store.count() + 1 }); },
  })),
);
```

`CounterStore` va aggiunto, per esempio, ai `providers` del componente e iniettato con `inject(CounterStore)`; `doubled()` segue `count()` senza una seconda copia dello stato. `{ providedIn: 'root' }` in `signalStore` crea invece un'istanza condivisa. `withHooks` collega `onInit`/`onDestroy` al lifecycle dell'istanza; `rxMethod` da `@ngrx/signals/rxjs-interop` collega input reattivi a pipeline RxJS per lavoro asincrono. Anche qui modellare loading, errore e cleanup: `patchState` non gestisce da solo il fallimento di una richiesta. `ComponentStore.effect` e `rxMethod` lavorano nell'istanza dello store, distinti dagli Effects che ascoltano le action globali.

## Scelta e confini dello stato

| Opzione | Adatta a | Costo o limite |
| --- | --- | --- |
| Service con `signal`/`computed` o RxJS | Pochi valori locali o condivisi con logica semplice. | Convenzioni, errori e lifecycle sono responsabilità dell'app. |
| `ComponentStore` | Feature locale già orientata a Observable e updater. | API e struttura aggiuntive; non offre il flusso di action dello Store globale. |
| NgRx SignalStore | Stato di feature o condiviso, con signal derivati e metodi incapsulati. | Richiede disciplina per side effects e tracing; lo scope dipende dai provider. |
| NgRx Store + Effects | Stato condiviso con molte sorgenti di eventi, transizioni auditabili e workflow complessi. | Action, reducer, selector ed effect aumentano i concetti e il coordinamento. |

Lo Store globale diventa eccessivo per un toggle o un form temporaneo; una feature con più viste, cache condivisa e transizioni da fonti diverse può giustificarlo. Non equiparare la portata allo strumento: un SignalStore può essere root o locale, mentre un service Angular può condividere stato fra route. I dati remoti richiedono comunque una politica di caricamento, invalidazione e concorrenza. Per la distinzione tra [Signals, Observable e change detection](05-signals-change-detection.md#interoperabilità-con-rxjs) vedi il capitolo precedente.

Nei test, chiamare reducer e selector puri con input noti; per gli Effects fornire `Actions` controllate (`provideMockActions`) e un service falso, verificando action di successo, errore e concorrenza. `provideMockStore` isola i consumer dello Store; testare separatamente metodi e computed di SignalStore, inclusi loading e failure. Per setup e test dei flussi asincroni vedi [Testing Angular](09-testing.md#test-dei-flussi-reattivi).
