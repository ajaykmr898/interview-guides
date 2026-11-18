# Node e npm

Node.js è un runtime per eseguire JavaScript fuori dal browser; npm è il package manager associato al registry di pacchetti usato comunemente nei progetti Node. Il linguaggio resta JavaScript, mentre API disponibili, moduli e modello di esecuzione dipendono dal runtime.

## Indice

- [Runtime Node](#runtime-node)
- [Moduli e risoluzione](#moduli-e-risoluzione)
- [Backend HTTP ed Express](#backend-http-ed-express)
- [Package e lockfile](#package-e-lockfile)
- [Script npm e dipendenze](#script-npm-e-dipendenze)
- [Versioni e sicurezza delle dipendenze](#versioni-e-sicurezza-delle-dipendenze)
- [Debug e diagnostica](#debug-e-diagnostica)

## Runtime Node

Node usa V8 per eseguire JavaScript e **libuv**, libreria nativa che gestisce l'event loop e parte dell'input/output (I/O) asincrono. Il browser fornisce invece API come DOM (Document Object Model) e rendering; Node espone tra l'altro filesystem, processi e rete. Nessuno dei due runtime rende asincrono il codice JavaScript che occupa il thread principale.

| Aspetto | Node.js | Browser |
| --- | --- | --- |
| Event loop | Fasi libuv, tra cui timer, poll per I/O e check per `setImmediate()`; `process.nextTick()` ha una coda dedicata. | Task, microtask e opportunità di rendering gestiti dal browser; niente `process.nextTick()` o `setImmediate()` di Node. |
| API host | `node:fs`, `node:http`, `process`, `Buffer`, stream. | DOM, eventi della pagina e API Web. |
| Promise | Le continuazioni sono microtask in entrambi, ma interagiscono con code e fasi host diverse. | Le microtask vengono eseguite prima che il browser prosegua con il task/rendering successivo. |

Un timer indica una soglia minima, non un istante garantito; anche l'ordine di `setTimeout(..., 0)` e `setImmediate()` fuori da una callback I/O non è fisso. Per il modello generale di [event loop e microtask](../javascript/04-javascript-async-dom.md#event-loop-e-task) si veda il capitolo JavaScript; le fasi libuv sono specifiche di Node.

Il modello **event-driven** registra callback per eventi e completamenti I/O: il thread JavaScript può servire altre richieste mentre il sistema operativo o il worker pool di libuv svolge alcune operazioni. Le API sincrone e il calcolo intenso bloccano comunque l'event loop; `node:fs/promises` restituisce Promise e usa operazioni asincrone, mentre `readFileSync` blocca il thread chiamante.

| Meccanismo | Quando usarlo | Limite principale |
| --- | --- | --- |
| Worker pool di libuv | Alcune API native asincrone, per esempio filesystem e parte della crittografia. | Non esegue automaticamente le funzioni JavaScript applicative. |
| Worker Threads | Calcolo JavaScript intenso in thread separati, con messaggi e possibile memoria condivisa. | Hanno costo di avvio e sincronizzazione; non accelerano di per sé l'I/O asincrono. |
| Child Processes | Eseguire un programma esterno o isolare un processo; `spawn` gestisce output come stream, `fork` avvia un processo Node con canale IPC (Inter-Process Communication). | Memoria separata e maggiore costo; comunicazione esplicita. |

`process` espone informazioni e controllo del processo: `process.env.NOME` legge una stringa o `undefined`, quindi configurazione e segreti vanno validati all'avvio. Per file piccoli `readFile(path, "utf8")` restituisce una Promise di stringa; senza encoding restituisce un `Buffer` di byte. Un errore di I/O rifiuta la Promise.

```js
import { readFile } from "node:fs/promises";
try {
  const configText = await readFile("config.json", "utf8");
  console.log(JSON.parse(configText));
} catch (error) {
  console.error("Configurazione non leggibile:", error);
  process.exitCode = 1;
}
```

`Buffer` è una sequenza di byte a lunghezza fissa, sottoclasse di `Uint8Array`, utile per dati binari. Gli **stream** (`Readable`, `Writable`, `Duplex`, `Transform`) elaborano dati a blocchi senza caricare necessariamente tutto in memoria. La **backpressure** rallenta il produttore quando il consumatore non tiene il passo: se `writable.write()` restituisce `false`, attendere `drain`; `pipeline()` coordina flusso, errori e chiusura degli stream.

```js
import { createReadStream, createWriteStream } from "node:fs";
import { pipeline } from "node:stream/promises";
await pipeline(createReadStream("input.bin"), createWriteStream("copy.bin"));
```

`EventEmitter` chiama i listener di `emit()` **sincronicamente** nell'ordine di registrazione: un evento non equivale a una callback asincrona. `emit()` restituisce un booleano che indica se esisteva almeno un listener; un evento `error` senza listener causa un errore non gestito. Rimuovere i listener non più necessari per evitare leak.

## Moduli e risoluzione

Node supporta **ESM (ECMAScript Modules)** e **CommonJS (CJS)**. Indicare esplicitamente `"type": "module"` in `package.json` per usare ESM nei file `.js`; `.mjs` è ESM e `.cjs` è CJS indipendentemente dal package. Evitare di assumere che import ed export siano intercambiabili tra i due sistemi.

| | ESM | CommonJS |
| --- | --- | --- |
| Sintassi | `import` / `export` | `require()` / `module.exports` |
| Valutazione | Supporta top-level `await`; risoluzione e interop seguono le regole ESM di Node. | Caricamento storicamente sincrono; le regole di `require()` e di interop con ESM dipendono anche dalla versione di Node. |
| Scelta | Preferibile per nuovi pacchetti quando gli strumenti lo supportano. | Necessario per dipendenze o strumenti ancora CJS. |

Il campo `"exports"` del package definisce entry point pubblici e può selezionarli per condizioni come `import` e `require`; può impedire deep import di file interni. `"imports"` definisce alias interni al package con prefisso `#`. Per le librerie pubblicate verificare entrambe le modalità di consumo e la risoluzione del runtime; un alias di un bundler non viene risolto automaticamente da Node. La [differenza tra moduli ESM e CJS nel tooling](09-build-tools.md#esm-e-commonjs) ha implicazioni anche per i build tool.

## Backend HTTP ed Express

Un server **HTTP (Hypertext Transfer Protocol)** riceve richieste con metodo, URL (Uniform Resource Locator), header e corpo e produce una risposta con status, header e corpo. `node:http` fornisce primitive di basso livello; il server va chiuso in modo ordinato durante lo shutdown e gli errori di rete vanno gestiti.

```js
import { createServer } from "node:http";
const server = createServer((req, res) => {
  if (req.method !== "GET" || req.url !== "/health") {
    res.writeHead(404).end();
    return;
  }
  res.writeHead(200, { "content-type": "application/json" }).end('{"ok":true}');
});
server.listen(3000, "127.0.0.1");
```

**Express** è un framework di routing e middleware sopra Node HTTP. Una route associa metodo e percorso a handler; un **middleware** può leggere/modificare richiesta e risposta, terminare la risposta oppure chiamare `next()` per proseguire. Se non fa nessuna delle due cose la richiesta resta aperta. Il middleware di errore usa quattro parametri `(err, req, res, next)`; in Express 5 una Promise rifiutata da un route handler viene inoltrata automaticamente, mentre Express 4 richiede passaggio esplicito dell'errore.

```js
import express from "express"; // package installato, progetto ESM
const app = express();
app.get("/items/:id", (req, res) => {
  const id = Number(req.params.id);
  if (!Number.isSafeInteger(id) || id <= 0) return res.status(400).json({ error: "id non valido" });
  res.json({ id });
});
app.listen(3000, "127.0.0.1");
```

Un'API **REST (Representational State Transfer)** modella risorse tramite URL, metodi HTTP e rappresentazioni. Ogni richiesta contiene il contesto necessario senza dipendere da una sessione del client sul server; lo stato delle risorse può comunque risiedere sul server. `GET` legge, `POST` spesso crea, `PUT` sostituisce, `PATCH` aggiorna parzialmente e `DELETE` elimina; status 4xx indicano problemi del client, 5xx del server. Validare parametri e body al confine, limitare dimensioni dei payload e non assumere che una conversione numerica validi l'intera richiesta.

**Autenticazione** verifica l'identità, **autorizzazione** controlla i permessi per risorsa e operazione. **CORS (Cross-Origin Resource Sharing)** comunica ai browser quali origin possono leggere risposte cross-origin; non autentica il chiamante e non blocca client non browser. Il **rate limiting** limita richieste per identità, IP o token e risponde tipicamente con `429`; dietro più istanze richiede stato condiviso o applicazione al gateway, e gli IP di proxy vanno interpretati solo se affidabili.

## Package e lockfile

`package.json` dichiara identità, entry point, script, compatibilità del runtime (per esempio `engines`) e dipendenze. `node_modules` è l'installazione locale generata dal package manager, da non usare come fonte di verità né versionare: la disposizione dei package può variare. `package-lock.json` registra l'albero risolto, versioni e informazioni di integrità per installazioni npm riproducibili; va versionato con il manifest, ma non garantisce che il codice bloccato sia sicuro.

| Comando | Effetto e uso |
| --- | --- |
| `npm install` | Installa o aggiorna dipendenze, può modificare manifest e lockfile; serve quando si aggiungono o aggiornano package. |
| `npm ci` | Richiede un lockfile coerente, rimuove `node_modules` esistente e installa l'intero progetto senza riscrivere manifest o lockfile; preferibile in CI (Continuous Integration). |

Le opzioni che influenzano la forma dell'albero (per esempio `legacy-peer-deps`) devono essere coerenti tra generazione del lockfile e `npm ci`. Un lockfile fissa la risoluzione delle dipendenze, non il comportamento di script d'installazione o servizi esterni.

## Script npm e dipendenze

`npm` installa e gestisce package; `npx`/`npm exec` esegue un binario del progetto e, se manca, può scaricare un package nella cache ed eseguirlo. Specificare una versione e controllare il package prima di lanciare tool non presenti localmente: l'esecuzione di codice di terzi è un confine di fiducia.

La sezione `"scripts"` di `package.json` associa nomi a comandi (`npm run build`); i binari locali sono disponibili nel `PATH` dello script. Gli hook `pre<nome>` e `post<nome>` circondano lo script e gli script lifecycle come `preinstall`/`postinstall` possono eseguire codice durante l'installazione. I dettagli del lifecycle dipendono dal comando npm: non assumere che `npm ci` sia privo di script.

| Campo | Contratto |
| --- | --- |
| `dependencies` | Package necessari al funzionamento dell'app o libreria distribuita. |
| `devDependencies` | Strumenti usati per sviluppo, test e build, non necessari al runtime del prodotto distribuito. |
| `peerDependencies` | Compatibilità richiesta verso un package fornito dall'app host, tipica di plugin e librerie per framework; non è una copia privata isolata. |
| `optionalDependencies` | Package il cui fallimento d'installazione può essere tollerato; il codice deve gestirne l'assenza. |

Nelle versioni moderne npm può tentare di installare automaticamente i peer: un conflitto di versioni va risolto esplicitamente, non nascosto con `--legacy-peer-deps` senza comprenderne gli effetti. Una dipendenza di build può essere necessaria nella pipeline di rilascio pur essendo `devDependency`.

## Versioni e sicurezza delle dipendenze

**SemVer (Semantic Versioning)** usa `major.minor.patch` per comunicare cambiamenti incompatibili, funzionalità compatibili e correzioni; è una convenzione del publisher, non una garanzia. Le range nel manifest si applicano alla prossima risoluzione, mentre il lockfile conserva la versione già scelta.

| Specifica | Versioni ammesse per esempio con `1.2.3` |
| --- | --- |
| `^1.2.3` | Da `1.2.3` fino a prima di `2.0.0` (minor e patch compatibili dichiarate). |
| `~1.2.3` | Da `1.2.3` fino a prima di `1.3.0` (patch). |
| `1.2.3` | Solo la versione esatta; il pin non sostituisce il lockfile delle dipendenze transitive. |

Per versioni `0.x`, `^` è più restrittivo: `^0.2.3` resta sotto `0.3.0`. `npm audit` confronta le dipendenze con vulnerabilità note dal registry e segnala possibili rimedi; non rileva ogni rischio del codice o della supply chain. `npm audit fix` può modificare l'albero e il lockfile: esaminare il diff e rieseguire i test, soprattutto se si forza un aggiornamento maggiore.

| Manager | Caratteristica utile | Trade-off |
| --- | --- | --- |
| npm | Integrato nel flusso Node, lockfile `package-lock.json`, workspaces. | Installazione tipica in `node_modules`; attenersi al lockfile del progetto. |
| pnpm | Store condiviso per contenuto e struttura `node_modules` basata su link, che rende più evidenti dipendenze non dichiarate. | Alcuni tool assumono un layout piatto; lockfile proprio. |
| Yarn | Workspaces e modalità Plug'n'Play (PnP) nelle versioni moderne, senza `node_modules` nella configurazione PnP. | PnP può richiedere configurazione dei tool; è disponibile anche la modalità `node_modules`. |

I **workspaces** gestiscono più package locali da un progetto radice, collegandoli e coordinando installazioni e script; ogni package deve comunque dichiarare le proprie dipendenze. Usare il manager e il lockfile scelti dal repository, senza mescolarli. Per la produzione scegliere una release Node **LTS (Long-Term Support)** attiva o in manutenzione, verificando lo stato corrente sul sito Node e la compatibilità `engines`/CI invece di fissare nella guida un numero destinato a scadere.

## Debug e diagnostica

`node --inspect` abilita l'inspector per breakpoint e profiling; `--trace-warnings` aiuta a localizzare warning. Non esporre l'inspector su reti non fidate. Misurare latenza dell'event loop e uso del worker pool quando un server rallenta: una callback lunga o I/O sincrono può bloccare tutte le richieste sul thread principale.

Gestire errori dove c'è contesto per recuperarli: `try`/`catch` per Promise attese, callback error-first per API callback, listener `error` per gli emitter e `pipeline()` per stream. In un server, rispondere con status coerenti senza inviare stack trace o segreti al client; registrare errori strutturati con contesto della richiesta, separando log applicativi da output diagnostico. `process.stdout` e `process.stderr` sono stream distinti; durante lo shutdown chiudere server e risorse prima di terminare il processo.
