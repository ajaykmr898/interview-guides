# MCP e agenti esterni

**MCP (Model Context Protocol)** è un protocollo per esporre a un'applicazione di intelligenza artificiale (AI) strumenti, dati e prompt tramite server collegati. Standardizza lo scambio di messaggi; scelta del modello, permessi ed esecuzione restano responsabilità dell'applicazione e dell'ambiente.

## Indice

- [Architettura MCP](#architettura-mcp)
- [Server tool resource e prompt](#server-tool-resource-e-prompt)
- [Transport e autorizzazione](#transport-e-autorizzazione)
- [Agenti esterni e integrazioni](#agenti-esterni-e-integrazioni)
- [Sicurezza e confini di fiducia](#sicurezza-e-confini-di-fiducia)
- [Diagnostica](#diagnostica)

## Architettura MCP

- **Host:** applicazione AI che coordina modello, utente e connessioni, per esempio un IDE (Integrated Development Environment) o una CLI (Command-Line Interface).
- **Client:** componente dell'host che comunica con un server MCP; l'host crea un client per ciascun server collegato.
- **Server:** processo locale o servizio remoto che pubblica capacità utilizzabili dall'host. Il server non è necessariamente un agente né decide autonomamente il flusso di lavoro.

MCP usa messaggi JSON-RPC (JSON, JavaScript Object Notation; RPC, Remote Procedure Call). Il client può scoprire le capacità e gli elementi esposti dal server, poi richiedere letture o chiamate. La revisione `2026-07-28` usa richieste senza sessione di protocollo, con versione e capacità del client in ogni richiesta e discovery opzionale tramite `server/discover`; le revisioni precedenti usano l'handshake `initialize`. Client e server devono supportare una revisione compatibile.

## Server tool resource e prompt

| Elemento | Funzione | Operazioni principali |
| --- | --- | --- |
| **Tool** | Operazione invocabile, per esempio cercare dati o modificare un ticket; può avere effetti esterni. | `tools/list`, `tools/call` |
| **Resource** | Dato identificato da URI (Uniform Resource Identifier), per esempio un file o uno schema; fornisce contesto. | `resources/list`, `resources/read` |
| **Prompt** | Template riutilizzabile di messaggi, eventualmente parametrico; guida un'interazione. | `prompts/list`, `prompts/get` |

Le liste permettono la discovery, ma quanto viene mostrato al modello e quando invocarlo dipende dal client. I tool sono tipicamente selezionati dal modello, le resource dall'applicazione e i prompt dall'utente; l'interfaccia concreta varia. Pubblicare un tool non autorizza ogni chiamata: il runtime deve verificare argomenti, permessi e risultato. Per il [tool calling](11-ai-fundamentals.md#tool-use-e-function-calling) in generale, vedi il capitolo introduttivo.

## Transport e autorizzazione

| Transport | Uso tipico | Confine da controllare |
| --- | --- | --- |
| **stdio** (standard input/output) | L'host avvia un processo locale e scambia messaggi sui suoi stream. | Il comando gira con i permessi dell'utente o della sandbox; controllare binario, dipendenze e variabili d'ambiente. |
| **Streamable HTTP** (Hypertext Transfer Protocol) | L'host chiama un endpoint, spesso remoto. | Verificare endpoint, connessione sicura, identità, token e destinazione dei dati. |

Un server locale può comunque accedere alla rete; un endpoint HTTP può risiedere sulla stessa macchina. **Autenticazione** identifica il chiamante, **autorizzazione** limita operazioni e dati: un login riuscito non concede automaticamente ogni tool. Un server HTTP può richiedere OAuth (framework di autorizzazione) e scope specifici; il server deve verificare il token e applicare i permessi per richiesta. Non salvare token in file condivisi né inoltrarli a server diversi. Un server fidato ha provenienza, codice, dipendenze e permessi verificati: il solo nome nel marketplace non basta.

**Esempio concettuale per il Copilot harness di VS Code**, in `.mcp.json` alla radice del workspace: configurazione del server, non implementazione del server. Il supporto della destinazione workspace dipende dalle impostazioni del client; l'endpoint illustrativo va sostituito con quello del fornitore.

```json
{
  "mcpServers": {
    "documentazione": {
      "type": "http",
      "url": "https://mcp.example.com/mcp",
      "tools": ["search"]
    }
  }
}
```

Nel Copilot harness questo formato usa `mcpServers`; la configurazione MCP gestita dall'editor VS Code in `.vscode/mcp.json` usa invece `servers`. La voce `tools` limita i nomi esposti, ma non sostituisce autorizzazione e approvazione. Per un server stdio, lo stesso formato usa `type: "stdio"`, `command` e, se necessari, `args` e `env`: installare il runtime nell'ambiente in cui gira l'host e passare i secrets tramite il meccanismo supportato dal client.

## Agenti esterni e integrazioni

| Componente | Responsabilità | Limite |
| --- | --- | --- |
| **Tool calling** | Il modello richiede un'operazione; il runtime la valida ed esegue. | Non definisce un catalogo interoperabile di server. |
| **Server MCP** | Espone tool, resource e prompt tramite un protocollo comune. | Non è una policy di sicurezza né una procedura di lavoro. |
| **Skill** | Istruzioni riutilizzabili, spesso in `SKILL.md`, per svolgere un compito. | Non fornisce da sola accesso a servizi o nuovi permessi. |
| **Agent plugin** | Pacchetto installabile che può distribuire skill, agenti, hook e connessioni MCP. | Manifest, componenti supportati e aggiornamenti dipendono dall'ecosistema. |

Un **agente custom** configura un ruolo riutilizzabile; un **subagent** è un'istanza delegata per un sottocompito. Possono usare tool MCP, ma non coincidono con il server che li espone. Definizioni, istruzioni e delega sono nel [capitolo sugli agenti](13-agents-instructions-skills.md#agenti-e-ciclo-di-lavoro).

- **GitHub Copilot** in VS Code e Copilot CLI condividono alcune configurazioni del Copilot harness; file, disponibilità e approvazioni dipendono dall'ambiente di esecuzione.
- **Claude Code** integra MCP da CLI e può distribuire server, skill, agenti e hook tramite plugin e marketplace propri.
- **OpenAI Codex** configura MCP nella CLI o nell'estensione IDE, con configurazione condivisa per queste superfici; usa convenzioni proprie per configurazione e plugin.

**Portabilità:** il protocollo rende interoperabile lo scambio con un server compatibile, non i manifest o il comportamento dei client. `AGENTS.md` e `SKILL.md` possono trasportare istruzioni e workflow dove supportati, ma discovery, precedenza, tool disponibili, sandbox e approvazioni cambiano tra prodotti; un file presente nel repository non è necessariamente letto da ogni client. Anche i formati MCP differiscono: Copilot harness usa `.mcp.json` con `mcpServers`, Codex usa `config.toml`, Claude Code accetta comandi CLI e configurazioni JSON. Verificare nel client di destinazione path, schema, revisioni MCP e supporto delle capacità; non copiare configurazioni alla cieca.

**Distribuzione:** una skill può essere installata singolarmente, anche da un marketplace di skill, o inclusa in un plugin; un marketplace è un catalogo, non una garanzia di sicurezza. Prima dell'installazione controllare autore e repository, versione fissata o politica di aggiornamento, manifest, script, dipendenze, server e permessi richiesti. Riesaminare le modifiche agli aggiornamenti: cambiare versione può cambiare sia istruzioni sia codice eseguibile. Vedi le [skill riutilizzabili](13-agents-instructions-skills.md#skill-riutilizzabili) per la struttura di base.

## Sicurezza e confini di fiducia

File, pagine web, resource, prompt e output di tool provenienti da terzi sono **dati a fiducia inferiore**: possono contenere istruzioni ostili. Una [prompt injection](11-ai-fundamentals.md#privacy-e-sicurezza) tenta di farle eseguire come comandi, per esempio esfiltrando secrets o indirizzando l'agente verso un altro endpoint. Un prompt fornito da un server MCP non acquista la stessa autorità delle istruzioni dell'utente o del progetto.

**Checklist prima di abilitare un'integrazione:**

- Verificare publisher, sorgente, versione e dipendenze del server o plugin; ispezionare script di installazione e aggiornamenti.
- Limitare filesystem, rete, account, scope OAuth e tool esposti al minimo necessario; separare credenziali personali e di progetto.
- Controllare comando stdio o URL (Uniform Resource Locator) remoto e destinazione dei dati; evitare secrets in chat, repository, argomenti e log.
- Riesaminare argomenti e risultati delle chiamate con effetti esterni; richiedere approvazione umana dove il client la offre. Un'approvazione non rende affidabile il contenuto restituito.
- Verificare hook e script dei plugin: possono eseguire comandi in eventi del ciclo di vita anche senza una nuova chiamata esplicita del modello. Usare sandbox e restrizioni del runtime, senza presumere che siano identiche tra client.

I controlli del server su identità e privilegi restano necessari anche se l'host applica sandbox o approvazioni; nessuno di questi livelli sostituisce gli altri. La [supervisione degli agenti](13-agents-instructions-skills.md#delegazione-e-supervisione) copre i passaggi di review e responsabilità.

## Diagnostica

1. **Connessione:** verificare transport, endpoint o comando, runtime e versione MCP supportata; per stdio leggere gli errori del processo, per HTTP stato e risposta del server.
2. **Discovery:** controllare che il client veda server, capacità e liste (`tools/list`, `resources/list`, `prompts/list`). Una lista vuota può dipendere da scope, configurazione o tool disabilitati.
3. **Autorizzazione:** distinguere credenziali mancanti o scadute, scope insufficienti, policy dell'organizzazione e approvazione locale negata.
4. **Esecuzione:** invocare un tool innocuo e verificare schema degli argomenti, risultato ed errori nei log del client e del server. Un tool scoperto ma non scelto dal modello non è necessariamente guasto.
5. **Portabilità:** ripetere i controlli nel client e nell'ambiente effettivi (locale, container o remoto); configurazione, secrets e binari non si trasferiscono automaticamente.
