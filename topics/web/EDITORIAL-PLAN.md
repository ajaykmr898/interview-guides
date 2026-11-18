# Piano editoriale — web

Requisiti specifici dei capitoli di questa directory. Per le regole editoriali comuni e i criteri di accettazione, seguire [AGENTS.md](../../AGENTS.md).

## `01-html-web.md` — HTML e Web Platform

**Da coprire:**
- Struttura documento: `<!DOCTYPE html>`, `html`, `head`, `body`, `lang`, charset, viewport; HTML vs HTML5.
- Tag, elementi, attributi; block/inline come concetti di rendering; elementi semantici vs generici.
- Semantica: `header`, `nav`, `main`, `section`, `article`, `aside`, `footer`; headings e gerarchia.
- Form: `form`, `label`, `input`, `select`, `textarea`, `button`; `name`, `value`, `type`, `autocomplete`, `required`, `pattern`; constraint validation; `GET` vs `POST`.
- Accessibility: WCAG, semantic HTML, ARIA, keyboard navigation, focus, `alt`, accessibilità dei form e messaggi d'errore.
- SEO e metadati: `title`, `meta description`, canonical, robots, Open Graph, structured data a livello introduttivo.
- DOM, BOM, CSSOM, render tree; parsing, reflow/layout, paint, compositing; script `defer` vs `async` vs module.
- Browser APIs: Fetch, Web Storage, Cookies, History, URL, Intersection/Mutation/ResizeObserver (definizioni brevi).
- HTTP/HTTPS: richiesta/risposta, metodi, status code, headers, caching, ETag, TLS, DNS, HTTP/2 e HTTP/3 (overview).
- Same-origin policy, CORS/preflight, XSS, CSRF, CSP; differenza authentication vs authorization.

**Confronti/snippet:** `section` vs `div`, `button` vs link, `async` vs `defer`, cookie vs localStorage vs sessionStorage; esempio di HTML semantico e form accessibile.

**Confine:** DOM manipulation e JavaScript event handling approfonditi nel capitolo JavaScript su asincronia e DOM.

## `02-css-scss.md` — CSS e SCSS

**Da coprire:**
- Cascade, specificity, inheritance, origin/layers, selettori, pseudo-classi, pseudo-elementi.
- Box model e `box-sizing`, display modes, unità `px`, `rem`, `em`, `%`, `vh`, `dvh`, ecc.
- Positioning: static, relative, absolute, fixed, sticky; containing block, `z-index` e stacking context.
- Flexbox: main/cross axis, align/justify, grow/shrink/basis; Grid: tracks, `fr`, areas, `minmax`, `auto-fit`.
- Responsive design, mobile-first, media queries, container queries; overflow, aspect ratio, immagini responsive.
- CSS variables/custom properties; `calc()`, `clamp()`, `min()`, `max()`; logical properties.
- Modern CSS: nesting, `@layer`, `:is()`, `:where()`, `:has()`; transforms, transitions, animations, View Transitions (overview).
- SCSS: variabili, nesting, mixin, funzioni, moduli `@use`/`@forward`; differenza SCSS vs CSS moderno.
- Organizzazione CSS: BEM, utility-first, design tokens; performance, maintainability e accessibility (`prefers-reduced-motion`).

**Confronti/snippet:** Flexbox vs Grid, `em` vs `rem`, `visibility:hidden` vs `display:none`, CSS custom properties vs SCSS variables.

**Confine:** ViewEncapsulation, `:host`, `::ng-deep` e component styling nel capitolo Angular su direttive e styling dei componenti.
