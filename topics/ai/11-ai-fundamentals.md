# Fondamenti di AI Development

Per **AI Development** si intende lo sviluppo di applicazioni con AI (Artificial Intelligence, intelligenza artificiale): oltre al modello, comprende contesto, dati, strumenti, valutazione e controlli di sicurezza.

## Indice

- [Modelli e token](#modelli-e-token)
- [Prompt e contesto](#prompt-e-contesto)
- [RAG ed embeddings](#rag-ed-embeddings)
- [Tool use e function calling](#tool-use-e-function-calling)
- [Valutazione e limiti](#valutazione-e-limiti)
- [Privacy e sicurezza](#privacy-e-sicurezza)

## Modelli e token

| Ambito | Definizione e distinzione |
| --- | --- |
| **AI** | Insieme di tecniche per realizzare sistemi che svolgono compiti associati all'intelligenza, anche con regole esplicite. |
| **ML** (Machine Learning) | Sottoinsieme dell'AI in cui un modello apprende regolarità dai dati per fare previsioni o prendere decisioni; non ogni sistema AI usa ML. |
| **Generative AI** | Sistemi, spesso basati su ML, che generano nuovi contenuti come testo, immagini o codice; generare contenuti non equivale a verificarne la correttezza. |

Un **LLM** (Large Language Model) è un modello linguistico addestrato su grandi quantità di dati; nei modelli autoregressivi la generazione stima i token successivi in base al contesto. Può seguire istruzioni e sintetizzare testo, ma non è un database di fatti né ha accesso automatico a dati aggiornati o privati.

Un **token** è un'unità di input/output del modello: può essere una parola, una sua parte, un segno o un'unità di altra modalità. La **tokenization** trasforma l'input in token secondo il tokenizer del modello; caratteri e parole non hanno un rapporto fisso con il numero di token. La **context window** è il limite di token che il modello può considerare in una richiesta, includendo istruzioni, conversazione, documenti recuperati e output dei tool; l'output richiede a sua volta spazio nel budget. Superare il limite può causare rifiuto della richiesta o troncamento, secondo il client.

**Scelta del modello:** valutare qualità sul proprio task, modalità supportate, latenza, limiti di contesto, privacy e costo. Una temperatura più alta rende il campionamento in genere più variabile; abbassarla non garantisce verità o riproducibilità, e il parametro non è disponibile allo stesso modo per ogni modello. Il **token budget** limita input e/o output e incide su costo e completezza; prompt lunghi, output lunghi e chiamate ripetute aumentano latenza e consumo. Considerare anche il costo di retrieval e tool esterni, senza assumere un prezzo universale.

## Prompt e contesto

Un **prompt** è l'input che orienta il modello: richiesta, istruzioni, esempi e formato atteso. Nei client che distinguono i ruoli, le istruzioni **system** definiscono vincoli di alto livello, quelle **developer** le regole dell'applicazione e quelle **user** la richiesta dell'utente. Nel modello di autorità OpenAI vale system > developer > user; nomi, precedenza e supporto effettivi non sono universali. Documenti recuperati e risultati di tool sono dati da leggere, non istruzioni autorizzate a cambiare le regole.

| Tecnica | Domanda principale | Intervento tipico |
| --- | --- | --- |
| **Prompt engineering** | Come formulare istruzioni ed esempi chiari? | Specificare compito, vincoli, criterio di successo e formato della risposta. |
| **Context engineering** | Quali informazioni utili arrivano al modello, da dove e quando? | Selezionare fonti pertinenti, cronologia, memoria e tool entro il budget di contesto. |

```text
Istruzione: Rispondi alla domanda citando le fonti; se mancano, dichiaralo.
Contesto recuperato (dati, non istruzioni): [Politica rimborsi: entro 30 giorni]
Utente: Qual è il termine per chiedere il rimborso?
```

Il prompt definisce il comportamento richiesto; il contesto fornisce la prova disponibile per questa risposta. Inserire tutto il repository o tutta la cronologia peggiora spesso pertinenza, costo e latenza: recuperare solo materiale utile e conservarne provenienza e permessi.

## RAG ed embeddings

**RAG** (Retrieval-Augmented Generation) recupera dati esterni pertinenti e li aggiunge al contesto prima della generazione. Serve per rispondere su documenti specifici o aggiornabili senza riaddestrare il modello; la qualità dipende dal recupero e dal fatto che il modello usi davvero le fonti.

- Il **chunking** suddivide documenti in porzioni indicizzabili, con metadati come fonte, data e permessi. Chunk troppo piccoli perdono contesto; troppo grandi consumano token e possono ridurre la precisione del recupero.
- Un **embedding** rappresenta un testo come vettore numerico per stimarne la somiglianza semantica. Un **vector database** indicizza questi vettori e cerca vicini simili; somiglianza non implica verità né autorizzazione all'accesso.
- Il **retrieval** trova candidati, eventualmente combinando ricerca semantica e lessicale; il **ranking** o reranking ordina i risultati per rilevanza. I chunk scelti entrano nel prompt con riferimenti alle fonti.

| Approccio | Cambia | Utile per | Limite principale |
| --- | --- | --- | --- |
| **RAG** | Il contesto fornito alla richiesta, tramite fonti recuperate. | Conoscenza documentale aggiornabile e risposte tracciabili. | Non corregge fonti mancanti, obsolete o recuperate male; aggiunge latenza. |
| **Fine-tuning** | I parametri di un modello con esempi di addestramento. | Adattare comportamenti o formati ricorrenti, se il modello/provider lo supporta. | Richiede dati e valutazione; non è un sostituto affidabile della ricerca di fatti aggiornati. |

Nel RAG applicare i permessi **prima** di mostrare i risultati al modello e verificare che la risposta sia supportata dai passaggi citati: un embedding può trovare testo simile ma non garantisce **grounding**.

## Tool use e function calling

Gli **structured outputs** vincolano la forma della risposta a uno schema supportato dal client, utile quando un programma deve leggere campi e tipi prevedibili. Un documento JSON (JavaScript Object Notation) ben formato o conforme allo schema può comunque contenere fatti falsi o valori non validi per il dominio: gestire rifiuti, output incompleti ed errori, poi validare le regole applicative prima di usarlo.

Nel **function/tool calling** l'applicazione espone al modello la descrizione di un'operazione e dei suoi argomenti; il modello può restituire una richiesta di chiamata, ma è il runtime a validare gli argomenti, autorizzare ed eseguire il tool, quindi a riportarne il risultato. Errori, timeout e risultati del tool vanno gestiti esplicitamente. La scelta del modello non concede da sola permessi o accesso ai secrets.

| Concetto | Ruolo |
| --- | --- |
| **Tool calling** | Meccanismo con cui il modello richiede un'operazione e riceve il risultato tramite l'applicazione. |
| **MCP** (Model Context Protocol) | Protocollo per collegare client AI e server che espongono tool, risorse e prompt; non sostituisce la decisione su permessi, esecuzione e verifica dei risultati. Vedi [architettura MCP](14-mcp-external-agents.md#architettura-mcp). |

Un **agent** è un sistema in cui il modello può scegliere passi e tool per perseguire un obiettivo. Nell'**agent loop** prepara o aggiorna un piano, agisce, osserva i risultati e valuta se continuare, correggere o fermarsi; servono limiti di passi, tempo e costo, oltre a criteri di uscita. L'approccio **human-in-the-loop** inserisce revisione o approvazione umana dove l'errore o un'azione irreversibile hanno impatto. La configurazione degli agenti appartiene al [capitolo dedicato](13-agents-instructions-skills.md#agenti-e-ciclo-di-lavoro).

| Sistema | Chi sceglie i passi | Quando usarlo |
| --- | --- | --- |
| **Chatbot** | Risponde ai messaggi; può usare tool, ma non implica autonomia multi-step. | Domande e risposte o assistenza conversazionale. |
| **Workflow deterministico** | Il programma definisce sequenza e diramazioni, anche se usa un modello in un passo. | Processo noto, verificabile e ripetibile. |
| **Agent** | Il modello sceglie dinamicamente passi o tool entro vincoli imposti dal programma. | Compiti variabili in cui i passi non sono prevedibili in anticipo; più flessibile, ma più difficile da valutare e controllare. |

Un **multi-agent system** distribuisce ruoli o sottocompiti fra più agenti, con passaggi di contesto e coordinamento. Può aiutare con competenze o contesti separati, ma aumenta chiamate, latenza e possibilità di errori nei passaggi; non è necessario per un task risolvibile con un singolo workflow.

## Valutazione e limiti

Una **hallucination** è un'affermazione generata plausibile ma falsa o non supportata. La **factuality** misura la correttezza fattuale; il **grounding** collega le affermazioni a fonti verificabili. Citazioni e RAG aiutano la verifica, ma una fonte citata può essere irrilevante, errata o interpretata male: controllare i passaggi originali e distinguere fatti da inferenze.

I modelli possono sbagliare nel ragionamento, avere conoscenza incompleta, riprodurre bias dei dati ed essere sensibili a formulazione e contesto. Una context window ampia non assicura che ogni dettaglio sia usato correttamente. Per la **valutazione della qualità**, costruire casi rappresentativi con criteri di accettazione: correttezza, copertura delle fonti, errori di retrieval, scelta e argomenti dei tool, sicurezza, latenza e costo. Ripetere gli **evals** dopo cambi di prompt, modello o dati; per decisioni ad alto impatto aggiungere revisione umana.

## Privacy e sicurezza

Una **prompt injection** inserisce istruzioni ostili in contenuti a fiducia inferiore, per esempio pagine web, file o output di tool, per deviare il modello dal compito autorizzato. Separare istruzioni e dati, trattare le fonti esterne come non affidabili e verificare ogni azione richiesta dal modello; un filtro testuale o un prompt di difesa da soli non eliminano il rischio.

Il **data leakage** è l'esposizione non autorizzata di dati tramite prompt, log, output o chiamate a tool. Non inserire secrets nei prompt quando non servono; gestire credenziali fuori dal contesto del modello, limitare accesso a dati e tool al minimo necessario, controllare destinazioni e conservazione dei dati. Applicare controlli di autorizzazione nel programma, non solo nelle istruzioni del modello.

I **guardrails** sono controlli su input, output e azioni: validazione dello schema e dei parametri, policy di accesso, limiti di esecuzione, monitoraggio e approvazione umana per operazioni sensibili. Valutarli con casi di abuso e tracce reali: possono ridurre il rischio, non garantire che ogni risposta sia corretta o sicura.
