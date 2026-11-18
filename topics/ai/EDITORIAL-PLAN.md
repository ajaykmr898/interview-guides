# Piano editoriale — AI Development

Requisiti specifici dei capitoli di questa directory. Per le regole editoriali comuni e i criteri di accettazione, seguire [AGENTS.md](../../AGENTS.md).

## Ambito e compatibilità

Questi capitoli trattano i concetti AI generali: LLM e context engineering, VS Code/Copilot, istruzioni e agenti, Agent Skills e `SKILL.md`, agenti esterni, MCP, plugin e marketplace, sicurezza, permessi e prompt injection. I workflow specifici dei framework appartengono ai rispettivi piani locali.

Per i formati di configurazione di agenti, skill, plugin e MCP, specificare client, scope e compatibilità; non presumere portabilità tra strumenti.

## `11-ai-fundamentals.md` — AI Fundamentals

**Da coprire:**
- AI vs ML vs Generative AI; LLM, tokens, tokenization e context window.
- Prompt, system/developer/user instructions, prompt engineering vs context engineering.
- Hallucinations, grounding, factuality e limiti dei modelli.
- Temperature, model selection, latenza, token budget e costi (senza numeri tariffari non verificati).
- Structured outputs, function/tool calling e gestione delle risposte non affidabili.
- RAG, embeddings, vector databases, retrieval, chunking, ranking (overview).
- Agentic AI: agent loop, planning, act/observe/evaluate, tool use, human-in-the-loop.
- Agent vs chatbot vs workflow deterministico; multi-agent systems come concetto.
- Prompt injection, data leakage, permissions, secrets, guardrails, valutazione della qualità.

**Confronti/snippet:** prompt vs context, RAG vs fine-tuning (overview), agent vs workflow, tool calling vs MCP.

## `12-copilot-vscode.md` — VS Code e GitHub Copilot

**Da coprire:**
- VS Code workspace, editor context, files/indexing e limiti del contesto.
- GitHub Copilot: inline suggestions, Chat, agent workflows, planning, workspace awareness.
- Code generation, refactoring, debug, tests, review, terminal/CLI tools.
- Model selection, tool approvals, permissions, local vs remote execution dove supportati.
- Persistent instructions e configurazione del workspace: panoramica, link al file sulle instructions.
- Prompt/context engineering per cambi multi-file, task piccoli e review del diff.
- Versioni e disponibilità dei comportamenti: verificare la documentazione dello specifico client e harness.
- Best practice per evitare modifiche fuori scope, regressioni e codice inventato.

**Confronti:** completamento inline vs chat agentica; planning vs editing; suggerimento vs azione autorizzata.

## `13-agents-instructions-skills.md` — Agents, Instructions e Skills

**Da coprire:**
- Agent, custom agent, subagent, delegation, orchestrazione, scoping e human oversight.
- Instructions vs agent definitions vs skill vs reusable prompt; quando usare ciascuno.
- `AGENTS.md`, `copilot-instructions.md`, `*.instructions.md`, `*.agent.md`, `SKILL.md`, `*.prompt.md`.
- Per ogni file: scopo, cartella possibile, frontmatter YAML (se applicabile), modalità di discovery/attivazione, precedenza/scope e compatibilità **client-specific**.
- Skill structure: `SKILL.md`, references, scripts, templates, progressive disclosure e invocation.
- Skills locali/project/user/global, skills di terze parti, aggiornamenti e trust.
- Agent workflows: task breakdown, context compaction, memory/context limits, evaluation e rollback.
- Configuration examples minimalisti e corretti; non dare per universale un path supportato da un solo client.

**Confronti/snippet:** instructions vs skills vs agents; esempio minimale di `AGENTS.md`, frontmatter skill, agent definition (verificare formato corrente).

## `14-mcp-external-agents.md` — MCP ed Ecosistema degli Agents

**Da coprire:**
- MCP: protocollo, host/client/server, tools, resources, prompts, discovery e transport (overview).
- Local vs remote MCP, autenticazione, autorizzazione, scope, secrets e trusted servers.
- MCP vs tool calling vs skill vs agent plugin: ruoli, composizione e limiti.
- Agenti esterni e ambienti: GitHub Copilot, Claude Code, OpenAI Codex, CLI e client differenti (senza assumere compatibilità completa).
- Agent plugins, skills marketplaces, installazione/distribuzione/versioning e verifica della provenienza.
- Cross-tool portability, `AGENTS.md`, `SKILL.md`, manifest/config differenti e convenzioni specifiche.
- Hook, tool execution, sandbox, approvals, prompt injection, supply-chain risks, exfiltration e review.
- Esempi concettuali di configurazione MCP e checklist di security, verificati sul client specifico.

**Confronti:** MCP server vs skill, custom agent vs subagent, local vs remote tools, trusted vs untrusted instructions.
