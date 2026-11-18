# Copilot e VS Code

VS Code (Visual Studio Code) è un editor con workspace, terminale e strumenti di sviluppo; **GitHub Copilot** vi aggiunge suggerimenti di codice, chat e, nei client supportati, agenti che possono usare tool. La qualità dipende dal contesto disponibile e dalla verifica del risultato, non dalla sola presenza del repository.

## Indice

- [Completamento inline](#completamento-inline)
- [Chat e contesto del workspace](#chat-e-contesto-del-workspace)
- [Prompt operativi](#prompt-operativi)
- [Review e verifica degli output](#review-e-verifica-degli-output)
- [Privacy e configurazione](#privacy-e-configurazione)
- [Limiti e fallimenti](#limiti-e-fallimenti)

## Completamento inline

Le **inline suggestions** propongono codice mentre si scrive: il *ghost text* appare al cursore e le *next edit suggestions* possono suggerire modifiche collegate. Copilot usa il file corrente e altri file aperti come contesto; lo sviluppatore accetta, modifica o scarta ogni suggerimento. Accettare testo modifica l'editor, ma non autorizza l'esecuzione di comandi né verifica che il codice sia corretto.

| Modalità nel Copilot harness di VS Code | Chi guida il lavoro | Effetto |
| --- | --- | --- |
| Inline suggestions | Lo sviluppatore scrive e sceglie ogni proposta. | Completamenti o edit puntuali, applicati solo se accettati. |
| Chat **Ask** | Lo sviluppatore pone domande e valuta la risposta. | Ricerca e spiegazioni con tool di sola lettura; non modifica file. |
| Chat **Plan** | L'agente analizza la richiesta e propone i passi. | Piano da rivedere prima dell'implementazione; pianificare non equivale a modificare. |
| Chat **Agent** | Il modello sceglie passi e tool entro i permessi della sessione. | Può cercare, modificare file e lanciare controlli; le azioni restano soggette a permessi e review. |

Per una modifica locale e ben delimitata sono spesso sufficienti inline suggestions o inline chat; un agent è utile quando servono ricerca, più file e verifiche coordinate. Una proposta visibile non è una modifica applicata; una modifica applicata non è ancora una modifica approvata per il commit.

## Chat e contesto del workspace

Il **workspace** di VS Code è l'insieme di una o più cartelle aperte nella finestra, con configurazioni e task associati. Il **contesto dell'editor** comprende, quando incluso nella richiesta, selezione, file attivo e file aperti; l'agente può anche cercare file e simboli nel workspace. Aggiungere esplicitamente un file, una selezione o il risultato di un test riduce l'ambiguità su cosa esaminare.

La **workspace awareness** non significa che l'intero repository entri nel prompt. Ricerca testuale, simboli e indice semantico individuano materiale pertinente; un indice aiuta a trovare file, ma non li allega tutti alla conversazione. Disponibilità dell'indice, esclusioni, tool e harness cambiano ciò che l'agente può trovare. Nelle versioni che lo supportano, `#codebase` richiede una ricerca sul workspace; i file ignorati o esclusi possono non comparire nei risultati, mentre un file ignorato aperto nell'editor può comunque fornire contesto.

La **context window** del modello limita quanto testo, cronologia e output dei tool può essere considerato: vedi [modelli e token](11-ai-fundamentals.md#modelli-e-token). Per richieste multi-file indicare i file rilevanti e far cercare chiamanti, test e configurazione; controllare poi le fonti effettivamente lette. Il modello può non vedere un file decisivo anche se esiste nel workspace.

## Prompt operativi

Un prompt utile specifica risultato, ambito, vincoli e verifica. Per un cambiamento che tocca più file, chiedere prima di individuare chiamanti e test; per un task piccolo, indicare il file o la selezione e il comportamento atteso. Il [prompt e il context engineering](11-ai-fundamentals.md#prompt-e-contesto) spiegano la distinzione generale.

```text
Obiettivo: correggi il caso in cui la ricerca mostra risultati obsoleti.
Ambito: trova implementazione, chiamanti e test; modifica solo i file necessari.
Vincoli: conserva l'interfaccia pubblica e gestisci le richieste concorrenti.
Verifica: esegui i test pertinenti e mostra il diff con eventuali limiti.
```

Copilot può aiutare a generare codice e test, proporre un refactoring, interpretare un errore di debug e rivedere un diff. Fornire l'errore reale e un caso riproducibile è più utile di «non funziona»; per i test chiedere casi di successo, errore e regressione, poi controllare che le asserzioni esercitino davvero il comportamento. Nel **terminale integrato** l'agente può eseguire comandi tramite tool se abilitati; **Copilot CLI** (Command-Line Interface) è invece un'interfaccia da terminale distinta, con proprie sessioni e capacità. Non assumere che i due ambienti condividano automaticamente contesto o autorizzazioni.

Usare **Plan** quando occorre valutare approccio, file coinvolti e criteri di verifica; passare a **Agent** per l'editing dopo aver rivisto il piano. Un piano può contenere supposizioni errate: verificare i riferimenti al codice prima di implementarlo.

## Review e verifica degli output

Dopo una modifica, esaminare il **diff completo**: file fuori scope, API (Application Programming Interface) inventate, import, gestione degli errori, permessi e contratti pubblici. Verificare chiamanti e test collegati; eseguire test, lint o build pertinenti e leggere gli esiti, inclusi controlli saltati. Una risposta «test passati» senza comando e risultato osservabile non dimostra la correttezza.

Per il **refactoring**, confrontare il comportamento prima e dopo e controllare che il diff non cambi semantica per errore. Per il **debug**, partire da riproduzione, log e stack trace, poi chiedere una causa verificabile. Dove disponibile, **Copilot Code Review** può commentare una selezione o modifiche non committate; richiede comunque giudizio umano, soprattutto su sicurezza, accessibilità, prestazioni e dati.

Le modifiche dell'agente possono essere salvate su disco prima della review; le viste di diff o working set variano per harness e tipo di sessione. Prima di integrare, verificare lo stato reale del repository e richiedere correzioni mirate sulle righe problematiche.

## Privacy e configurazione

Le **istruzioni persistenti** descrivono convenzioni riutilizzabili del progetto. Nel Copilot harness di VS Code sono supportati, secondo la configurazione, `.github/copilot-instructions.md` o `AGENTS.md`; regole mirate possono usare file `*.instructions.md`. Discovery, scope e precedenza dipendono dall'harness: vedi [istruzioni persistenti](13-agents-instructions-skills.md#istruzioni-persistenti). Le indicazioni specifiche appartengono al [capitolo sull'AI (Artificial Intelligence) per Angular](../../guides/angular/12-ai-copilot-angular.md#contesto-e-istruzioni-del-progetto).

La **scelta del modello** bilancia qualità, latenza, contesto e costo; modelli disponibili e modalità dipendono da harness, account, piano e policy dell'organizzazione. Cambiare modello non cambia l'harness né concede nuovi tool. Le **approvazioni** decidono quali azioni richiedono conferma; il **sandbox** limita accesso a file e rete dove supportato. Controllare nome del tool, argomenti, percorso e destinazione prima di approvare scritture, comandi o accessi esterni: autorizzare una chiamata non significa approvare il risultato.

Una sessione può eseguire tool nel workspace locale, su una macchina remota o, dove disponibile, in un ambiente cloud; il modello può essere ospitato altrove anche quando i comandi girano localmente. Verificare **host, cartella, worktree e permessi** della sessione: file, estensioni e secrets locali non sono automaticamente presenti in cloud. Le sessioni cloud e alcune integrazioni remote possono essere sperimentali o soggette a disponibilità dell'account.

File e output dei tool possono contenere dati sensibili o istruzioni malevole. Non inviare secrets nel prompt; usare le policy di accesso e di esclusione del provider, verificandone i limiti. `.gitignore` e l'indice servono anche a filtrare la ricerca, ma non sono da soli una garanzia di riservatezza. Per repository sconosciuti, **Workspace Trust** in modalità ristretta impedisce l'uso degli agenti finché il workspace non è considerato attendibile.

## Limiti e fallimenti

Copilot può inventare API, confondere versioni, scegliere un file sbagliato o produrre test che confermano un'implementazione errata. Chiedere riferimenti a codice e documentazione, riprodurre i fallimenti e verificare in modo indipendente le affermazioni sensibili alla versione. Le istruzioni riducono l'ambiguità, ma non sostituiscono compilatore, test e review.

Quando una funzione manca o si comporta diversamente, controllare **versione di VS Code**, estensioni, harness selezionato, modello, piano/account, policy aziendali e posizione di esecuzione. Nomi delle modalità e disponibilità cambiano fra client: le indicazioni qui riguardano VS Code con GitHub Copilot e vanno confrontate con la documentazione della propria installazione.
