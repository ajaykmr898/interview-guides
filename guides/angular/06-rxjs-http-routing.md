# RxJS HTTP e routing Angular

Angular usa RxJS (Reactive Extensions for JavaScript) per rappresentare richieste Hypertext Transfer Protocol (HTTP) e stati di navigazione come stream; `HttpClient` gestisce le richieste, mentre il **Router** associa percorsi URL (Uniform Resource Locator) a viste e regole di navigazione. Service, interceptor e route tengono separati accesso ai dati, politiche trasversali e interfaccia.

## Indice

- [HttpClient e interceptor](#httpclient-e-interceptor)
- [Gestione degli errori HTTP](#gestione-degli-errori-http)
- [Router e lazy loading](#router-e-lazy-loading)
- [Guard e resolver](#guard-e-resolver)
- [Parametri e navigazione](#parametri-e-navigazione)
- [Integrazione con RxJS](#integrazione-con-rxjs)

## HttpClient e interceptor

`HttpClient` si inietta in un service e offre metodi come `get<T>(url, options)` e `post<T>(url, body, options)`. Restituiscono `Observable<T>` **cold**: la richiesta parte alla subscription, e due subscription allo stesso Observable generano due richieste. Il generic `T` controlla i tipi in compilazione, ma non valida il payload ricevuto a runtime. Gli Observable di risposta normalmente emettono e completano; un interceptor può modificarne il comportamento.

```ts
import { HttpClient, HttpParams } from '@angular/common/http';
import { Injectable, inject } from '@angular/core';
import { Observable } from 'rxjs';

type User = { id: number; name: string };
@Injectable({ providedIn: 'root' })
export class UsersApi {
  private readonly http = inject(HttpClient);
  list(page: number): Observable<User[]> {
    const params = new HttpParams().set('page', page);
    return this.http.get<User[]>('/api/users', { params });
  }
}
```

`HttpParams` e `HttpHeaders` sono immutabili: `set`/`append` restituiscono nuove istanze, quindi conservarne il risultato. Per casi semplici `params: { page }` e `headers: { Accept: 'application/json' }` evitano oggetti dedicati. `observe: 'response'` restituisce `HttpResponse<T>` per leggere status e header; `observe: 'events'` restituisce uno stream di `HttpEvent<T>`. Gli eventi di upload richiedono `withXhr()`, perché il backend `fetch` corrente non li supporta; evitare XHR nel server-side rendering. Dalla v21 `HttpClient` è disponibile per l'iniezione per default; `provideHttpClient(...)` resta il punto per configurare feature e interceptor. `HttpClientModule` è l'alternativa legacy ora deprecata.

Un **functional interceptor** riceve `HttpRequest` e `HttpHandlerFn`, e restituisce un `Observable<HttpEvent<unknown>>`. È middleware per autenticazione, logging, retry o cache condivisi; la richiesta è in gran parte immutabile, quindi si modifica con `clone()`. `withInterceptors([fn])` li registra in ordine esplicito; l'injection context consente `inject()` nella funzione. Gli interceptor a classe restano supportati tramite `withInterceptorsFromDi()`, ma l'ordine può essere meno prevedibile.

```ts
import { inject } from '@angular/core';
import { HttpInterceptorFn } from '@angular/common/http';
import { AuthState } from './auth-state';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = inject(AuthState).accessToken();
  const ownApi = req.url.startsWith('/api/');
  return next(token && ownApi
    ? req.clone({ setHeaders: { Authorization: `Bearer ${token}` } })
    : req);
};
```

`AuthState` è un service applicativo che fornisce il token corrente; si registra l'interceptor con `provideHttpClient(withInterceptors([authInterceptor]))`. Limitare l'header agli endpoint fidati, senza esporre token nei log. Con sessioni basate su cookie servono anche le corrette protezioni contro richieste cross-site; un header Bearer non sostituisce l'autorizzazione sul server.

## Gestione degli errori HTTP

`HttpClient` invia gli errori nel canale `error` come `HttpErrorResponse`: `status === 0` può indicare errore di rete o timeout, mentre uno status HTTP non riuscito porta il codice restituito dal server. `catchError` converte l'errore in uno stato o fallback solo se il chiamante può usarlo; un interceptor può centralizzare logging, normalizzazione e policy comuni, lasciando al componente la presentazione di errori attesi. Evitare di trasformare silenziosamente un errore in una lista vuota se ciò nasconde un guasto.

Per un caricamento mostrare **loading**, **success/empty** ed **error** separati; `finalize` rilascia uno stato di caricamento anche su errore o unsubscription. Una subscription annullata interrompe la richiesta in corso, utile con `AsyncPipe` e `switchMap` per ricerche in cui conta l'ultimo input. `retry` si risottoscrive e può ripetere una scrittura: applicarlo con limite e criterio solo dove la richiesta è idempotente o il server gestisce duplicati. Per una cache condivisa, riusare davvero lo stesso Observable per chiave e definire invalidazione dopo mutazioni; `shareReplay` senza una strategia di durata può servire dati obsoleti. La [semantica di operatori, cache e teardown](../../topics/rxjs/10-rxjs.md#unsubscription-e-teardown) è nel capitolo RxJS.

Un interceptor che gestisce `401` può avviare **un solo refresh** per richieste concorrenti e ritentare la richiesta originale una volta con il nuovo token. Escludere login e refresh dall'intercettazione ricorsiva e chiudere la sessione se il refresh fallisce. Questa è una politica client di sessione: il backend verifica sempre identità e permessi su ogni richiesta.

## Router e lazy loading

`provideRouter(routes)` registra il Router; ogni `Route` associa un `path` a un componente, a figli, a un redirect o a un caricamento lazy. `<router-outlet />` è il punto in cui compare la vista attiva. `RouterLink` crea link navigabili nel template; `RouterLinkActive` applica una classe quando il link è attivo e può impostare `ariaCurrentWhenActive="page"` per l'accessibilità. Nei componenti standalone importare le direttive usate dal template.

```ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  { path: '', pathMatch: 'full', redirectTo: 'catalog' },
  { path: 'catalog', loadChildren: () => import('./catalog/routes').then(m => m.catalogRoutes) },
  { path: 'item/:id', loadComponent: () => import('./item/page').then(m => m.ItemPage) },
  { path: '**', loadComponent: () => import('./not-found').then(m => m.NotFound) },
];
```

`component` importa la pagina in modo eager; `loadComponent` carica un componente e `loadChildren` carica un insieme di route con `import()` al primo accesso. Il caricamento lazy riduce il bundle iniziale se la feature non è anche importata da codice eager, ma introduce latenza alla prima visita. Il redirect dalla stringa vuota usa `pathMatch: 'full'` per non catturare ogni URL; il wildcard `**` va dopo le route specifiche. Le route figlie in `children` rendono dentro un `router-outlet` del genitore.

Il **preloading** anticipa il caricamento di codice lazy dopo la navigazione iniziale: `withPreloading(PreloadAllModules)` riduce l'attesa alle visite successive ma consuma rete e memoria; il default è `NoPreloading`. Non equivale a precaricare i dati restituiti da un resolver. Per [confini dei chunk](01-core-architecture.md#dependency-graph) vedi il capitolo architetturale.

## Guard e resolver

Le **guard** decidono se una navigazione può corrispondere, entrare o uscire da una route. Le funzioni moderne `CanMatchFn`, `CanActivateFn` e `CanDeactivateFn<T>` usano l'injection context della route; le guard a classe registrate come token sono un approccio legacy deprecato. Possono restituire `boolean`, `UrlTree`/`RedirectCommand` oppure una Promise/Observable di tali valori: per un Observable il Router usa la prima emissione e si disiscrive. Per un redirect restituire un `UrlTree`, senza chiamare `navigate()` dentro la guard.

| Guardia | Decisione e limite |
| --- | --- |
| `CanMatch` | Decide se la route corrisponde; `false` fa provare le route successive, utile per feature flag. |
| `CanActivate` | Decide l'ingresso nella route selezionata; `false` blocca la navigazione. |
| `CanDeactivate` | Decide se lasciare il componente corrente, per esempio con modifiche non salvate. |

Una **guard client non è un confine di sicurezza**: codice e URL nel browser sono controllabili dall'utente; autorizzazione e dati protetti vanno verificati dal server. Per più guard asincrone non assumere esecuzione seriale: se dipendono dallo stesso stato, coordinarlo in un service o una sola guard.

Un **resolver** `ResolveFn<T>` ottiene dati prima dell'attivazione e li espone in `ActivatedRoute.data` con la chiave dichiarata in `resolve`. Può restituire un valore, Promise o Observable; se è lento blocca la navigazione, se fallisce produce un errore di navigazione da gestire, per esempio con `withNavigationErrorHandler`. Usarlo per dati indispensabili prima del rendering; per contenuto non critico può essere migliore un caricamento nella pagina con stato visibile. `Route.providers` dà a guard, resolver e componenti della route dipendenze della feature; la durata segue l'injector di route, non necessariamente ogni singola visita ([scope dei provider](04-services-di-lifecycle.md#gerarchia-degli-injector)).

## Parametri e navigazione

Un **path parameter** come `:id` in `item/:id` identifica una risorsa e partecipa al matching; un **query parameter** come `?sort=name` configura filtro o ordinamento senza cambiare la struttura della route. `ActivatedRoute.snapshot.paramMap.get('id')` legge una fotografia e può restituire `null`; `paramMap`, `queryParamMap` e `data` sono Observable da usare se lo stesso componente resta attivo mentre cambiano parametri o dati. `data` contiene valori statici della route e risultati dei resolver.

`Router.navigate(['/item', id], { queryParams: { tab: 'details' } })` costruisce la destinazione da segmenti; `navigateByUrl('/item/42')` parte da una URL completa. Entrambi restituiscono `Promise<boolean>`: `true` segnala successo, `false` una navigazione non riuscita; gli errori possono rifiutare la Promise. `relativeTo` permette navigazione relativa a `ActivatedRoute`; `queryParamsHandling: 'merge'` conserva altri parametri, ma valutare se i vecchi filtri vadano davvero mantenuti.

## Integrazione con RxJS

`AsyncPipe` sottoscrive un Observable nel template, espone l'ultimo valore e si disiscrive quando la vista termina o la sorgente cambia. `takeUntilDestroyed()` completa lo stream e chiude la subscription quando lo scope di `DestroyRef` viene distrutto; fuori da un injection context passargli esplicitamente un `DestroyRef`. È utile per subscription imperative a stream lunghi, mentre una richiesta `HttpClient` singola normalmente completa da sé ma può comunque rispondere dopo la distruzione del consumer.

`toSignal(observable, { initialValue })` crea un Signal dal valore più recente e una subscription immediata con cleanup legato allo scope; un errore viene rilanciato alla lettura. `toObservable(signal)` produce uno stream dai cambiamenti del Signal, con emissioni successive asincrone. Per timing, `requireSync` e altre distinzioni vedi [Signals e interoperabilità](05-signals-change-detection.md#interoperabilità-con-rxjs).

```ts
import { Component, inject } from '@angular/core';
import { toSignal } from '@angular/core/rxjs-interop';
import { ActivatedRoute } from '@angular/router';
import { map } from 'rxjs';

@Component({ template: '{{ id() }}' })
export class ItemPage {
  private readonly route = inject(ActivatedRoute);
  readonly id = toSignal(this.route.paramMap.pipe(map(p => p.get('id'))), { initialValue: null });
}
```

`id` segue anche i cambi di parametro senza una subscription manuale; `null` rappresenta l'assenza dell'identificatore. `rxResource` collega parametri reattivi a uno stream Observable e ne espone valore, loading ed errore come Signal; lo stream deve emettere almeno un valore o errore prima di completare. `httpResource` offre lo stesso modello per `HttpClient` con interceptor. Entrambe sono stabili dalla v22; per [resource e stato asincrono](05-signals-change-detection.md#stato-asincrono-con-resource) vedi Signals.

Nei test di integrazione, `provideHttpClientTesting()` con `HttpTestingController` verifica richieste, risposte ed errori senza rete; dichiararlo dopo `provideHttpClient()`. `provideRouter(routes)` e `RouterTestingHarness` verificano URL, guard e componenti renderizzati. Gli esempi completi appartengono al capitolo [testing Angular](09-testing.md#test-http-e-routing).
