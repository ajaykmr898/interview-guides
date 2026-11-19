# Service DI e lifecycle Angular

La **dependency injection (DI)** di Angular fornisce a componenti, direttive e service le dipendenze richieste tramite una gerarchia di injector. Il **lifecycle** dei componenti scandisce creazione, controlli, rendering e distruzione; scegliere lo scope dei provider e il punto di cleanup evita stato condiviso per errore e risorse trattenute oltre il necessario.

## Indice

- [Service e provider](#service-e-provider)
- [Gerarchia degli injector](#gerarchia-degli-injector)
- [Injection context e inject](#injection-context-e-inject)
- [Lifecycle dei componenti](#lifecycle-dei-componenti)
- [Cleanup e DestroyRef](#cleanup-e-destroyref)

## Service e provider

Un **service** incapsula logica o accesso ai dati indipendenti dal template; il componente coordina la vista e l'interazione. Con l'**inversion of control (IoC)** Angular crea o recupera le dipendenze tramite un **provider**, invece di farle istanziare direttamente al consumer. La dependency inversion è utile quando il consumer dipende da un token e può ricevere implementazioni diverse, per esempio nei test. `@Injectable` fornisce i metadata necessari alla DI; `providedIn: 'root'` registra il service nell'injector applicativo, mentre `@Injectable()` da solo richiede un provider esplicito.

```ts
import { Injectable, InjectionToken, inject } from '@angular/core';

export const TAX_RATE = new InjectionToken<number>('TAX_RATE');

@Injectable({ providedIn: 'root' })
export class PriceService {
  private readonly rate = inject(TAX_RATE);
  total(net: number): number { return net * (1 + this.rate); }
}
```

`InjectionToken<T>` crea un identificatore runtime univoco, tipizzato in TypeScript, per valori non rappresentati da una classe; una `interface` TypeScript scompare a runtime e non può essere un token. Nell'esempio `PriceService` è root, ma `TAX_RATE` deve essere registrato prima dell'iniezione: senza provider Angular segnala un errore di risoluzione.

```ts
// src/app/app.config.ts
import { ApplicationConfig } from '@angular/core';
import { TAX_RATE } from './price.service';

export const appConfig: ApplicationConfig = {
  providers: [{ provide: TAX_RATE, useValue: 0.22 }],
};
```

| Provider | Risultato | Quando usarlo |
| --- | --- | --- |
| `{ provide: Token, useClass: Impl }` | Crea un'istanza di `Impl` per quell'injector quando richiesta; `providers: [Impl]` è la forma breve per la propria classe. | Sostituire un'implementazione mantenendo il token. |
| `{ provide: Token, useValue: value }` | Restituisce il valore fornito, senza costruire una classe. | Configurazione o fake nei test. |
| `{ provide: Token, useFactory: fn }` | Chiama la factory nel contesto DI; usa il valore restituito, anche con dipendenze ottenute tramite `inject()`. | Costruzione che dipende da configurazione o altri token. |
| `{ provide: Alias, useExisting: Token }` | Alias dello stesso valore/istanza già fornito per `Token`; non crea una seconda istanza. | Compatibilità tra token o contratti. |

`useClass` e `useExisting` non sono equivalenti: due provider `useClass` separati possono creare istanze distinte. Un provider registra una ricetta, non implica che il service venga creato subito. Per l'organizzazione [component vs service](01-core-architecture.md#struttura-dellapplicazione) vedi il capitolo architetturale.

## Gerarchia degli injector

Angular risolve un token partendo dal contesto che lo richiede: cerca prima nell'**ElementInjector** dell'elemento e degli antenati, poi nella gerarchia degli **EnvironmentInjector** (route e applicazione). Il provider più vicino prevale; se nessuno risolve il token, l'iniezione obbligatoria fallisce. Nelle applicazioni basate su NgModule interviene anche la gerarchia `ModuleInjector`.

| Registrazione | Visibilità e durata tipiche | Scelta |
| --- | --- | --- |
| `@Injectable({ providedIn: 'root' })` o provider in `ApplicationConfig` | Un'istanza per l'injector root dell'app, condivisa dove non è sovrascritta. | Servizi trasversali; non equivale a un singleton globale tra app diverse. |
| `@Component({ providers: [S] })` | Nuova istanza per ciascun host del componente, accessibile anche ai discendenti secondo le regole di visibilità; distrutta con quel componente. | Stato isolato per una porzione di interfaccia. |
| `Route.providers` | Istanza nell'`EnvironmentInjector` della route, visibile a route e figli. | Configurazione o service della feature, senza esporli al resto dell'app. |

Un provider locale dello stesso token prevale su quello root. `viewProviders` limita ulteriormente la visibilità alla vista del componente, escludendo il contenuto proiettato. Lo scope di un service di route segue la vita dell'injector della route: non assumere che venga distrutto a ogni navigazione. Registrare in root solo ciò che deve essere condiviso; una nuova registrazione in ogni componente cambia identità e stato del service, anche se la classe è la stessa. Per i [confini lazy delle route](06-rxjs-http-routing.md#router-e-lazy-loading) vedi Router.

I modificatori della risoluzione servono quando la ricerca standard non esprime il contratto voluto: `inject(Token, { optional: true })` restituisce `null` se manca il provider, invece di fallire; `{ self: true }` limita la ricerca all'injector corrente; `{ skipSelf: true }` parte dal parent, utile per leggere un provider sovrascritto. Evitare `optional` come rimedio generico a provider mancanti: gestire esplicitamente il caso `null`.

## Injection context e inject

L'**injection context** è il tratto sincrono di esecuzione in cui Angular conosce l'injector corrente. `inject(Token)` restituisce la dipendenza risolta, oppure lancia se il token manca; è valido nei field initializer, nel costruttore e nelle factory dei provider. Fuori contesto lancia `NG0203`: non chiamarlo in `ngOnInit`, nei listener o dopo un `await`. `runInInjectionContext(injector, fn)` crea un contesto per la sola esecuzione sincrona di `fn`, non lo prolunga attraverso operazioni asincrone.

| Forma | Caratteristica | Limite |
| --- | --- | --- |
| Constructor injection | Le dipendenze sono parametri del costruttore; per un token non-classe si usa `@Inject(TOKEN)`. | Allunga la firma se le dipendenze sono molte. |
| `inject()` | Restituisce il valore tipizzato dal token nel punto di inizializzazione; supporta opzioni come `optional`, `self`, `skipSelf`. | Richiede injection context; non è un service locator da invocare in qualunque metodo. |

Preferire l'iniezione diretta in un initializer quando la dipendenza è stabile per tutta l'istanza; passare poi la dipendenza ai metodi o callback che ne hanno bisogno. La [semantica dei decorator](03-directives-pipes-decorators.md#decoratori-angular) è trattata nel capitolo dedicato.

## Lifecycle dei componenti

Gli **hook di lifecycle** sono metodi invocati da Angular durante la vita di un componente. Il costruttore crea l'istanza ma non indica che input, content o view siano già pronti. Durante l'inizializzazione, il primo `ngOnChanges` (se ci sono input modificati) precede `ngOnInit`; gli hook `Checked` possono eseguire molte volte e non sono posti adatti a lavoro costoso.

| Hook | Quando viene chiamato | Uso e limite |
| --- | --- | --- |
| `ngOnChanges(changes)` | A ogni cambiamento degli input, anche iniziale; prima del controllo del template del componente. | Leggere `SimpleChanges` (`previousValue`, `currentValue`, `firstChange`); le chiavi sono i nomi delle proprietà, non gli alias. |
| `ngOnInit()` | Una volta, dopo l'inizializzazione degli input e prima del template proprio. | Inizializzare stato dipendente dagli input. |
| `ngDoCheck()` | Prima di ogni controllo del template. | Rilevamento manuale solo se indispensabile; costo potenzialmente alto. |
| `ngAfterContentInit()` | Una volta, dopo l'inizializzazione del contenuto proiettato. | Leggere content query già inizializzate. |
| `ngAfterContentChecked()` | A ogni controllo del contenuto proiettato. | Osservare risultati aggiornati; evitare lavoro ripetuto. |
| `ngAfterViewInit()` | Una volta, dopo l'inizializzazione della vista del componente. | Leggere view query già inizializzate. |
| `ngAfterViewChecked()` | A ogni controllo della vista. | Osservare risultati aggiornati; evitare lavoro ripetuto. |
| `ngOnDestroy()` | Una volta, prima della distruzione dell'istanza. | Rimuovere risorse possedute dal componente. |

Cambiare stato già controllato negli hook `AfterContent*`/`AfterView*` può produrre `ExpressionChangedAfterItHasBeenCheckedError`. Per la distinzione fra [view e content query](02-components-templates-syntax.md#content-projection) e per i trigger della [change detection](05-signals-change-detection.md#change-detection) vedi i capitoli dedicati.

`afterNextRender(callback)` registra un callback dopo il prossimo rendering dell'intera applicazione; `afterEveryRender(callback)` dopo ogni rendering. Sono funzioni, non metodi della classe: si registrano in injection context, restituiscono un `AfterRenderRef` annullabile con `destroy()`, e non vengono eseguite durante server-side rendering o prerendering. Usarle per accesso al Document Object Model (DOM) dopo il render, con fasi `write`/`read` se si manipola il layout; non sostituiscono `ngAfterViewInit` per la disponibilità delle view query.

## Cleanup e DestroyRef

`DestroyRef` lega una funzione di cleanup alla distruzione del componente o della direttiva che lo inietta; in un service segue invece la vita del relativo injector. `onDestroy(callback)` restituisce una funzione per annullare la registrazione; `destroyed` indica se lo scope è già stato distrutto, utile prima di applicare risultati asincroni tardivi.

```ts
import { Component, DestroyRef, inject, signal } from '@angular/core';

@Component({ selector: 'app-clock', template: '{{ now() }}' })
export class Clock {
  readonly now = signal(Date.now());
  constructor() {
    const timer = setInterval(() => this.now.set(Date.now()), 1000);
    inject(DestroyRef).onDestroy(() => clearInterval(timer));
  }
}
```

Il timer si ferma quando `Clock` viene distrutto. Applicare lo stesso principio a listener DOM, observer e sottoscrizioni che il componente crea. Per RxJS, `takeUntilDestroyed()` completa l'Observable e chiude la sottoscrizione quando termina lo scope DI; fuori da un injection context richiede un `DestroyRef` esplicito. Una sottoscrizione in un service root può vivere fino alla distruzione dell'app: scegliere consapevolmente dove crearla. `AsyncPipe` gestisce la propria sottoscrizione nel template; per [Observable e cleanup](06-rxjs-http-routing.md#integrazione-con-rxjs) vedi il capitolo RxJS Angular.
