# Build tools

I build tools trasformano sorgenti e asset in file eseguibili o distribuibili, coordinando risoluzione dei moduli, compatibilità con il target e ottimizzazioni. Un tool può svolgere più operazioni, ma i termini seguenti descrivono passaggi diversi.

## Indice

- [Bundling e code splitting](#bundling-e-code-splitting)
- [Transpilation e polyfill](#transpilation-e-polyfill)
- [ESM e CommonJS](#esm-e-commonjs)
- [Tooling di sviluppo](#tooling-di-sviluppo)
- [Source map e analisi del bundle](#source-map-e-analisi-del-bundle)
- [Build in CI](#build-in-ci)

## Bundling e code splitting

| Operazione | Cosa produce | Cosa non implica |
| --- | --- | --- |
| Compilazione | Output da un linguaggio o formato sorgente, eventualmente con controlli statici. | Non implica un singolo file né minificazione. |
| Transpilation | Codice sorgente riscritto in altro codice sorgente, per esempio TypeScript in JavaScript o sintassi recente in sintassi meno recente. | Non implica type checking o supporto delle funzionalità mancanti nel runtime. |
| Bundling | Uno o più file risultanti da un grafo di `import` e dipendenze. | Non implica che tutto finisca in un unico file. |
| Minification | Codice equivalente, più compatto, con spazi e identificatori ridotti dove possibile. | Non aggiunge compatibilità né elimina automaticamente ogni codice inutilizzato. |

Il **code splitting** divide il grafo in **chunk** caricati in momenti diversi: riduce il JavaScript iniziale, ma aggiunge richieste e può duplicare dipendenze se la configurazione è errata. Un `import()` dinamico è un confine comune per il lazy loading; restituisce una Promise che risolve al namespace del modulo e rifiuta se caricamento, parsing o esecuzione falliscono. Il bundler può generarne un chunk separato, da verificare nell'output.

```js
// details.js esporta renderDetails()
async function showDetails() {
  try {
    const { renderDetails } = await import("./details.js");
    renderDetails();
  } catch (error) {
    console.error("Modulo non disponibile", error);
  }
}
```

Il **tree shaking** elimina esportazioni non usate tramite analisi statica, soprattutto con ESM (ECMAScript Modules). Non può scartare in sicurezza effetti collaterali: dichiarare `sideEffects: false` per un package che importa CSS (Cascading Style Sheets), polyfill o codice di inizializzazione può far sparire comportamento necessario. Verificare il bundle di produzione, non solo la dimensione apparente dei sorgenti.

**Webpack** costruisce un grafo a partire da uno o più **entry** e scrive asset secondo **output**. I **loaders** trasformano singoli tipi di modulo (per esempio CSS o TypeScript), mentre i **plugins** agiscono sul ciclo di compilazione e sugli asset complessivi. `import()` e la configurazione dei chunk abilitano code splitting; tree shaking e minificazione dipendono dalla modalità, dal formato dei moduli e dagli effetti collaterali dichiarati. È utile quando serve controllo dettagliato su una pipeline complessa, al costo di più configurazione.

## Transpilation e polyfill

Una trasformazione sintattica riscrive costrutti, per esempio una sintassi JavaScript recente verso un target più vecchio; un **polyfill** fornisce a runtime un'API (Application Programming Interface) assente, come una funzione globale o un metodo. Cambiare `target` non installa polyfill e alcuni comportamenti non sono riproducibili completamente sui runtime vecchi. Definire i browser o runtime supportati prima di scegliere trasformazioni e polyfill.

| Tool | Responsabilità | Limite da ricordare |
| --- | --- | --- |
| Babel | Trasforma sintassi JavaScript con plugin/preset; `@babel/preset-env` seleziona trasformazioni per i target. | Non effettua type checking TypeScript; per i polyfill serve una scelta esplicita di package e configurazione, che varia con la versione. |
| TypeScript compiler (`tsc`) | Controlla i tipi e può emettere JavaScript e dichiarazioni `.d.ts`. | Il target di emissione non fornisce API runtime mancanti; l'emissione può avvenire anche con errori salvo opzioni come `noEmitOnError`. |
| esbuild | Trasforma, bundle e minifica rapidamente JavaScript/TypeScript. | Rimuove i tipi TypeScript senza controllarli e non emette `.d.ts`; serve un controllo separato. |

Per il [sistema di tipi e la configurazione TypeScript](../typescript/07-typescript-advanced.md#configurazione-e-dichiarazioni) vedere il capitolo dedicato. In una pipeline che traspila TypeScript per singolo file, eseguire `tsc --noEmit` separatamente e verificare `isolatedModules`; non considerare un build riuscito come prova di correttezza dei tipi.

## ESM e CommonJS

ESM usa `import`/`export` e consente al bundler di analizzare staticamente le dipendenze; **CJS (CommonJS)** usa `require()`/`module.exports` e può avere import dinamici difficili da analizzare. Un tool può accettare un formato in input ed emetterne un altro, ma questo non rende identiche le semantiche di risoluzione, export, top-level `await` o esecuzione del codice.

Per applicazioni browser, scegliere output compatibile con i target e con il modo in cui lo si carica; per librerie, dichiarare entry point e formati coerenti con i consumer. Un package CJS può richiedere conversione o plugin, e il tree shaking può essere meno efficace. Per le [regole ESM/CJS di Node](08-node-npm.md#moduli-e-risoluzione) e la [semantica dei moduli JavaScript](../javascript/05-javascript-advanced-ecmascript.md#moduli-e-strict-mode) si vedano i capitoli canonici.

## Tooling di sviluppo

Un **dev server** serve i moduli durante lo sviluppo, osserva i file e aggiorna il browser. **HMR (Hot Module Replacement)** sostituisce moduli modificati senza ricaricare tutta la pagina quando esiste un confine di aggiornamento gestibile; lo stato può comunque andare perso o richiedere un reload. La build di produzione applica ottimizzazioni, chunking e target che possono differire dal percorso di sviluppo: testare anche l'output finale.

| Tool | Ruolo tipico | Quando sceglierlo e limite |
| --- | --- | --- |
| Webpack | Bundler configurabile con loaders, plugins e dev server. | Pipeline articolate o integrazioni esistenti; configurazione più impegnativa. |
| Vite | Dev server con trasformazioni su richiesta e HMR; comando di build per asset ottimizzati. | App web e avvio rapido; il transpile TypeScript non include type checking. |
| esbuild | Motore veloce di transpilation, bundling e minification, utilizzabile anche da altri tool. | Pipeline snelle; interfaccia dei plugin e trasformazioni non equivalgono a un framework di sviluppo completo. |
| Rollup | Bundler orientato ai moduli, tree shaking e output in più formati. | Librerie e build con controllo dei formati; dev server e gestione degli asset richiedono integrazioni. |

La composizione interna di Vite è cambiata nel tempo: la documentazione corrente descrive **Rolldown** per la build e il pre-bundling, mentre guide precedenti citano Rollup ed esbuild. Evitare di basare la scelta del tool su una pipeline interna presunta; verificare la versione del progetto. Una CLI (Command-Line Interface) di framework può inoltre orchestrare bundler, compiler e dev server: per Angular la scelta del builder è nella configurazione del workspace, trattata nella [guida Angular](../../guides/angular/01-core-architecture.md#build-e-configurazione).

## Source map e analisi del bundle

Una **source map** associa posizioni dell'output a quelle dei sorgenti, così debugger e stack trace possono mostrare il codice originale dopo transpilation, bundling o minification. Non modifica l'esecuzione; se pubblicata con sorgenti incorporati può rendere visibile codice che non si intende distribuire. Verificare percorso e accessibilità delle mappe nel deployment.

Gli **asset statici** (immagini, font, CSS) importati nel grafo possono essere trasformati, rinominati e referenziati con URL (Uniform Resource Locator) aggiornati; una directory pubblica del tool può invece copiare file senza trasformarli. Per il caching a lungo termine usare nomi con hash del contenuto e cache policy coerenti, evitando che HTML (HyperText Markup Language) o manifest puntino a chunk già eliminati da un nuovo deploy.

L'analisi del bundle misura dimensioni, dipendenze duplicate e chunk iniziali; un **performance budget** impone soglie verificabili su asset o entry point. Ridurre byte iniziali, costo di parsing/esecuzione e richieste critiche, non solo la dimensione totale: troppo chunking può peggiorare il caricamento. Le variabili d'ambiente sostituite nella build client diventano leggibili nel codice distribuito: in Vite, per esempio, le variabili con prefisso `VITE_` sono esposte a `import.meta.env`; non inserirvi segreti.

## Build in CI

In **CI (Continuous Integration)** usare installazioni riproducibili con il lockfile del package manager, poi eseguire type check, test e build di produzione come passaggi distinti. Controllare budget e output effettivo, inclusi asset e source maps; pubblicare gli artefatti solo dopo i controlli. Il [confronto tra `npm install` e `npm ci`](08-node-npm.md#package-e-lockfile) spiega perché il secondo è adatto a una pipeline con lockfile coerente.

Un errore solo in produzione può dipendere da tree shaking, minification, chunk caricati tardi, URL base o variabili sostituite alla build. Riprodurre con gli artefatti di produzione e verificare che il server li serva correttamente; evitare di assumere che il dev server validi il comportamento del deploy.
