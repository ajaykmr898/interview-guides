# Architettura e DevOps

L'**architettura software** organizza responsabilità, dipendenze e confini di un sistema; **DevOps** integra sviluppo e operazioni per rendere verificabili, distribuibili e osservabili le modifiche. Le scelte dipendono da dimensioni del prodotto, team e costo del cambiamento.

## Indice

- [Architettura frontend](#architettura-frontend)
- [API e contratti](#api-e-contratti)
- [CI e CD](#ci-e-cd)
- [Container e deployment](#container-e-deployment)
- [Observability](#observability)
- [Qualità e tradeoff](#qualità-e-tradeoff)

## Architettura frontend

### Principi e dipendenze

**SOLID** raccoglie cinque principi per mantenere il codice modificabile; non prescrive classi o interfacce ovunque.

| Principio | Significato operativo |
| --- | --- |
| **S** — Single Responsibility | Un modulo ha un motivo principale di cambiamento, legato a una responsabilità del dominio. |
| **O** — Open/Closed | Estendere un comportamento tramite un punto di variazione stabile, senza modificare ogni chiamante. |
| **L** — Liskov Substitution | Un'implementazione sostitutiva rispetta il contratto osservabile del tipo base, incluse precondizioni ed effetti. |
| **I** — Interface Segregation | I client dipendono solo dalle operazioni che usano; evitare interfacce troppo ampie. |
| **D** — Dependency Inversion | La logica di alto livello dipende da astrazioni condivise, non da dettagli infrastrutturali concreti. |

**Dependency inversion** riguarda la direzione delle dipendenze; la **dependency injection** è un modo per fornire un'implementazione dall'esterno. Nell'esempio, `Checkout` conosce il contratto `OrderStore`, non il database:

```ts
type Order = { id: string };
interface OrderStore { save(order: Order): Promise<void>; }
class Checkout {
  constructor(private readonly store: OrderStore) {}
  complete(order: Order): Promise<void> { return this.store.save(order); }
}
```

- **DRY (Don't Repeat Yourself):** centralizzare una conoscenza o regola duplicata; codice simile con ragioni di cambiamento diverse può restare separato.
- **KISS (Keep It Simple, Stupid):** scegliere la soluzione comprensibile più semplice che soddisfa i requisiti, senza nascondere complessità reale.
- **YAGNI (You Aren't Gonna Need It):** rimandare flessibilità ipotetica finché non emerge un caso concreto.
- **Separation of concerns:** separare responsabilità come presentazione, dominio e accesso ai dati, definendo contratti espliciti fra i confini.

### Pattern e struttura

Un **design pattern** descrive una soluzione ricorrente e i suoi trade-off, non un'implementazione da applicare automaticamente.

| Pattern | Quando aiuta | Limite tipico |
| --- | --- | --- |
| **Factory** | Centralizza la creazione quando la variante dipende da input o configurazione. | Una factory per ogni oggetto aggiunge indirezione inutile. |
| **Strategy** | Sostituisce un algoritmo dietro un contratto stabile. | Troppe varianti complicano scelta e configurazione. |
| **Observer** | Notifica più subscriber quando cambia uno stato. | Ordine, lifecycle e cleanup dei listener vanno gestiti. |
| **Adapter** | Traduce l'interfaccia di una dipendenza in quella attesa dal client. | Non elimina differenze semantiche o errori della dipendenza. |
| **Facade** | Offre un ingresso semplice a un sottosistema complesso. | Può diventare un punto centrale eccessivamente ampio. |
| **Repository** | Incapsula query e persistenza dietro operazioni del dominio. | Può nascondere costi, transazioni e query inefficienti. |
| **Singleton** | Garantisce una sola istanza nell'ambito definito. | Stato globale implicito e test condivisi possono creare accoppiamento. |

**Composition vs inheritance:** la composizione assembla oggetti attraverso contratti e permette di cambiare collaboratori; l'ereditarietà riusa o specializza comportamento tramite una gerarchia e richiede un vero rapporto di sostituibilità. Preferire composizione quando le varianti evolvono indipendentemente; una gerarchia stabile può comunque essere adatta.

La **Clean Architecture** dispone le dipendenze verso regole di dominio più interne: interfacce ai confini separano casi d'uso da interfaccia utente, database e framework. Aiuta quando i confini cambiano, ma più livelli e mapping hanno un costo. Il **modular design** assegna a ogni modulo un contratto e nasconde dettagli interni; misurare coesione interna e dipendenze tra moduli, non il solo numero di cartelle.

| Organizzazione | Vantaggio | Rischio |
| --- | --- | --- |
| **Feature-based** | Raggruppa UI, stato e servizi di una funzionalità; facilita ownership e cambiamenti verticali. | Logica condivisa può essere duplicata o creare dipendenze circolari. |
| **Layer-based** | Raggruppa componenti per ruolo tecnico, utile per regole trasversali. | Una feature richiede modifiche in molti punti e confini di dominio deboli. |

**Code smells** come moduli molto grandi, dipendenze cicliche, parametri eccessivi o duplicazione segnalano punti da indagare: non dimostrano da soli un difetto. Refactoring mirato e test proteggono il comportamento mentre si migliora la struttura.

| Repository | Utile quando | Trade-off |
| --- | --- | --- |
| **Monorepo** | Più progetti condividono codice e cambiamenti coordinati in un unico repository. | CI, ownership e controllo dei confini richiedono disciplina. |
| **Multi-repo** | Team e cicli di rilascio sono indipendenti. | Versioni condivise e modifiche trasversali richiedono coordinamento tra repository. |

**Nx** è un tool per workspace multi-progetto: usa grafi di progetti e task per orchestrare build e test, selezionare lavoro interessato dalle modifiche e usare cache. Non sostituisce confini architetturali o ownership; input e output dei task vanno modellati correttamente. Per workspaces e [build tools](../node-tooling/09-build-tools.md#build-in-ci) vedi i capitoli dedicati.

I **microfrontend** dividono una UI in parti sviluppabili o distribuibili da team distinti, integrate a build o runtime. Possono dare autonomia organizzativa, ma aumentano costo di integrazione, versionamento, coerenza visiva, dipendenze condivise e osservabilità. Una singola applicazione modulare è spesso sufficiente se i team rilasciano insieme.

## API e contratti

Un'**API (Application Programming Interface)** è un contratto fra componenti: definisce operazioni, dati, errori e compatibilità. L'implementazione può cambiare senza rompere i consumer solo se mantiene il comportamento promesso; validare input ai confini e documentare casi di errore, non solo il percorso riuscito. Per HTTP (Hypertext Transfer Protocol), metodi, status e sicurezza sono trattati nel [capitolo Node](../node-tooling/08-node-npm.md#backend-http-ed-express).

La separazione frontend/backend richiede ownership del contratto, gestione delle versioni e migrazioni compatibili: aggiungere un campo opzionale è in genere più facile da distribuire di rinominare uno obbligatorio, ma ogni consumer va verificato. I **contract test** controllano che provider e consumer concordino su richieste e risposte rilevanti; non sostituiscono test dell'intero flusso o comportamento interno del provider.

## CI e CD

### Git e collaborazione

Un **branch** è un riferimento mobile a una linea di commit. Un **PR (Pull Request)** propone l'integrazione di modifiche e raccoglie review e controlli automatici; il merge resta una decisione sul codice e sul rischio. Commit piccoli e descrittivi facilitano review, revert e diagnosi. Una convenzione come `feat:`/`fix:` aiuta changelog e automazione solo se il team la applica con coerenza.

| Operazione Git | Effetto principale | Attenzione |
| --- | --- | --- |
| `merge` | Integra la storia di un branch, con fast-forward o commit di merge. | Può richiedere risoluzione di conflitti. |
| `rebase` | Riproduce commit su una nuova base, generando nuovi identificativi. | Evitare di riscrivere storia condivisa senza accordo. |
| `revert` | Crea un commit che annulla gli effetti di uno precedente. | È adatto a storia già pubblicata; può produrre conflitti. |
| `reset` | Con un commit sposta `HEAD` e, secondo la modalità, index e working tree; con path aggiorna solo l'index. | `--hard` può perdere modifiche locali. |
| `cherry-pick` | Applica su un altro branch la modifica di commit selezionati. | Crea commit nuovi; possibili conflitti o duplicati logici. |

**Merge vs rebase:** merge conserva la relazione fra storie; rebase costruisce una sequenza lineare riscrivendo i commit riprodotti. Entrambi possono richiedere risoluzione di conflitti; la scelta dipende dalla policy del team, soprattutto sui branch pubblicati.

### Pipeline e rilascio

**CI (Continuous Integration)** integra frequentemente modifiche e le verifica con build e test automatici. **CD** può significare **Continuous Delivery**, che mantiene un artefatto pronto al rilascio con decisione di deploy separata, oppure **Continuous Deployment**, che distribuisce automaticamente ogni modifica che supera i controlli. Chiarire quale significato usa il team.

Una **build pipeline** esegue passaggi ripetibili come installazione da lockfile, lint, analisi statica, test, build e pubblicazione dell'artefatto. I **quality gates** bloccano la promozione quando falliscono controlli richiesti; scegliere segnali affidabili, perché test flaky e gate lenti riducono il feedback. Costruire una volta e promuovere lo stesso artefatto fra ambienti riduce differenze accidentali. Le trasformazioni di build e i [controlli in CI](../node-tooling/09-build-tools.md#build-in-ci) sono approfonditi nel capitolo tooling.

## Container e deployment

Un'**image Docker** è un pacchetto immutabile di file e configurazione, spesso costruito da un `Dockerfile`; un **container** è un processo isolato avviato dall'image, con uno strato scrivibile proprio. I container condividono il kernel dell'ambiente che li esegue, che su Docker Desktop può essere una macchina virtuale; l'isolamento non è un confine di sicurezza assoluto. Persistenza, pubblicazione delle porte e accesso di rete vanno considerati esplicitamente; dati scritti solo nello strato del container possono sparire quando viene eliminato.

Gli **ambienti di esecuzione** (environment: sviluppo, staging, produzione) differiscono per configurazione e credenziali, non dovrebbero richiedere codice sorgente diverso. Passare configurazioni a runtime e gestire i secrets con un servizio dedicato e permessi minimi; variabili d'ambiente possono essere visibili a chi controlla il processo o il container. Evitare credenziali in image, repository e log.

Una **release** associa artefatto, configurazione, migrazioni e verifica post-deploy a una versione identificabile. **Rollback** ripristina una versione funzionante o interrompe un rollout, ma non annulla automaticamente dati già scritti o migrazioni incompatibili. Pianificare cambiamenti di schema compatibili fra versioni e controllare health check, errori e metriche dopo il deploy.

## Observability

L'**observability** rende interrogabile lo stato interno del sistema attraverso segnali emessi durante l'esecuzione:

| Segnale | Cosa mostra | Limite |
| --- | --- | --- |
| **Logging** | Eventi e contesto discreti, idealmente strutturati e con identificativo di correlazione. | Volume elevato e dati sensibili richiedono filtri e retention. |
| **Metrics** | Misure aggregate nel tempo, come latenza, errori e saturazione. | Non spiegano da sole la causa di una singola richiesta. |
| **Tracing** | Percorso di una richiesta attraverso servizi e operazioni, tramite trace e span. | Richiede propagazione del contesto; sampling può omettere casi rari. |
| **Error monitoring** | Raggruppa eccezioni e fallimenti con frequenza e contesto. | Un'eccezione catturata o un errore di business può sfuggire senza strumentazione. |

Correlare segnali e release per distinguere una regressione da un problema preesistente; allarmi su sintomi per l'utente sono più utili del solo numero di log. Non registrare password, token o dati personali non necessari.

**Application security** significa proteggere i confini di fiducia: validare input, autenticare identità, autorizzare per risorsa, limitare privilegi e mantenere secrets fuori dal codice. Un **dependency audit** confronta versioni installate con vulnerabilità note, ma non scopre ogni difetto né prova che una dipendenza sia sicura; valutare impatto, aggiornamenti e test. Per lockfile e [`npm audit`](../node-tooling/08-node-npm.md#versioni-e-sicurezza-delle-dipendenze) vedi il capitolo Node.

## Qualità e tradeoff

**Clean Code** privilegia nomi chiari, responsabilità circoscritte e comportamento leggibile; non impone una lunghezza universale per funzioni o classi. Il **refactoring** cambia la struttura preservando il comportamento osservabile: piccoli passi e test pertinenti riducono il rischio. In **code review** verificare correttezza, contratti, sicurezza, test e comprensibilità, separando difetti reali da preferenze stilistiche.

**Formatting** applica uno stile meccanico; **linting** segnala regole di codice e talvolta corregge problemi; **static analysis** esamina codice senza eseguirlo, per esempio tipi, flussi o pattern vulnerabili. I risultati possono avere falsi positivi o non vedere errori di runtime: la pipeline li integra con test e review.

| Test | Scopo | Limite principale |
| --- | --- | --- |
| **Unit** | Verifica una piccola unità con dipendenze controllate. | Non prova l'integrazione reale. |
| **Integration** | Verifica la collaborazione fra componenti o con infrastruttura reale. | Setup più costoso e possibili failure esterne. |
| **Component** | Verifica un componente applicativo nel suo confine, per esempio una UI con input e interazioni. | Non prova l'intero percorso utente. |
| **E2E (End-to-End)** | Attraversa l'app come un utente, inclusi più livelli. | Lento e sensibile ad ambiente e dati. |
| **Contract** | Verifica aspettative condivise fra consumer e provider. | Non prova la correttezza interna di entrambi. |
| **Smoke** | Controlla rapidamente funzioni essenziali dopo build o deploy. | Copertura intenzionalmente ridotta. |
| **Regression** | Controlla che un comportamento esistente non si sia rotto. | È uno scopo: può usare unit, integration o E2E. |

La **test pyramid** suggerisce molti test veloci e mirati alla base e meno test completi e costosi in alto; la proporzione dipende dai rischi del prodotto. Uno **stub** restituisce risposte predisposte, un **spy** registra chiamate, un **mock** definisce aspettative di interazione: usarli per isolare un confine, senza riscrivere nel test la stessa logica dell'implementazione. Testare anche errori, cleanup e contratti nei punti dove il costo del guasto è alto.
