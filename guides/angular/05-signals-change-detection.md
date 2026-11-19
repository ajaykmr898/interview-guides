# Signals e change detection Angular

Un **Signal** espone un valore corrente leggibile sincronicamente e notifica i consumer reattivi che lo hanno letto quando cambia. La **change detection** aggiorna i binding del template nelle viste che Angular decide di controllare; Signals e strategia del componente determinano quali aggiornamenti richiedono un nuovo controllo.

## Indice

- [Signal computed ed effect](#signal-computed-ed-effect)
- [Input e output reattivi](#input-e-output-reattivi)
- [Change detection](#change-detection)
- [OnPush e zone](#onpush-e-zone)
- [Interoperabilità con RxJS](#interoperabilità-con-rxjs)

## Signal computed ed effect

| Funzione | Comportamento | Uso |
| --- | --- | --- |
| `signal(initial)` | Restituisce un `WritableSignal<T>`: `s()` legge, `set(value)` sostituisce, `update(fn)` calcola il nuovo valore dal precedente. | Stato locale modificabile. |
| `computed(fn)` | Restituisce un `Signal<T>` read-only derivato: calcolo lazy e memoizzato, rieseguito alla lettura dopo l'invalidazione. | Stato derivato senza copie da sincronizzare. |
| `effect(fn)` | Restituisce un `EffectRef`; esegue un side effect almeno una volta e poi quando cambiano le dipendenze lette nell'ultima esecuzione. | Sincronizzare stato con sistemi imperativi. |

```ts
import { computed, signal } from '@angular/core';

const count = signal(1);
const double = computed(() => count() * 2);
count.set(2);
count.update(n => n + 1);
console.log(double()); // 6
```

Le dipendenze di `computed` sono **dinamiche**: un signal letto solo in un ramo attivo viene tracciato solo mentre quel ramo è eseguito. Il valore è confrontato per default con `Object.is`; mutare un oggetto o array in place conservando la stessa reference non notifica i consumer. Preferire aggiornamenti immutabili. `asReadonly()` limita la scrittura tramite i metodi del signal, ma non congela profondamente l'oggetto contenuto. Una funzione `computed` deve restare pura: non scrivere altri signal mentre calcola.

`effect()` richiede un [injection context](04-services-di-lifecycle.md#injection-context-e-inject), salvo `{ injector }` esplicito. Traccia le letture sincrone: leggere un signal solo dentro un timer non crea una dipendenza. L'esecuzione è schedulata, non immediata; gli effect di componente seguono la sincronizzazione della vista, quelli root non appartengono al tree dei componenti. `onCleanup` libera risorse prima della riesecuzione e alla distruzione; Angular distrugge automaticamente l'effect con il suo scope, oppure si può chiamare `EffectRef.destroy()`.

```ts
import { Component, effect, signal } from '@angular/core';

@Component({ template: '' })
export class Search {
  readonly query = signal('');
  constructor() {
    effect((onCleanup) => {
      const value = this.query();
      const timer = setTimeout(() => console.log(value), 300);
      onCleanup(() => clearTimeout(timer));
    });
  }
}
```

L'esempio cancella il timer precedente quando cambia `query`. Usare `computed` per derivazioni e `linkedSignal` per stato derivato ma modificabile; propagare valori fra signal tramite `effect` può introdurre cicli e controlli superflui.

## Input e output reattivi

`input<T>()` dichiara un `InputSignal<T | undefined>` read-only; `input.required<T>()` produce `InputSignal<T>` e impone il binding al consumer. Il valore arriva dal genitore durante l'inizializzazione, quindi un required input non va letto prima che Angular lo imposti; è invece sicuro usarlo in un `computed` valutato dal template dopo il binding. Le query `viewChild()` e `contentChild()` restituiscono signal aggiornati quando cambia la rispettiva vista o il contenuto proiettato: senza risultato valgono `undefined`, mentre `.required()` lancia se letto senza risultato disponibile. Per [input e output](02-components-templates-syntax.md#comunicazione-tra-componenti) e [view/content query](02-components-templates-syntax.md#content-projection) vedi il capitolo componenti.

`model(initial)` restituisce un `ModelSignal<T>` scrivibile; senza valore iniziale, `model<T>()` include anche `undefined`. `set`/`update` cambiano il valore ed emettono l'output `nomeChange`, utile per il two-way binding; `output<T>()` emette eventi ma non è un Signal. `linkedSignal(() => source())` crea un writable signal il cui valore si riallinea quando cambia la sorgente: è adatto a una selezione modificabile che deve tornare valida dopo un cambio di opzioni, mentre `computed` non accetta `set`.

```ts
import { ChangeDetectionStrategy, Component, computed, input, signal } from '@angular/core';

@Component({
  selector: 'app-total',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: '<button (click)="increment()">{{ total() }}</button>',
})
export class Total {
  readonly price = input.required<number>();
  readonly qty = signal(1);
  readonly total = computed(() => this.price() * this.qty());
  increment(): void { this.qty.update(n => n + 1); }
}
```

Un nuovo `[price]` dal genitore o il click che aggiorna `qty` ricalcola `total`; la lettura del signal nel template notifica Angular di aggiornare il componente `OnPush`. L'annotazione esplicita resta utile per codice compatibile con versioni precedenti alla v22.

### Stato asincrono con resource

`resource({ params, loader })` restituisce un `ResourceRef` che espone `value()`, `status()`, `error()` e `isLoading()` come signal: `params` è reattivo, il `loader` restituisce una Promise e riceve `{ params, abortSignal }`; la richiesta si annulla se il loader usa quel segnale quando i parametri cambiano o lo scope termina. Senza `defaultValue`, `value()` è inizialmente `undefined` e lancia se lo stato è `error`: controllare `hasValue()`/`error()` prima di leggerlo. Usarlo per letture asincrone, non per mutazioni che l'annullamento renderebbe ambigue.

`rxResource` usa un `stream` Observable al posto del loader Promise; `httpResource` integra `HttpClient` e i suoi interceptor. Queste funzioni sono stabili dalla v22 (in versioni precedenti avevano uno stato diverso): verificare la versione del progetto. Per [richieste e interoperabilità RxJS](06-rxjs-http-routing.md#integrazione-con-rxjs) vedi il capitolo dedicato.

## Change detection

Angular valuta i binding delle viste controllate, confronta i risultati e aggiorna il Document Object Model (DOM) quando necessario. Un cambiamento di stato non implica che ogni componente venga ricontrollato: contano la strategia della vista e la notifica che avvia il controllo.

| Strategia | Quando controlla la vista | Limite |
| --- | --- | --- |
| `ChangeDetectionStrategy.Eager` | Ogni volta che il traversal di change detection raggiunge la vista. `Default` è l'alias legacy equivalente, deprecato. | Può ricontrollare sottoalberi non interessati. |
| `ChangeDetectionStrategy.OnPush` | Con nuovi input da template, eventi nel sottoalbero, `markForCheck()`/`AsyncPipe` o modifiche a signal letti nel template. | Mutazioni in place con reference invariata non notificano un nuovo input. |

`OnPush` è il default dalla v22; nelle versioni precedenti il default era `Default`/oggi `Eager`. Per gli input `OnPush` Angular confronta il nuovo valore con il precedente tramite `Object.is`; l'aggiornamento di un signal letto nel template rinfresca i componenti che lo leggono. `ChangeDetectorRef.markForCheck()` marca la vista per un prossimo controllo e restituisce `void`, senza render immediato; `detectChanges()` esegue invece un controllo sincrono locale, utile anche per viste temporaneamente `detach()` dal tree. Preferire input, signal e `AsyncPipe` a controlli manuali quando esprimono già la dipendenza.

## OnPush e zone

`Zone.js` intercetta eventi e task asincroni del browser per segnalare ad Angular possibili aggiornamenti, anche quando nessuno stato usato dalla vista è cambiato. Nelle app basate su zone, `NgZone.runOutsideAngular(fn)` evita che lavoro frequente provochi controlli; `run(fn)` rientra nella zone. `NgZone` resta compatibile con app zoneless, ma il solo rientro nella zone non sostituisce una notifica esplicita.

**Zoneless** è il default dalla v21; dalla v20.2 si abilita con `provideZonelessChangeDetection()`, mentre `provideZoneChangeDetection()` configura il comportamento basato su Zone.js. Senza Zone.js, Promise, timer o modifiche a proprietà ordinarie non programmano da soli una verifica. La programmano, tra gli altri, un signal letto nel template che cambia, `markForCheck()`, un input impostato da Angular e un listener host/template. `OnPush` aiuta a rendere esplicite le dipendenze, ma non è requisito tecnico per zoneless. Per [migrazioni di versione](11-evolution-migrations.md#versioni-e-compatibilità) vedi il capitolo evoluzione.

Nel server-side rendering (SSR) e nella hydration, i binding da signal devono produrre una struttura DOM coerente tra server e browser; mutazioni dirette del DOM prima della hydration possono creare mismatch. Per accesso DOM dopo il rendering usare i [render callback](04-services-di-lifecycle.md#lifecycle-dei-componenti), che non girano sul server. Un `resource` può evitare il secondo caricamento al client tramite un `id` stabile e `TransferState`; non serializzare così dati utente se l'HTML può essere condiviso o messo in cache. Il quadro completo è nel capitolo [SSR e hydration](10-performance-security-ssr.md#ssr-e-hydration).

## Interoperabilità con RxJS

`toSignal(observable)` sottoscrive subito e restituisce un `Signal<T | undefined>` finché la sorgente non emette; `initialValue` fornisce un valore iniziale, `requireSync: true` impone un'emissione sincrona oppure lancia. Per default la subscription termina con lo scope di dependency injection (DI); un errore dell'Observable viene rilanciato alla lettura del signal, mentre dopo la completion rimane l'ultimo valore. Riutilizzare il signal risultante anziché creare più subscription alla stessa sorgente.

`toObservable(signal)` restituisce un `Observable<T>` tramite un effect: la prima emissione può essere sincrona, le successive sono asincrone e più `set` prima della stabilizzazione possono produrre solo l'ultimo valore. Di norma entrambe le conversioni richiedono un injection context o un `Injector` esplicito; `toSignal(observable, { manualCleanup: true })` può farne a meno, ma la subscription resta attiva fino alla completion della sorgente. Per la semantica di [Observable e Subscription](../../topics/rxjs/10-rxjs.md#observable-e-subscription) e per l'integrazione con [HttpClient e cleanup](06-rxjs-http-routing.md#integrazione-con-rxjs) vedi i capitoli dedicati.

| Scelta | Adatta a | Trade-off |
| --- | --- | --- |
| Signal / `computed` | Stato locale corrente e derivazioni sincrone nel template. | Non rappresenta da solo stream con completion, errori e cancellazione. |
| Observable RxJS | Eventi asincroni, richieste e composizione di stream. | Richiede gestione di subscription e stati iniziali nel consumer. |
| NgRx Store | Stato globale condiviso con transizioni, selector ed effect espliciti. | Introduce struttura e disciplina aggiuntive; per stato locale può essere eccessivo. |

Per [NgRx e gestione dello stato](08-ngrx-state.md#store-action-reducer-e-selector) vedi il capitolo dedicato.
