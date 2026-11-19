# Form Angular

Un **form Angular** collega controlli dell'interfaccia a un modello di valori, validazione e stato di interazione. Angular offre template-driven forms, Reactive Forms e Signal Forms: cambiano il proprietario dello stato e il modo in cui si dichiarano campi e regole, mentre la validazione finale dei dati resta responsabilità del server.

## Indice

- [Reactive forms](#reactive-forms)
- [FormControl FormGroup e FormArray](#formcontrol-formgroup-e-formarray)
- [Validazione sincrona e asincrona](#validazione-sincrona-e-asincrona)
- [Typed forms](#typed-forms)
- [Signal Forms e coesistenza](#signal-forms-e-coesistenza)
- [Stato e accessibilità](#stato-e-accessibilità)
- [ControlValueAccessor](#controlvalueaccessor)

## Reactive forms

| Approccio | Fonte dei valori e configurazione | Quando usarlo |
| --- | --- | --- |
| **Template-driven** | `FormsModule` abilita `[(ngModel)]` e crea `NgForm` sul `<form>`; validatori come `required` vivono nel template. | Form semplici; struttura e tipi del form meno espliciti. |
| **Reactive Forms** | `ReactiveFormsModule` lega `formGroup`/`formControlName` a un albero di `AbstractControl` costruito in TypeScript. | Form complessi, dinamici, con validazione e test espliciti. |
| **Signal Forms** | Un writable signal contiene il modello; `form()` crea l'albero di campi e regole, `FormField` li collega alla vista. | Progetti Angular v22+ che usano già Signals; dettagli nella sezione [Signal Forms e coesistenza](#signal-forms-e-coesistenza). |

Con `ngModel` dentro `NgForm` ogni controllo registrato richiede un `name`; `(ngSubmit)` gestisce l'invio. Per un controllo indipendente si può usare `[ngModelOptions]="{standalone: true}"`. Evitare `ngModel` sullo stesso controllo gestito da `formControlName`: le due forme competono per la stessa sorgente di verità. Per il [two-way binding](02-components-templates-syntax.md#template-binding) vedi i template.

```ts
import { Component } from '@angular/core';
import { FormControl, FormGroup, ReactiveFormsModule, Validators } from '@angular/forms';

@Component({ imports: [ReactiveFormsModule], templateUrl: './login.html' })
export class Login {
  readonly form = new FormGroup({
    email: new FormControl('', { nonNullable: true, validators: [Validators.required, Validators.email] }),
  });
  submit(): void { if (this.form.valid) console.log(this.form.getRawValue()); }
}
```

```html
<form [formGroup]="form" (ngSubmit)="submit()">
  <label for="email">Email</label>
  <input id="email" type="email" formControlName="email">
  <button type="submit" [disabled]="form.invalid || form.pending">Invia</button>
</form>
```

Il modello Reactive è la sorgente dei valori; il template collega il controllo e presenta lo stato. In un'app reale l'invio chiama un service e gestisce anche errori del server, anziché limitarsi a stampare i dati.

## FormControl FormGroup e FormArray

| Tipo | Cosa rappresenta | Uso e limite |
| --- | --- | --- |
| `FormControl<T>` | Valore, validatori e stato di un campo; `valueChanges` e `statusChanges` sono Observable. | Input singolo, anche senza gruppo. |
| `FormGroup<TControls>` | Controlli con chiavi note, anche annidati; aggrega valore e validità. | Schema stabile; la rimozione dinamica richiede chiavi opzionali nel tipo. |
| `FormArray<TControl>` | Sequenza indicizzata di controlli dello stesso tipo, aggiunti o rimossi a runtime. | Liste di campi ripetuti, con ordine significativo. |
| `FormRecord<TControl>` | Dizionario di controlli omogenei con chiavi non note in anticipo. | Campi dinamici identificati per nome. |

`FormBuilder` abbrevia la creazione con `control()`, `group()`, `array()` e `record()`. `NonNullableFormBuilder` (anche `fb.nonNullable`) crea controlli non-nullable per default; sceglierlo solo se `null` non è uno stato valido. Nei form generati da metadati, costruire controlli e template dalla stessa definizione e preservare l'identità dei controlli quando cambia l'ordine; `FormArray` e `FormRecord` coprono rispettivamente quantità variabile e chiavi variabili.

Su `FormGroup` e `FormArray`, `setValue(value)` richiede la struttura completa e lancia se mancano controlli; `patchValue(partial)` aggiorna solo i campi presenti e può ignorare chiavi sconosciute. Per un payload esterno completo preferire `setValue` dopo averlo validato, così un mismatch non passa inosservato. `updateOn: 'change' | 'blur' | 'submit'` controlla quando un input utente aggiorna il modello e la validazione; `change` è il default, `blur` o `submit` possono ridurre richieste di validazione costose.

## Validazione sincrona e asincrona

Un `ValidatorFn` riceve un `AbstractControl` e restituisce subito `ValidationErrors` oppure `null`; `Validators.required`, `email` e `minLength` coprono regole comuni. Un `AsyncValidatorFn` restituisce invece una Promise o un Observable finito dello stesso esito. Angular avvia gli async validator solo dopo il successo dei validator sincroni; durante l'attesa lo status è `PENDING`. Un Observable che non completa può lasciare il controllo in attesa. Per verifiche remote scegliere `updateOn: 'blur'` o un debounce appropriato e decidere esplicitamente cosa mostrare se la rete fallisce.

Una regola **cross-field** si applica al `FormGroup` quando confronta campi fratelli; l'errore appartiene al gruppo e va letto da `group.errors`, non appare automaticamente in `confirm.errors`.

```ts
import { ValidatorFn } from '@angular/forms';

export const passwordsMatch: ValidatorFn = group => {
  const password = group.get('password')?.value;
  const confirm = group.get('confirm')?.value;
  return password === confirm ? null : { passwordMismatch: true };
};
```

Si registra con `new FormGroup({ password: ..., confirm: ... }, { validators: passwordsMatch })`; `Validators.required` sui singoli campi gestisce i vuoti. Se i validator cambiano a runtime tramite `setValidators`/`addValidators`, chiamare `updateValueAndValidity()`. Mostrare errori leggibili dopo `touched` o un tentativo di invio, senza nascondere `PENDING`; la validazione client migliora l'esperienza ma non sostituisce quella server.

## Typed forms

I Reactive Forms tipizzati inferiscono il valore dai controlli: `new FormControl('')` ha tipo `FormControl<string | null>` perché `reset()` porta il valore a `null`. Con `{ nonNullable: true }` diventa `FormControl<string>` e `reset()` ripristina il valore iniziale: cambia anche il comportamento runtime, non solo TypeScript. Un `FormGroup` abilitato omette i controlli disabilitati da `.value`, quindi il tipo è parziale; `.getRawValue()` include anche i valori disabilitati. Inviare al server solo i campi voluti, senza presumere che un valore disabilitato sia sicuro da accettare.

## Signal Forms e coesistenza

Dalla v22 le **Signal Forms** in `@angular/forms/signals` sono stabili. `form(model, schema?)` riceve un writable signal e restituisce un `FieldTree` tipizzato dalla forma del modello, senza copiarne i dati; lo schema associa regole ai path. `[formField]` sincronizza campo e controllo nativo. `login.email().value()`, `valid()`, `invalid()`, `pending()`, `touched()` ed `errors()` leggono signal di stato; durante una validazione asincrona `valid()` e `invalid()` possono essere entrambi falsi.

```ts
import { Component, signal } from '@angular/core';
import { FormField, form, required } from '@angular/forms/signals';

@Component({
  imports: [FormField],
  template: `<label for="email">Email</label>
    <input id="email" [formField]="login.email">
    @if (login.email().touched() && login.email().invalid()) {
      <p role="alert">Email obbligatoria</p>
    }`,
})
export class SignalLogin {
  readonly model = signal({ email: '' });
  readonly login = form(this.model, path => required(path.email));
}
```

Il form richiede un injection context alla creazione. Il modello strutturale usa oggetti e array semplici: valori `undefined` non creano campi, mentre classi, `Map` e `Set` non sono supportati come contenitori. La forma dei dati resta nel signal; le regole nello schema possono essere condizionali e asincrone. Il modello del form non è `model()`, che serve alla [comunicazione tra componenti](02-components-templates-syntax.md#comunicazione-tra-componenti).

Reactive Forms rimane adatto a form complessi già basati su `FormControl`, Observable e librerie esistenti; Signal Forms è naturale quando il modello è già un signal. Possono coesistere nell'app: migrare per feature, non duplicare lo stesso campo in due sorgenti di verità. Se serve un ponte, `compatForm` incorpora controlli Reactive in un Signal Form e `SignalFormControl` inserisce un controllo signal in un `FormGroup`; entrambi sono in `@angular/forms/signals/compat` dalla v22. Per i [Signals](05-signals-change-detection.md#signal-computed-ed-effect) vedi il capitolo dedicato.

## Stato e accessibilità

| Stato Reactive | Significato |
| --- | --- |
| `pristine` / `dirty` | Nessuna modifica utente / valore modificato tramite l'interfaccia; `setValue()` programmatico non implica `dirty`. |
| `untouched` / `touched` | Nessuna interazione completata / blur o marcatura esplicita; `touched` non implica `dirty`. |
| `VALID` / `INVALID` / `PENDING` / `DISABLED` | Valido / errori / async validator in corso / escluso da validazione e valore aggregato del genitore. |

`valid`, `invalid`, `pending` e `disabled` sono proprietà booleane derivate dallo status; durante `PENDING` non trattare `!valid` come errore. `disable()` non è un controllo di autorizzazione né un modo per proteggere dati. `valueChanges`, `statusChanges` e `events` sono Observable di stato: una subscription imperativa di lunga durata richiede cleanup ([RxJS in Angular](06-rxjs-http-routing.md#integrazione-con-rxjs)). In un'app zoneless, modifiche programmatiche ai Reactive Forms non programmano da sole la change detection del componente; collegare lo stato a un signal letto nel template o notificare Angular ([change detection](05-signals-change-detection.md#onpush-e-zone)).

Associare ogni controllo a una `<label>`, esporre un messaggio specifico con `aria-describedby` e impostare `aria-invalid` quando l'errore è mostrato. Dopo un invio non valido, portare il focus al primo errore o a un riepilogo; non affidarsi al solo colore. Le [regole HTML e accessibilità](../../topics/web/01-html-web.md#form-e-validazione) sono nel capitolo Web.

Nei test, verificare valore iniziale, transizioni di validità, errori di gruppo, `PENDING` e invio; per validator asincroni controllare la risposta senza rete reale. Per Signal Forms testare lo schema con un `Injector` esplicito o `TestBed.runInInjectionContext()`, poi usare test del componente per binding, focus e controllo custom. Approfondimento nel capitolo [testing Angular](09-testing.md#test-dei-componenti).

## ControlValueAccessor

`ControlValueAccessor` (CVA) è il contratto che collega un controllo custom a `ngModel` o a un `FormControl`: il componente registra un provider `NG_VALUE_ACCESSOR` con `multi: true` e implementa questi metodi.

| Metodo | Direzione e responsabilità |
| --- | --- |
| `writeValue(value)` | Dal form alla vista: aggiorna il controllo senza richiamare il callback di change. |
| `registerOnChange(fn)` | Memorizza il callback da chiamare quando l'utente cambia il valore. |
| `registerOnTouched(fn)` | Memorizza il callback da chiamare al blur o interazione equivalente. |
| `setDisabledState(isDisabled)` | Riflette lo stato disabled nella vista; è opzionale nell'interfaccia, ma necessario se il controllo deve rispettarlo. |

Un CVA deve conservare valore, touched e disabled coerenti senza loop tra `writeValue` e `onChange`; la validazione resta nel form o in un validator associato, non implicita nel CVA. Per nuovi controlli Signal Forms v22, `FormValueControl` o `FormCheckboxControl` con `FormField` evita il CVA; `FormField` può comunque adattare un CVA esistente durante la migrazione. Non implementare entrambi i contratti sullo stesso componente.
