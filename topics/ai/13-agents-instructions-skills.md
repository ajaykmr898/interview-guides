# Agenti istruzioni e skill

Un **agent AI** (Artificial Intelligence, intelligenza artificiale) è un sistema che usa un modello e, quando autorizzato, tool per scegliere e svolgere passi verso un obiettivo. Istruzioni, agenti personalizzati e skill ne guidano il comportamento a livelli diversi; il client o **harness** determina quali file riconosce e quali permessi concede.

## Indice

- [Agenti e ciclo di lavoro](#agenti-e-ciclo-di-lavoro)
- [Istruzioni persistenti](#istruzioni-persistenti)
- [Skill riutilizzabili](#skill-riutilizzabili)
- [Context engineering](#context-engineering)
- [Delegazione e supervisione](#delegazione-e-supervisione)
- [Verifica e sicurezza](#verifica-e-sicurezza)

## Agenti e ciclo di lavoro

Un agent interpreta il compito, pianifica quanto serve, agisce tramite tool, osserva l'esito e decide se proseguire o fermarsi: vedi [agent loop](11-ai-fundamentals.md#tool-use-e-function-calling). Un **custom agent** definisce una *configurazione riutilizzabile* per un ruolo, con istruzioni e, nei client che lo supportano, tool o modello preferito. Un **subagent** è invece un'*esecuzione delegata* con contesto proprio: può usare un custom agent, ma i due concetti non coincidono.

| Scelta | Quando serve | Attivazione tipica |
| --- | --- | --- |
| Prompt diretto | Richiesta occasionale o vincolo valido per un solo task. | Inserito dall'utente nella conversazione. |
| Istruzioni persistenti | Convenzioni che valgono per un progetto o per file mirati. | Caricate dal client quando sono applicabili. |
| Skill | Workflow ripetibile con passi, riferimenti o script. | Selezionata esplicitamente o dal modello, se il client lo consente. |
| Custom agent | Ruolo ricorrente con contesto o tool dedicati. | Selezionato come agente o usato in delegazione. |
| Prompt file | Testo riutilizzabile invocato dall'utente. | Comando esplicito nei client che lo supportano. |

Scegliere la soluzione più piccola che copre il bisogno: un file di istruzioni non crea tool, e una skill non richiede un nuovo agente. Tool esterni e protocollo MCP (Model Context Protocol) appartengono al [capitolo sulle integrazioni](14-mcp-external-agents.md#architettura-mcp).

## Istruzioni persistenti

Le **instructions** sono regole o convenzioni che il client inserisce nel contesto quando scopre un file applicabile; non sono controlli di sicurezza eseguibili. Questa tabella usa percorsi verificati per **VS Code** (Visual Studio Code) con Copilot harness e, dove indicato, Codex. **YAML** (YAML Ain't Markup Language) è il formato dei metadati opzionali o obbligatori tra delimitatori `---`; le cartelle sono esempi supportati, non posizioni universali.

| File | Scopo e posizione esemplificativa | Frontmatter e attivazione |
| --- | --- | --- |
| `AGENTS.md` | Regole di progetto nella root o in sottocartelle; supportato da Copilot e Codex. Codex può avere anche un file nella propria home. | Markdown senza frontmatter YAML; Codex legge root → directory di lavoro, con regole specifiche più tarde; Copilot combina i file applicabili lungo i percorsi su cui lavora. |
| `.github/copilot-instructions.md` | Convenzioni generali del repository per Copilot; una variante personale è `~/.copilot/copilot-instructions.md`. | Markdown senza frontmatter; Copilot lo include come istruzione di progetto/personale quando il relativo scope è attivo. Non è un formato nativo Codex. |
| `.github/instructions/**/*.instructions.md` | Regole Copilot mirate a file o aree del repository; per l'utente esiste `~/.copilot/instructions/`. | Frontmatter YAML opzionale; `applyTo` usa glob riferiti alla root del progetto. In Copilot si applica ai file su cui l'agente lavora; senza pattern non assumere attivazione automatica. |
| `.github/agents/*.agent.md` | Definizione di custom agent Copilot nel workspace; possibile anche `~/.copilot/agents/`. | Frontmatter YAML opzionale per nome, descrizione e tool; attivato scegliendo l'agente o delegandogli un compito. Nel Copilot harness le sue istruzioni prevalgono sulle istruzioni di progetto in caso di conflitto. |
| `<skill>/SKILL.md` | Workflow in una directory di skill: per esempio `.github/skills/` in Copilot o `.agents/skills/` in Codex. | Frontmatter YAML con `name` e `description` obbligatori secondo la specifica Agent Skills; si carica quando invocata o selezionata per il task, non per ogni richiesta. |
| `.github/prompts/*.prompt.md` | Prompt riutilizzabile del workspace nel **Local harness** di VS Code. | Frontmatter YAML opzionale; invocazione manuale come comando `/nome`. Il Local harness è deprecato; il Copilot harness non esegue questi prompt file. |

Per `AGENTS.md` e `copilot-instructions.md` basta Markdown, per esempio in **`AGENTS.md` nella root**:

```markdown
# Regole del repository

- Limita le modifiche ai file richiesti.
- Esegui i test pertinenti e riporta quelli non eseguibili.
```

Non esiste una precedenza unica fra client: Copilot combina le istruzioni applicabili senza ordine generale tra file; Codex costruisce la catena dalla root alla directory di lavoro e supporta `AGENTS.override.md` per sostituire il file della stessa directory. Le istruzioni dei ruoli della conversazione e i permessi del runtime restano distinti da questi file; in VS Code le istruzioni personalizzate non si applicano alle inline suggestions durante la digitazione. Evitare regole contraddittorie e verificare in una nuova sessione quali istruzioni sono state caricate. In VS Code i vecchi `*.chatmode.md` sono un formato precedente dei custom agent: migrare a `*.agent.md`.

## Skill riutilizzabili

Una **Agent Skill** è una cartella con `SKILL.md`: frontmatter per discovery e corpo Markdown con input, passi e risultato atteso. Può includere `references/` per dettagli consultati solo quando servono, `scripts/` per operazioni deterministiche e `assets/` per template. La **progressive disclosure** carica prima nome e descrizione, poi il corpo quando la skill è attivata e infine le risorse necessarie: limita l'uso della context window senza nascondere i requisiti essenziali.

Esempio di **`.github/skills/review-diff/SKILL.md` per Copilot**; la stessa struttura è utilizzabile in una posizione supportata da un altro client, dopo averne verificato discovery e tool:

```markdown
---
name: review-diff
description: Esamina un diff per trovare regressioni. Usa questa skill per revisionare modifiche prima del commit.
---
# Review del diff

1. Leggi il diff e i chiamanti coinvolti.
2. Riporta problemi con file e motivazione; non modificare il codice.
```

`name` deve corrispondere alla directory; `description` deve dire cosa fa la skill e quando usarla. In Copilot una skill può comparire come slash command o essere selezionata dal modello in base alla descrizione; discovery non garantisce selezione o esecuzione corretta. I campi opzionali e il supporto alle invocazioni variano per client. Una skill **locale al progetto** sta nel repository, una skill **user/personale** nella home del rispettivo harness; «globale» indica di solito uno scope del client, non una cartella comune a tutti gli agenti o a tutte le macchine.

## Context engineering

Il [context engineering](11-ai-fundamentals.md#prompt-e-contesto) sceglie istruzioni, file, esempi, tool e cronologia utili al task. Un file può essere scoperto senza essere incluso nella richiesta; un link a un riferimento non garantisce che venga letto. Indicare la fonte canonica e allegare o far leggere esplicitamente i dati indispensabili.

La **context window** è finita: istruzioni duplicate, log lunghi e output di subagent possono sottrarre spazio ai dettagli decisivi. La **context compaction** riassume cronologia e risultati per continuare il lavoro, ma può perdere eccezioni o decisioni; dopo una compattazione verificare stato dei file, vincoli e passi ancora aperti. La **memory**, dove il client la supporta, conserva preferenze o fatti fra sessioni con scope e durata propri; non sostituisce requisiti versionati nel repository né va usata per secrets.

## Delegazione e supervisione

La **delegation** assegna un sottocompito delimitato a un subagent; l'**orchestrazione** decide ordine, dipendenze e come integrare i risultati. Per un task grande, dividere per output verificabili (ricerca, implementazione, review), comunicando a ciascuno file, vincoli e criterio di completamento. Contesti separati possono ridurre rumore, ma non isolano i file: per evitare conflitti fra modifiche simultanee servono workspace o worktree separati, oltre a coordinamento.

Esempio di **`.github/agents/reviewer.agent.md` per Copilot harness**; gli alias `read` e `search` sono tool riconosciuti da quel client:

```markdown
---
name: Reviewer
description: Rivede modifiche e segnala regressioni senza editarle.
tools: ['read', 'search']
---
Esamina il diff e i file collegati. Riporta solo problemi verificabili con riferimenti ai file.
```

La definizione restringe i tool del ruolo, ma non crea nuovi permessi. Il delegante resta responsabile di confrontare i risultati con il codice reale; **human oversight** significa che una persona controlla piano, diff e azioni sensibili prima di integrarli o autorizzarli, secondo il rischio. Uno scope esplicito evita che un subagent trasformi una review in un'implementazione.

## Verifica e sicurezza

Testare separatamente **discovery** e **comportamento**: avviare una sessione con il client previsto, controllare quali istruzioni/skill/agenti compaiono, eseguire un task rappresentativo e verificare tool, file toccati ed esito. Dopo modifiche a prompt o configurazione, ripetere gli **evals** pertinenti; il fatto che un file sia stato caricato non prova che il modello lo abbia seguito.

Per i workflow agentici registrare passi, decisioni, comandi e risultati necessari a ricostruire un errore. Prima di integrare, esaminare il diff e i test; per il **rollback** conservare un punto di ritorno tramite controllo versione o worktree isolato, evitando di cancellare modifiche preesistenti. La procedura di rollback deve distinguere edit dell'agente da lavoro umano nella stessa directory.

Skill e agenti di terze parti sono istruzioni e, talvolta, script eseguibili: verificare autore, provenienza, dipendenze, comandi e cambiamenti a ogni aggiornamento. Trattare riferimenti esterni e output dei tool come dati non attendibili; una **prompt injection** può tentare di trasformarli in ordini. Applicare permessi minimi e approvazioni nel runtime, tenere fuori dal contesto credenziali e secrets e rivedere le azioni con effetti esterni: vedi [privacy e sicurezza](11-ai-fundamentals.md#privacy-e-sicurezza).
