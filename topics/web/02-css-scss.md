# CSS e SCSS

**CSS (Cascading Style Sheets)** è il linguaggio di fogli di stile che descrive la presentazione dei documenti strutturati. Associa selettori e dichiarazioni di proprietà e valori agli elementi; la cascade risolve le regole concorrenti, separando struttura del contenuto e resa visiva.

## Indice

- [Cascade specificità ed ereditarietà](#cascade-specificità-ed-ereditarietà)
- [Box model e layout](#box-model-e-layout)
- [Flexbox e Grid](#flexbox-e-grid)
- [Responsive design](#responsive-design)
- [SCSS e organizzazione degli stili](#scss-e-organizzazione-degli-stili)
- [Animazioni e rendering](#animazioni-e-rendering)

## Cascade specificità ed ereditarietà

### Selettori, pseudo-classi e pseudo-elementi

Un selettore individua gli elementi a cui applicare una regola: `p` per tipo, `.card` per classe, `#menu` per identificatore, `[disabled]` per attributo. I combinatori descrivono relazioni: `A B` discendente, `A > B` figlio diretto, `A + B` fratello immediato, `A ~ B` fratelli successivi.

- Le **pseudo-classi** selezionano uno stato o una relazione, per esempio `:hover`, `:focus-visible`, `:nth-child()`; `:has()` seleziona un elemento in base ai suoi discendenti o ad altre relazioni.
- Gli **pseudo-elementi** selezionano una parte rappresentata dell'elemento, per esempio `::before`, `::after`, `::first-line`. Il contenuto generato con `content` non sostituisce contenuto semantico essenziale.
- `:is()` raggruppa alternative; `:where()` fa lo stesso con specificità zero. `:is()` e `:has()` assumono la specificità massima dei selettori nei loro argomenti.

### Cascade e specificità

La **cascade** sceglie il valore di ogni proprietà tra le dichiarazioni applicabili: considera origine (browser, utente, autore) e importanza, poi layer, specificità e ordine; a parità di specificità può intervenire anche la vicinanza di uno `@scope`. Un selettore più specifico non supera una dichiarazione di origine o layer con precedenza maggiore.

- La specificità si confronta per colonne: identificatori, classi/attributi/pseudo-classi, tipi/pseudo-elementi. Per esempio `#nav .item a` vale `1-1-1`, `.item a` vale `0-1-1`; non è un numero decimale.
- Gli stili inline normali prevalgono sui normali stili autore nei fogli CSS; `!important` cambia la precedenza e va riservato a casi motivati. Un'animazione `@keyframes` prevale sulle dichiarazioni normali, mentre una transizione attiva può prevalere anche su `!important`.
- `@layer` fissa l'ordine tra gruppi di regole della stessa origine: per dichiarazioni normali prevale il layer dichiarato dopo; le regole normali senza layer prevalgono su quelle nei layer. Con `!important` l'ordine dei layer si inverte.

```css
@layer base, components;
@layer base { .button { color: navy; } }
@layer components { .button { color: white; background: navy; } }
```

L'**ereditarietà** trasmette ai discendenti solo alcune proprietà, come `color` e `font-family`; `margin` e `border` normalmente non si ereditano. `inherit` forza l'eredità, `initial` usa il valore iniziale della proprietà, `unset` eredita se la proprietà è ereditabile e altrimenti usa il valore iniziale.

### Nesting CSS

Il nesting nativo consente regole annidate e `&` rappresenta il selettore padre. Serve a mantenere vicine varianti di un componente, ma annidamenti profondi rendono più difficili da leggere i selettori e possono aumentare la specificità. Con liste di selettori, la specificità segue la logica di `:is()`.

```css
.card {
  color: navy;
  &:hover { color: teal; }
  & > h2 { margin-block-start: 0; }
}
```

## Box model e layout

### Box model, display e visibilità

Ogni box comprende **content**, **padding**, **border** e **margin**. Con `box-sizing: content-box` (predefinito), `width` e `height` misurano il content box; con `border-box` includono padding e border, rendendo più prevedibile una larghezza assegnata. Il margin resta esterno; i margini verticali di box nel normale flusso a blocchi possono collassare.

| `display` | Effetto principale |
| --- | --- |
| `block` | Inizia su una nuova riga e partecipa al flusso a blocchi. |
| `inline` | Scorre nel testo; `width` e `height` non ne definiscono la dimensione. |
| `inline-block` | Scorre inline ma accetta dimensioni come un box. |
| `flex` / `grid` | Crea rispettivamente un contenitore di layout flessibile o a griglia. |
| `none` | Non genera un box e non occupa spazio nel layout. |

| Proprietà | Spazio nel layout | Visibilità e accessibilità |
| --- | --- | --- |
| `display: none` | Rimosso | Contenuto generalmente escluso anche dall'albero di accessibilità. |
| `visibility: hidden` | Conservato | Invisibile e generalmente non esposto alle tecnologie assistive. |

Nessuna delle due proprietà è una tecnica per nascondere solo visivamente testo destinato agli screen reader.

### Unità, funzioni e proprietà logiche

| Unità | Riferimento e uso tipico |
| --- | --- |
| `px` | Pixel CSS, non necessariamente pixel fisici; utile per bordi e dettagli. |
| `rem` | Dimensione del font dell'elemento radice; utile per una scala coerente. |
| `em` | Font del genitore per `font-size`, font dell'elemento stesso per molte altre proprietà; può accumularsi nei discendenti. |
| `%` | Riferimento definito dalla proprietà, spesso il containing block; per `height` serve una dimensione di riferimento determinabile. |
| `vw` / `vh` | Percentuale della viewport; `vh` può non riflettere le barre mobili dinamiche. |
| `dvw` / `dvh` | Viewport dinamica; segue l'interfaccia mobile, ma può variare durante lo scroll. |

`calc()` combina valori e unità compatibili; `min()` sceglie il minore, `max()` il maggiore, `clamp(min, preferito, max)` limita un valore entro un intervallo. Sono valutati dal browser e si adattano al contesto.

```css
.panel {
  width: min(100%, 60rem);
  padding-inline: clamp(1rem, 3vw, 2rem);
  margin-inline: auto;
}
```

Le **proprietà logiche** (`margin-inline`, `padding-block`, `inset-inline-start`) seguono direzione e modalità di scrittura invece di fissare `left`/`right` o `top`/`bottom`: sono utili per interfacce multilingue.

### Positioning e stacking context

Il **containing block** è il riferimento per dimensioni percentuali e offset; dipende dal tipo di posizionamento. In genere `absolute` usa l'antenato posizionato più vicino, mentre `fixed` usa la viewport; `transform` su un antenato può cambiare il riferimento di `fixed` e `absolute`.

| `position` | Comportamento |
| --- | --- |
| `static` | Flusso normale; gli offset non hanno effetto. |
| `relative` | Rimane nel flusso e si sposta rispetto alla posizione originale. |
| `absolute` | Esce dal flusso; si posiziona rispetto al containing block. |
| `fixed` | Esce dal flusso; normalmente resta ancorato alla viewport. |
| `sticky` | Resta nel flusso e si ancora durante lo scroll entro i limiti del contenitore; richiede un offset come `top`. |

`z-index` ordina box nello stesso **stacking context**, non in tutta la pagina. Un elemento posizionato con `z-index` diverso da `auto`, `transform`, `opacity < 1`, oppure `position: fixed`/`sticky` possono creare nuovi stacking context: un figlio non può scavalcare con `z-index` un contesto fratello di livello superiore. Un antenato con `overflow` può anche impedire l'effetto sticky atteso.

## Flexbox e Grid

### Flexbox

Flexbox dispone gli elementi principalmente su **un asse**: il main axis segue `flex-direction`, il cross axis è perpendicolare. `justify-content` distribuisce spazio lungo il main axis; `align-items` allinea lungo il cross axis. Cambiando `flex-direction` cambiano anche i riferimenti, non i nomi delle proprietà.

- `flex-basis` stabilisce la dimensione iniziale sull'asse principale; `flex-grow` distribuisce spazio positivo, `flex-shrink` riduce gli elementi quando manca spazio. Lo shorthand `flex: 1 1 12rem` esplicita i tre valori.
- `flex-wrap` permette più righe, ma ogni riga viene distribuita indipendentemente: non crea colonne allineate come Grid.
- Il minimo automatico degli item può impedire la riduzione sotto la dimensione del contenuto; `min-width: 0` (o `min-height: 0` con asse verticale) può essere necessario per consentire troncamento o overflow controllato.

```css
.toolbar { display: flex; align-items: center; gap: 1rem; }
.toolbar__search { flex: 1 1 12rem; min-width: 0; }
```

### Grid

Grid organizza **righe e colonne** tramite tracks. `fr` ripartisce lo spazio disponibile, `minmax()` vincola la dimensione di un track, `repeat(auto-fit, ...)` adatta il numero di colonne alle dimensioni disponibili. Le aree nominate con `grid-template-areas` sono utili per strutture bidimensionali esplicite; non devono cambiare l'ordine semantico del [DOM (Document Object Model)](01-html-web.md#browser-e-caricamento-delle-risorse).

```css
.cards {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(min(100%, 16rem), 1fr));
  gap: 1rem;
}
```

| Tecnica | Preferirla quando | Limite principale |
| --- | --- | --- |
| Flexbox | Si allineano elementi su un asse, per esempio una toolbar. | Le righe create da `flex-wrap` sono indipendenti. |
| Grid | Servono tracks coordinati su due assi, per esempio una griglia di card. | Richiede di definire o accettare una struttura di righe e colonne. |

## Responsive design

Il **responsive design** adatta layout e contenuti allo spazio disponibile, senza basarsi su modelli di dispositivo. Nell'approccio **mobile-first** le regole base funzionano in spazi stretti; media query `min-width` aggiungono layout più ampi quando il contenuto lo richiede. I breakpoint vanno scelti in base al layout, non a dimensioni nominali dei telefoni.

```css
.layout { display: block; }
@media (min-width: 48rem) {
  .layout { display: grid; grid-template-columns: 2fr 1fr; gap: 2rem; }
}
```

Le **media queries** interrogano caratteristiche della viewport o dell'ambiente (`width`, `hover`, preferenze utente). Le **container size queries** interrogano invece un antenato dichiarato con `container-type`, così un componente risponde allo spazio che riceve; la query applica stili ai discendenti, non al contenitore che misura.

```css
.card-list { container-type: inline-size; }
@container (min-width: 40rem) {
  .card { display: grid; grid-template-columns: 10rem 1fr; }
}
```

`overflow: auto` aggiunge scorrimento quando necessario; `hidden` ritaglia il contenuto e può influire su sticky e focus visibile. Evitare larghezze rigide che creano scorrimento orizzontale. `aspect-ratio` riserva un rapporto tra dimensioni, ma contenuti con dimensioni intrinseche o vincoli possono modificare la dimensione finale.

Per immagini responsive, `max-width: 100%; height: auto` evita che un'immagine superi il contenitore e conserva il rapporto; `object-fit: cover` riempie un box di dimensioni fissate ritagliando l'immagine. In [HTML (HyperText Markup Language)](01-html-web.md#struttura-del-documento), `srcset` e `sizes` permettono al browser di scegliere una risorsa adatta alla risoluzione e allo spazio previsto, riducendo download inutili.

## SCSS e organizzazione degli stili

### SCSS e moduli Sass

**SCSS (Sassy CSS)** è la sintassi di **Sass (Syntactically Awesome Style Sheets)** compatibile con CSS: il compiler produce CSS prima che il browser lo legga. Le variabili `$name` sono risolte in fase di compilazione; nesting, `@mixin`/`@include` e `@function` aiutano a riusare regole e calcoli quando il CSS generato lo giustifica. Annidare molti livelli crea selettori difficili da mantenere anche in SCSS.

```scss
@use "sass:math";
$gap: 1rem;
@mixin stack($space) { display: grid; gap: $space; }
@function half($value) { @return math.div($value, 2); }
.list { @include stack($gap); padding: half($gap); }
```

`@use` carica un modulo e ne espone variabili, mixin e funzioni con namespace nel file corrente; `@forward` riesporta i membri pubblici di un modulo da un punto d'ingresso, senza renderli automaticamente disponibili nel file che inoltra. Il vecchio `@import` di Sass crea uno spazio globale ed è **deprecato in Dart Sass dalla versione 1.80.0**: per nuovo codice preferire `@use`/`@forward`. Questo non equivale al diverso `@import` CSS.

| | CSS custom properties | Variabili SCSS |
| --- | --- | --- |
| Valutazione | Nel browser, nella cascade; possono cambiare a runtime. | In compilazione; il browser riceve il valore prodotto. |
| Ambito | Seguono cascade ed eredità, salvo definizioni specifiche. | Ambito lessicale/modulare Sass. |
| Uso tipico | Temi e valori dipendenti dal contesto; `var(--color, fallback)`. | Costanti e calcoli per generare CSS. |

Le **custom properties** CSS si dichiarano con `--nome` e si leggono con `var()`. Il fallback di `var()` opera se la proprietà è assente o non valida come custom property; non sostituisce automaticamente un valore presente ma incompatibile con la proprietà CSS che lo usa.

```css
:root { --accent: rebeccapurple; }
.button { color: var(--accent, navy); }
.theme-dark { --accent: plum; }
```

CSS moderno offre già nesting, custom properties, funzioni matematiche e layer; SCSS resta utile per moduli, mixin o calcoli in compilazione. Valutare il CSS generato: un mixin incluso molte volte duplica le dichiarazioni.

### Organizzazione e manutenzione

- **BEM (Block Element Modifier)** usa nomi come `.card__title--compact` per distinguere un componente autonomo (block), una sua parte (element) e una variante (modifier), limitando dipendenze dalla struttura DOM; può diventare verboso.
- **Utility-first** compone classi a singola responsabilità direttamente nel markup; accelera combinazioni ricorrenti, ma richiede convenzioni condivise per evitare markup difficile da leggere.
- I **design tokens** sono nomi condivisi per colori, spaziature e tipografia. Possono essere pubblicati come custom properties per temi runtime o trasformati in variabili SCSS durante la build; il formato scelto dipende dai consumatori.

Preferire selettori semplici, componenti isolabili e una strategia coerente per l'ordine dei layer. Misurare dimensione del CSS, regole inutilizzate e costo di ricalcolo prima di ottimizzare: selettori complessi e override ripetuti peggiorano soprattutto la manutenzione; non ogni nesting è un problema di performance.

## Animazioni e rendering

`transform` modifica la rappresentazione visiva (`translate`, `scale`, `rotate`) senza spostare gli altri box nel layout; può creare uno stacking context. Una **transition** interpola il cambio di una proprietà tra due stati; `@keyframes` con `animation` descrive invece più stati e può ripetersi. Proprietà come `transform` e `opacity` possono spesso essere animate senza ricalcolare il layout, ma il lavoro di paint/compositing dipende dal browser e dal contenuto: verificare con gli strumenti di performance.

```css
.button { transition: transform 180ms ease; }
.button:hover { transform: translateY(-2px); }
@media (prefers-reduced-motion: reduce) {
  .button { transition: none; }
}
```

`prefers-reduced-motion` rileva una preferenza di sistema: ridurre o rimuovere i movimenti non essenziali senza eliminare feedback necessari. `will-change` va applicato solo dopo misurazione e per il tempo necessario, perché la promozione di layer consuma risorse.

Le **View Transitions** animano il passaggio tra due stati visivi: nello stesso documento `document.startViewTransition()` avvolge l'aggiornamento del DOM; tra documenti della stessa origine entrambe le pagine possono aderire con `@view-transition { navigation: auto; }`. Gli pseudo-elementi `::view-transition-*` consentono di personalizzare l'animazione. Sono un miglioramento progressivo: la navigazione o l'aggiornamento deve funzionare anche senza transizione, rispettando `prefers-reduced-motion`.
