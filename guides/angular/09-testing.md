# Testing Angular

Il **testing Angular** verifica comportamento e integrazione di componenti, service, template, dependency injection e Router con l'infrastruttura di test del framework. Scegliere il confine più piccolo che osserva il comportamento richiesto, poi aggiungere test più ampi per le interazioni che quel confine non copre.

## Indice

- [TestBed e fixture](#testbed-e-fixture)
- [Test dei componenti](#test-dei-componenti)
- [Test di service e DI](#test-di-service-e-di)
- [Test HTTP e routing](#test-http-e-routing)
- [Mock e harness](#mock-e-harness)
- [Test dei flussi reattivi](#test-dei-flussi-reattivi)

## TestBed e fixture

| Tipo | Cosa verifica nel contesto Angular |
| --- | --- |
| Unit | Un reducer, validator o service isolato, con dipendenze controllate. |
| Component | Template, input/output, rendering e interazioni di un componente. |
| Integration | Collaborazione reale fra più componenti, provider, Router o backend di test. |
| E2E (End-to-End) | Percorso utente nell'app avviata in un browser, attraverso più livelli. |
| Contract | Compatibilità di richieste e risposte fra client Angular e servizio remoto. |
| Smoke / regression | Smoke verifica pochi flussi essenziali dopo build/deploy; regression controlla che comportamenti esistenti restino corretti, a qualsiasi livello. |

Per scopo e limiti dei [tipi di test](../../topics/engineering/15-architecture-devops.md#qualità-e-tradeoff) vedi il capitolo generale. `TestBed` configura un injector e un ambiente Angular isolato per lo spec: `configureTestingModule({ imports, providers })` aggiunge componenti standalone e dipendenze; `inject(token)` recupera un provider. `createComponent(C)` restituisce una `ComponentFixture<C>` con istanza, elemento host e metodi per il rendering. Configurare o sovrascrivere i provider **prima** di creare il componente: la configurazione si blocca alla creazione della fixture.

`fixture.detectChanges()` avvia un controllo sincrono dei binding e il lifecycle; `await fixture.whenStable()` attende che Angular completi le attività che conosce e il rendering programmato. `fixture.nativeElement` espone il DOM (Document Object Model) host, `fixture.debugElement` le query Angular. Non confondere la creazione della fixture con il primo rendering completato, soprattutto con signal e lavoro asincrono.

Nei nuovi progetti Angular CLI (Command Line Interface) dalla v21, **Vitest** è il runner unit test stabile e predefinito (`ng test`), di norma con `jsdom`; `Jasmine` fornisce funzioni di spec/spy e `Karma` esegue i test in browser nei workspace legacy ancora supportati. La migrazione di un progetto esistente da Karma/Jasmine a Vitest è indicata come sperimentale: controllare builder, polyfill e matcher prima di cambiare runner. `jsdom` emula il DOM, ma non riproduce ogni funzione o comportamento di un browser reale.

## Test dei componenti

Un test di componente osserva il **contratto pubblico**: testo, controlli, eventi e output, non proprietà private. Per un componente standalone usare i suoi `imports` quando necessari; interrogare il DOM per ruolo, label o testo visibile quando possibile, e usare `DebugElement.query(By.css(...))` se serve accedere a direttive o provider Angular. L'assertion deve seguire l'evento e il ciclo di rendering, non un timeout arbitrario.

```ts
import { Component, signal } from '@angular/core';
import { TestBed } from '@angular/core/testing';
import { expect, it } from 'vitest';

@Component({ template: `<button (click)="count.update(n => n + 1)">{{ count() }}</button>` })
class Counter { readonly count = signal(0); }
it('aggiorna il testo al click', async () => {
  const fixture = TestBed.createComponent(Counter);
  await fixture.whenStable();
  const button = fixture.nativeElement.querySelector('button') as HTMLButtonElement;
  button.click();
  await fixture.whenStable();
  expect(button.textContent).toContain('1');
});
```

Per un `input()` required usare `fixture.componentRef.setInput('nome', valore)` prima del controllo dei binding; `setInput` segue la semantica degli input Angular e notifica la change detection. Verificare gli `output()` sottoscrivendoli oppure con un host che li lega a un handler, poi attivare l'azione nel DOM. Se il componente legge `valueChanges` o Observable da service, fornire una sorgente controllata, emettere un valore e attendere il rendering; per i [Signals e change detection](05-signals-change-detection.md#change-detection) vedi il capitolo dedicato.

Per i form, testare validator e stato senza DOM quando basta; aggiungere un test del componente per binding, messaggi, submit e accessibilità. Modificare `input.value` non avvisa Angular: inviare anche `input.dispatchEvent(new Event('input'))`. Le Signal Forms richiedono un injection context per `form()` nei test isolati, per esempio `TestBed.runInInjectionContext()`; vedi i [form Angular](07-forms.md#signal-forms-e-coesistenza).

## Test di service e DI

Un service senza dipendenze Angular può essere istanziato direttamente; `TestBed.inject()` serve quando il comportamento dipende dalla dependency injection (DI), da token, scope o provider. `useValue` sostituisce una dipendenza con un valore controllato; `overrideProvider` modifica un provider prima di creare la fixture o iniettare l'istanza. Per provider dichiarati nel componente può servire `overrideComponent`, perché l'injector del componente è figlio di quello di TestBed.

```ts
import { Injectable, InjectionToken, inject } from '@angular/core';
import { TestBed } from '@angular/core/testing';
import { expect, it } from 'vitest';

const RATE = new InjectionToken<number>('RATE');
@Injectable()
class PriceService {
  private readonly rate = inject(RATE);
  total(net: number): number { return net * (1 + this.rate); }
}
it('usa il provider del test', () => {
  TestBed.configureTestingModule({ providers: [PriceService, { provide: RATE, useValue: 0.2 }] });
  expect(TestBed.inject(PriceService).total(100)).toBe(120);
});
```

Un service finto deve rispettare il contratto che il consumer usa, inclusi errori e tempi di risposta rilevanti. Per l'[injection context e gli scope](04-services-di-lifecycle.md#injection-context-e-inject) vedi il capitolo service.

## Test HTTP e routing

Per testare richieste HTTP (Hypertext Transfer Protocol), `provideHttpClientTesting()` sostituisce il backend di `HttpClient` senza rete; `HttpTestingController.expectOne()` richiede esattamente una richiesta corrispondente e restituisce un `TestRequest`. `flush(body, options?)` consegna risposta o errore HTTP; `error(...)` simula un errore di rete. `verify()` fallisce se rimangono richieste inattese: usarlo anche in `afterEach`. Se si configura `HttpClient` nel test, mettere `provideHttpClient(...)` **prima** di `provideHttpClientTesting()`, altrimenti si può sovrascrivere il backend di test.

```ts
import { HttpClient, provideHttpClient } from '@angular/common/http';
import { HttpTestingController, provideHttpClientTesting } from '@angular/common/http/testing';
import { TestBed } from '@angular/core/testing';
import { expect, it } from 'vitest';

it('intercetta una richiesta senza rete', () => {
  TestBed.configureTestingModule({ providers: [provideHttpClient(), provideHttpClientTesting()] });
  const http = TestBed.inject(HttpClient);
  const backend = TestBed.inject(HttpTestingController);
  let name: string | undefined;
  http.get<{ name: string }>('/api/user').subscribe(user => name = user.name);
  backend.expectOne({ method: 'GET', url: '/api/user' }).flush({ name: 'Ada' });
  expect(name).toBe('Ada');
  backend.verify();
});
```

Per un service HTTP, asserire URL (Uniform Resource Locator), metodo, parametri e risultato; provare anche status di errore, cancellazione e retry quando previsti. Un test di interceptor deve registrarlo con `provideHttpClient(withInterceptors([fn]))`, poi controllare la richiesta intercettata. Il generic di `get<T>` non valida la risposta a runtime; per [HttpClient e interceptor](06-rxjs-http-routing.md#httpclient-e-interceptor) vedi il capitolo dedicato.

Per il Router usare `provideRouter(routes)` e `RouterTestingHarness.create()`: `await harness.navigateByUrl('/path', Page)` esegue la navigazione e può restituire il componente; verificare URL, guard, parametri e `routeNativeElement`. Evitare di sostituire tutto il Router con un mock quando si testa il routing. `RouterTestingModule` è deprecato: rimane nel codice legacy, mentre `provideRouter` è la scelta corrente.

## Mock e harness

Uno **stub** restituisce dati predisposti, uno **spy** registra chiamate, un **fake** implementa una versione semplificata funzionante; un **mock** può indicare un sostituto con aspettative sulle interazioni. Scegliere il doppio più piccolo che isola un confine esterno, senza duplicare la logica testata. Nei test di componente, preferire provider reali economici e sostituire rete, tempo o servizi non deterministici; pulire timer, subscription e stato condiviso fra spec.

**Angular Testing Library** è una libreria esterna sopra TestBed: `render()` e le query per ruolo/testo favoriscono test osservabili dall'utente, con meno setup manuale. I **Component Harnesses** del Component Dev Kit (CDK) espongono un'interfaccia di interazione stabile per widget riusabili, per esempio Angular Material; `TestbedHarnessEnvironment` li usa con una fixture e nasconde dettagli del DOM. Entrambi aggiungono un'astrazione utile quando riduce test fragili, ma non sono necessari per ogni componente.

| Strumento E2E | Punto di forza | Limite da considerare |
| --- | --- | --- |
| Playwright | Browser context isolati, locator con auto-wait e copertura Chromium/Firefox/WebKit. | Richiede gestire server, dati e browser nella Continuous Integration (CI). |
| Cypress | Runner interattivo, command log e retry automatico di query/assertion. | Comandi accodati con semantica diversa dalle Promise; WebKit è sperimentale. |

Usare pochi E2E per percorsi critici e molti test mirati per regole e componenti; i contract test proteggono il confine con servizi indipendenti. La **coverage** misura codice eseguito, non qualità delle assertion: scegliere casi da rischi, errori e confini, evitando test che replicano l'implementazione. Per la [strategia generale di test](../../topics/engineering/15-architecture-devops.md#qualità-e-tradeoff) vedi il capitolo condiviso.

## Test dei flussi reattivi

Per un `signal` testare valore iniziale, `set`/`update` e `computed` leggendo i valori; per l'effetto visibile nel template attendere `fixture.whenStable()`. Per Observable controllare separatamente emissioni, errori, completion e unsubscription, usando `Subject` o stream finti quando serve determinare l'ordine. Nei test NgRx, reducer e selector si chiamano come funzioni pure; Effects e SignalStore richiedono sorgenti e dipendenze controllate ([NgRx](08-ngrx-state.md#scelta-e-confini-dello-stato)). La [semantica RxJS (Reactive Extensions for JavaScript)](../../topics/rxjs/10-rxjs.md#observable-e-subscription) resta nel capitolo canonico.

Preferire `async`/`await` per Promise e stabilizzazione della fixture. I fake timer del runner sono utili per debounce e timeout: avanzare il clock e ripristinare i timer reali dopo il test; una Promise usa microtask, `setTimeout` usa macrotask, quindi un clock avanzato non garantisce da solo che ogni callback o render Angular sia terminato. `fakeAsync`, `tick()` e `flushMicrotasks()` sono utility basate su Zone.js: non funzionano nel setup Vitest standard; per test legacy la guida di migrazione documenta `zone.js/plugins/vitest-patch`. Con Karma/Zone.js restano disponibili. Non mescolare clock di Vitest e `fakeAsync` nello stesso test. Un test affidabile attende la condizione osservabile, invece di dormire per un numero fisso di millisecondi.
