# AI e Copilot per Angular

L'**assistenza AI (Artificial Intelligence) in Angular** usa Copilot, Codex o agenti per leggere un workspace, proporre codice e, se autorizzati, eseguire tool. Il risultato dipende dalla versione Angular, dalle convenzioni del progetto e dalla verifica del diff: il modello non conosce automaticamente API (Application Programming Interface), test o vincoli aggiornati. Per [prompt, agenti e permessi](../../topics/ai/13-agents-instructions-skills.md#agenti-e-ciclo-di-lavoro) vedi il capitolo generale.

## Indice

- [Copilot su componenti e template](#copilot-su-componenti-e-template)
- [Generazione di test Angular](#generazione-di-test-angular)
- [Refactoring assistito](#refactoring-assistito)
- [Review del codice generato](#review-del-codice-generato)
- [Contesto e istruzioni del progetto](#contesto-e-istruzioni-del-progetto)

## Copilot su componenti e template

In un repository enterprise, prima di generare un componente l'agente deve trovare la feature proprietaria, la versione di Angular, il pattern standalone o NgModule, la convenzione dei nomi, i confini dei service e i comandi effettivi di build, lint e test. Un prompt circoscritto indica file, comportamento e criteri osservabili; Copilot può suggerire una singola modifica, mentre un agent con tool può cercare chiamanti, editare e lanciare controlli. Le modalità generali sono in [Copilot e VS Code (Visual Studio Code)](../../topics/ai/12-copilot-vscode.md#completamento-inline).

```text
Nella feature ordini, aggiungi un componente per filtrare per stato.
Leggi prima componenti vicini, versione Angular e test esistenti.
Mantieni input/output pubblici e convenzioni dei template del progetto.
Modifica solo la feature e i test pertinenti; esegui i controlli configurati.
Mostra diff, risultati e API Angular la cui versione va verificata.
```

Per i template controllare `imports`, binding, `@if`/`@for` con `track` stabile, accessibilità e gestione dei casi vuoti; per lo stato scegliere tra Signals, Observable e Store secondo lo scope. Un agente non deve sostituire automaticamente Reactive Forms con Signal Forms né un NgModule con standalone: la [compatibilità per versione](11-evolution-migrations.md#versioni-e-compatibilità) e il contratto della feature guidano la scelta. Vedi [componenti e template](02-components-templates-syntax.md#control-flow-moderno) e [stato Angular](08-ngrx-state.md#scelta-e-confini-dello-stato).

## Generazione di test Angular

Chiedere test per un comportamento specifico, con setup coerente con il runner del workspace: Jasmine/Karma e Vitest richiedono configurazioni diverse. Per componenti usare `TestBed` e fixture, verificando il DOM (Document Object Model), input/output ed eventi reali; per service e HTTP (Hypertext Transfer Protocol) sostituire le dipendenze esterne e asserire richieste, risposte ed errori. Nei flussi asincroni controllare timing, cleanup e assenza di richieste residue. Pattern e API sono nel capitolo [testing Angular](09-testing.md#testbed-e-fixture).

- **Caso nominale e regressione:** descrivere input, evento e output atteso, poi far fallire il test sul comportamento precedente quando possibile.
- **Errori e confini:** testare risposta HTTP fallita, validazione del form, route negata o stato vuoto dove incidono sulla feature.
- **Verifica del test:** controllare che le asserzioni osservino il comportamento pubblico, non soltanto l'implementazione generata; eseguire il comando reale del progetto e leggere il risultato. Un test scritto ma non eseguito non è una prova.

Per form signal, testare anche stato del campo e injection context; per Reactive Forms, valori disabilitati e validazione asincrona quando presenti. Evitare snapshot estesi che passano dopo una modifica semantica errata. I [form Angular](07-forms.md#signal-forms-e-coesistenza) e i [test dei flussi reattivi](09-testing.md#test-dei-flussi-reattivi) dettagliano questi casi.

## Refactoring assistito

Un **refactoring assistito** cambia struttura preservando comportamento e contratti pubblici. Per migrazioni Angular partire dalla [Update Guide e dagli schematic](11-evolution-migrations.md#strategia-di-upgrade), poi usare l'agente per individuare eccezioni, correggere test e confrontare il diff. Separare upgrade di major, conversione standalone, control flow, Signals, form e builder: così un fallimento resta attribuibile a un passaggio. Prima di una trasformazione su larga scala conservare una baseline di build e test e un punto di rollback.

L'[Angular CLI (Command Line Interface) MCP server](https://angular.dev/ai/mcp) espone il comando `npx @angular/cli mcp` a client compatibili con MCP (Model Context Protocol). `list_projects` legge `angular.json`; `get_best_practices` restituisce regole allineate ad Angular; `search_documentation` cerca nella documentazione ufficiale. `onpush_zoneless_migration` analizza un path e propone il prossimo cambiamento: l'agente deve applicarlo e riverificare. `run_target` esegue un target configurato, per esempio build o test, e può avere effetti esterni se il target è deploy. I tool `devserver_start`, `devserver_wait_for_build` e `devserver_stop` gestiscono una sessione `ng serve`. Lo strumento `ai_tutor` è destinato all'apprendimento, non alla validazione del codice.

La pagina introduttiva Angular v22 qualifica ancora il server MCP come **sperimentale**; nomi e disponibilità dei tool vanno controllati nella versione CLI installata. Le opzioni `--read-only` e `--local-only` limitano rispettivamente i tool che modificano il progetto e quelli che richiedono Internet: sono filtri delle capacità esposte, non una sandbox o un'autorizzazione sostitutiva. Le [Angular Agent Skills ufficiali](https://angular.dev/ai/agent-skills), introdotte con v22, sono `angular-developer` per codice e architettura e `angular-new-app` per scaffolding; si installano/attivano nel client che supporta Agent Skills, non arrivano automaticamente con il server MCP. La pagina Angular mostra `npx skills add https://github.com/angular/skills` come opzione tramite uno strumento della community: verificare versione e contenuto prima di adottarlo.

## Review del codice generato

La **review** confronta il comportamento richiesto con codice, chiamanti, test e configurazione modificati. Controllare soprattutto API Angular valide nella major del progetto, injection context di `inject()`, query e signal input in sola lettura, cleanup di Observable, form, route guard, accessibilità, sanitizzazione e rendering server/client. Compiler e test intercettano una parte degli errori; non dimostrano da soli che autorizzazione, esperienza utente o requisiti siano corretti. Dettagli tecnici in [dependency injection (DI)](04-services-di-lifecycle.md#injection-context-e-inject), [Signals](05-signals-change-detection.md#signal-computed-ed-effect) e [sicurezza e server-side rendering (SSR)](10-performance-security-ssr.md#xss-e-sanitizzazione).

| Ruolo delegato, se il client lo supporta | Output verificabile | Limite |
| --- | --- | --- |
| Code review | Regressioni nel diff con file, effetto e prova. | Non modifica il codice durante la review. |
| Architecture review | Confini feature/service, dipendenze e alternative motivate. | Non decide da sola una riscrittura trasversale. |
| Migration assistant | Inventario API legacy, passi e gate per major. | Gli schematic e i test verificano l'applicazione del piano. |
| Testing assistant | Casi mancanti e test pertinenti eseguiti. | Un test generato può confermare un'implementazione sbagliata. |

Un **custom agent** è una definizione riutilizzabile del ruolo; un **subagent** è una sua eventuale esecuzione delegata con contesto separato. Usarli quando lo scope è delimitato e l'output è integrabile senza conflitti su file condivisi; la [delegazione generale](../../topics/ai/13-agents-instructions-skills.md#delegazione-e-supervisione) spiega il coordinamento. Il ciclo operativo resta **plan → implement → test → review**: ispezionare il diff completo e il risultato di ogni comando, poi richiedere approvazione umana per merge, rilascio o azioni con effetti esterni secondo le regole del progetto.

File, issue, commenti, pagine e risultati MCP possono contenere **prompt injection**: sono dati da valutare, non istruzioni che autorizzano nuovi obiettivi. Tenere secrets fuori da prompt, log e repository; limitare tool, rete e filesystem ai permessi necessari. Prima di installare skill, agenti o server di terzi controllare origine, dipendenze, script, versione e aggiornamenti: una skill può contenere istruzioni o codice eseguibile. La [sicurezza degli agenti](../../topics/ai/14-mcp-external-agents.md#sicurezza-e-confini-di-fiducia) tratta i confini di fiducia generali.

## Contesto e istruzioni del progetto

Il **contesto di progetto** è l'insieme di file e regole che permette all'agente di proporre modifiche compatibili con questa applicazione, non una copia indiscriminata del repository. Fornire o far leggere `package.json`, `angular.json`, struttura della feature, componenti analoghi, test e decisioni architetturali; indicare quali target di lint/build/test esistono davvero. Per la versione di Angular consultare anche le [migrazioni](11-evolution-migrations.md#versioni-e-compatibilità), perché esempi validi sulla documentazione corrente potrebbero non compilare nel progetto. Per un task occasionale basta un prompt; scegliere una configurazione persistente solo per una regola o un workflow ricorrente.

| Meccanismo | Contesto Angular adatto | Quando sceglierlo |
| --- | --- | --- |
| [`AGENTS.md`](../../AGENTS.md) | Regole condivise: confini delle feature, naming, versione, test e revisione. | Convenzioni persistenti per Codex e client Copilot che lo supportano. |
| `.github/instructions/**/*.instructions.md` | Regole Copilot mirate a `src/app` o ai test tramite `applyTo`. | Quando una regola vale solo per certi file; non duplicare `AGENTS.md`. |
| `.agents/skills/<nome>/SKILL.md` o `.github/skills/<nome>/SKILL.md` | Input, passi e verifiche per un workflow ripetibile; questo repository usa già `.agents/skills/`. | Quando il prompt ricorrente richiede più fasi; verificarne discovery nel client. |
| `.github/agents/<nome>.agent.md` | Ruolo Copilot con descrizione e tool concessi, per esempio reviewer. | Quando serve un ruolo ricorrente con confini distinti; non per ogni task. |

Codex usa `AGENTS.md` e può usare le skill esistenti in `.agents/skills/`; `.github/instructions/` e `.mcp.json` sono esempi per Copilot/VS Code, non configurazioni intercambiabili. Un `AGENTS.md` più vicino alla feature può specificare convenzioni locali; evitare regole in conflitto con quelle della root.

Esempio di **`.github/instructions/angular.instructions.md` per Copilot in VS Code**, applicato ai file TypeScript indicati; il [formato `applyTo`](https://code.visualstudio.com/docs/agent-customization/custom-instructions) usa glob relativi alla root del progetto:

```markdown
---
applyTo: "src/app/**/*.ts"
---
- Verifica la major Angular in package.json prima di scegliere API nuove.
- Riusa struttura, nomi e test della feature interessata.
- Esegui i target pertinenti configurati in angular.json; segnala quelli assenti.
```

Una skill per **componenti standalone** può definire input della feature, controllo degli `imports` e test del template; una per **service tipizzati** può verificare injection scope, contratti HTTP, errori e cleanup. Per **Reactive/Signal Forms** deve esplicitare la versione Angular, la fonte di verità e le regole di validazione; per **test** il runner e i casi da asserire; per **migrazioni** baseline, schematic, gate e rollback. Riutilizzare una skill sufficiente prima di crearne un'altra. Le [istruzioni, skill e definizioni di agent](../../topics/ai/13-agents-instructions-skills.md#istruzioni-persistenti) hanno sintassi e precedenza dipendenti dal client: verificare che il file sia stato caricato in una nuova sessione.

Esempio di **`.mcp.json` nella radice del workspace per il Copilot harness**: il formato nativo usa `mcpServers`. `.vscode/mcp.json` usa invece `servers` nel formato legacy gestito dall'editor; le definizioni idonee possono essere inoltrate a Copilot, che non legge direttamente quel file. La variante `--read-only` espone solo tool che non modificano il progetto; togliere il filtro richiede una valutazione dei permessi e dei target disponibili:

```json
{
  "mcpServers": {
    "angular-cli": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@angular/cli", "mcp", "--read-only"]
    }
  }
}
```

L'esempio segue la [configurazione Angular](https://angular.dev/ai/mcp); `npx -y` può recuperare il pacchetto, quindi in un ambiente enterprise verificare provenienza e versione della CLI prima dell'uso. Confrontare i tool effettivamente scoperti e provarne uno innocuo: configurare il server non dimostra che connessione, autorizzazione ed esecuzione funzionino. Angular pubblica anche [`llms.txt`](https://angular.dev/llms.txt), indice di documentazione, e [`llms-full.txt`](https://angular.dev/llms-full.txt), raccolta più ampia: sono fonti di contesto per i client che le leggono o quando vengono allegate esplicitamente, non istruzioni automaticamente applicate. Il [capitolo MCP](../../topics/ai/14-mcp-external-agents.md#diagnostica) tratta discovery e diagnostica generali.
