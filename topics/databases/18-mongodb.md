# MongoDB

MongoDB è un database documentale: conserva documenti BSON (Binary JSON) in collection, invece di righe con colonne fisse. Il modello e le garanzie di lettura/scrittura dipendono da query, schema, topologia e configurazione. Gli esempi usano MongoDB Server, `mongosh` e il driver Node.js ufficiale; per il modello relazionale vedi [MySQL](17-mysql.md#modello-relazionale-e-schema).

## Indice

- [Modello documentale e schema](#modello-documentale-e-schema)
- [Letture e scritture](#letture-e-scritture)
- [Indici e piani di esecuzione](#indici-e-piani-di-esecuzione)
- [Atomicità e consistenza](#atomicità-e-consistenza)
- [Operatività e scalabilità](#operatività-e-scalabilità)

## Modello documentale e schema

Un **database** contiene collection; una **collection** contiene documenti con campi e valori BSON. I documenti della stessa collection possono avere strutture diverse, ma un modello coerente resta necessario per query, indici e invarianti.

### Documenti, BSON e identità

BSON è una rappresentazione binaria con tipi non presenti nel JSON (JavaScript Object Notation) ordinario, fra cui `Date`, `ObjectId` e `Decimal128`. Usare il tipo corretto nel driver: una data serializzata come stringa e un importo salvato come floating point non hanno automaticamente semantica di data o decimale esatto. `Decimal128` è adatto quando serve precisione decimale, con conversioni esplicite ai confini applicativi.

Ogni documento di una collection ordinaria ha un `_id`, con indice unique creato per default; nelle collection sharded con `_id` diverso dalla shard key, l'unicità è applicata per shard. Se manca, driver o server possono generare un `ObjectId`; è possibile usare anche un altro identificatore BSON valido. L'identità tecnica non sostituisce una chiave di dominio. Il limite di un documento BSON è **16 mebibyte (MiB)**: grandi array in crescita vanno spesso modellati come documenti separati.

Esempio in `mongosh`, per una collection `todos`:

```js
db.todos.insertOne({
  listId: ObjectId('507f1f77bcf86cd799439011'),
  ownerId: 'u7', title: 'Ripassare indici',
  completed: false, labels: ['database'],
  createdAt: ISODate('2026-11-19T09:00:00Z')
});
```

Un campo **assente** non è un campo con valore `null`: il filtro `{ note: null }` trova entrambi; `{ note: { $exists: false } }` trova solo gli assenti e `{ note: { $type: 'null' } }` solo i `null` espliciti. Definire quale stato rappresenta ciascuno, invece di trattarli come equivalenti nel codice.

### Embedding, riferimenti e validazione

**Embedding** inserisce dati correlati nello stesso documento, utile se vengono letti e modificati insieme e hanno cardinalità limitata; la scrittura del documento resta atomica. Un **riferimento** conserva l'identificatore di un altro documento, utile per relazioni molti-a-molti, entità indipendenti o collezioni che crescono senza limite. Per esempio, impostazioni piccole possono stare nella `list`, mentre i `todos` crescenti hanno `listId` e una collection propria. Il riferimento non è una foreign key applicata dal server: cancellazioni e integrità vanno gestite esplicitamente. `$lookup` può combinare collection, ma aggiunge lavoro rispetto a una lettura incorporata.

Duplicare campi per evitare letture multiple può accelerare query frequenti, ma ogni modifica richiede una regola di sincronizzazione. MongoDB non richiede campi uniformi per default; la **schema validation** della collection, anche tramite `$jsonSchema`, può vincolare tipi e valori in ingresso. Non sostituisce la validazione del caso d'uso e va evoluta insieme ai documenti esistenti e alle versioni dell'applicazione.

## Letture e scritture

Le operazioni CRUD (Create, Read, Update, Delete) prendono filtri documentali. Il driver traduce richieste e risultati; il server applica filtri, modifiche, vincoli e indici.

### Filtri, proiezioni e CRUD

`findOne(filter)` restituisce una Promise del documento o `null`; `find(filter)` restituisce un cursor, da iterare o convertire con `toArray()` (Promise di un array). Un filtro può usare campi annidati tramite dot notation e cercare un elemento di array, per esempio `{ labels: 'database' }`. Una **proiezione** limita i campi restituiti; `_id` resta incluso salvo esclusione esplicita. `sort` e `limit` controllano ordine e quantità, ma `limit` senza ordinamento non definisce una pagina stabile.

Frammento con il driver Node.js `mongodb`: `client` è un `MongoClient` connesso e riusato dall'applicazione; `ownerId` e `title` sono già validati. L'esempio crea, legge, modifica ed elimina un todo:

```js
async function crud(client, ownerId, title) {
  const todos = client.db('app').collection('todos');
  const { insertedId } = await todos.insertOne({ ownerId, title, completed: false, createdAt: new Date() });
  const page = await todos.find({ ownerId }).project({ title: 1, completed: 1 })
    .sort({ createdAt: -1, _id: -1 }).limit(10).toArray();
  const update = await todos.updateOne({ _id: insertedId, ownerId }, { $set: { completed: true } });
  const deletion = await todos.deleteOne({ _id: insertedId, ownerId });
  return { page, update, deletion };
}
```

`insertOne` risolve con `insertedId`; `updateOne` con `matchedCount` e `modifiedCount` (una corrispondenza può non cambiare il valore); `deleteOne` con `deletedCount`. `updateMany`/`deleteMany` agiscono su tutti i documenti corrispondenti: evitare filtri vuoti accidentali. `$set` modifica campi, `$inc` incrementa un valore numerico; `replaceOne` sostituisce invece il documento, salvo `_id`. Con `{ upsert: true }`, un `updateOne` senza corrispondenza inserisce un documento e riporta `upsertedId`: usare un filtro e vincoli unici coerenti con l'identità richiesta.

Non passare direttamente oggetti di richiesta a filtri o update: chiavi e operatori come `$gt` o `$set` cambiano la semantica della query. Costruire campi e operatori consentiti e verificare l'autorizzazione sul documento; il driver non la deduce dall'input. Per [contratti ed errori HTTP (Hypertext Transfer Protocol)](../api/16-restful-apis.md#rappresentazioni-errori-e-collezioni) vedi il capitolo dedicato.

### Aggregation pipeline

Una **aggregation pipeline** passa documenti fra stage: `$match` filtra, `$project` seleziona o calcola campi, `$group` aggrega per chiave. Qui si contano i todo completati e incompleti di un utente:

```js
db.todos.aggregate([
  { $match: { ownerId: 'u7' } },
  { $group: { _id: '$completed', count: { $sum: 1 } } },
  { $project: { _id: 0, completed: '$_id', count: 1 } }
]);
```

`aggregate()` produce un cursor, non l'array finale già materializzato. `$lookup` esegue un join verso un'altra collection quando un riferimento serve nella risposta; su molti documenti può costare più di un modello adatto agli accessi frequenti. Ridurre presto i dati con filtri indicizzabili aiuta, ma l'ottimizzatore può riordinare alcuni stage: verificare il piano invece di affidarsi solo all'ordine scritto.

## Indici e piani di esecuzione

Un indice ordina chiavi e riferimenti ai documenti per accelerare filtri e sort; occupa spazio e aumenta il lavoro di ogni scrittura. Va scelto sulle query effettive.

### Indici singoli, composti e multikey

L'indice unico su `_id` esiste per default nelle collection ordinarie. Un indice singolo serve un campo; uno **composto** serve combinazioni secondo l'ordine delle chiavi. Per `{ ownerId: 1, createdAt: -1, _id: -1 }`, il prefisso `ownerId` può filtrare e i campi successivi sostenere l'ordinamento dei todo di quell'utente. Un indice solo su `createdAt` non offre lo stesso filtro, mentre un indice solo su `ownerId` può richiedere un sort separato. Anche selettività, direzione del sort e piano scelto contano; non ogni filtro su un campo indicizzato usa l'indice.

Un indice su un campo array come `labels` diventa **multikey** e crea chiavi per i suoi valori. Un indice **unique** impedisce duplicati della chiave indicizzata fra documenti, ma non equivale a una foreign key; in un indice unique su un solo campo, documenti con campo assente o `null` condividono la chiave `null`, quindi solo uno è ammesso. Indici ridondanti costano spazio e scritture: misurare prima di aggiungerli.

### explain e paginazione

`explain('executionStats')` mostra il piano scelto e statistiche osservate: `COLLSCAN` è una scansione della collection, `IXSCAN` usa un indice; confrontare documenti e chiavi esaminati con `nReturned` e controllare eventuale sort. Una scansione completa può essere ragionevole per collection piccole o filtri poco selettivi.

```js
db.todos.createIndex({ ownerId: 1, createdAt: -1, _id: -1 });
db.todos.find({ ownerId: 'u7' })
  .sort({ createdAt: -1, _id: -1 }).limit(10)
  .explain('executionStats');
```

`skip(n)` deve attraversare risultati precedenti e diventa costoso con offset alti. Una pagina basata sull'ultima coppia ordinata `(createdAt, _id)` può evitare offset crescenti, se filtro e indice la sostengono; `_id` risolve i pari timestamp. Il [contratto di paginazione HTTP](../api/16-restful-apis.md#rappresentazioni-errori-e-collezioni) resta separato dalla strategia di query.

## Atomicità e consistenza

**Atomicità** indica che una modifica applica tutti o nessuno dei suoi effetti nell'unità garantita. La visibilità e la durabilità delle scritture dipendono anche da read concern, write concern e replica.

### Scritture atomiche e transazioni

Una scrittura su **un documento** è atomica anche se cambia più campi. `updateMany` modifica ogni documento atomicamente, ma l'operazione nel suo complesso non è una transazione: altri accessi possono intercalarsi e un errore può lasciare modifiche precedenti. Per evitare lost update, includere nel filtro la versione o il valore atteso e verificare `matchedCount`; `$inc` evita il ciclo lettura-calcolo-scrittura per un contatore sullo stesso documento.

Quando un'invariante richiede più documenti, una transazione usa una **sessione**; costa più di una scrittura singola e non corregge un modello mal scelto. Esempio: aggiungere un todo e incrementare `todoCount` della lista esistente nello stesso database. `todo` contiene dati validati, `listId` è un `ObjectId` e `client` è un `MongoClient` connesso a un replica set o cluster compatibile:

```js
async function addTodoAndCount(client, todo, listId) {
  const session = client.startSession();
  try {
    return await session.withTransaction(async () => {
      const db = client.db('app');
      const { insertedId } = await db.collection('todos').insertOne({ ...todo, listId }, { session });
      const result = await db.collection('lists').updateOne(
        { _id: listId }, { $inc: { todoCount: 1 } }, { session });
      if (result.matchedCount !== 1) throw new Error('Lista assente');
      return insertedId;
    });
  } finally { await session.endSession(); }
}
```

`withTransaction` conferma al termine del callback e abortisce in caso di errore; può ritentare callback o commit per errori transitori. Tenere il callback privo di effetti esterni non idempotenti e non eseguire operazioni parallele sulla stessa sessione. Un errore di commit può lasciare esito incerto: non ripetere ciecamente l'operazione applicativa. Le **retryable writes** del driver riguardano singole operazioni ammesse; non sono un retry automatico arbitrario di tutto il caso d'uso.

### Read concern, write concern e read preference

| Impostazione | Cosa controlla | Distinzione essenziale |
| --- | --- | --- |
| **Write concern** | Quando una scrittura è confermata al client. | `w: 1` attende il primary; `w: 'majority'` attende una maggioranza idonea. Una scrittura con sola conferma del primary può essere rollbackata dopo un failover. |
| **Read concern** | Quali dati sono ammissibili alla lettura. | `local` può leggere dati non confermati dalla maggioranza; `majority` legge dati confermati dalla maggioranza, ma non promette da solo il valore più recente. |
| **Read preference** | A quale membro indirizzare la lettura. | `primary` è la scelta predefinita; un secondary può essere indietro per replica lag, indipendentemente dal read concern. |

Per *read-after-write* leggere dal primary quando serve la propria modifica appena confermata; dopo failover contano anche write concern e possibili rollback. Se la lettura deve attraversare membri diversi con garanzie causali, usare una sessione causalmente consistente con read e write concern `majority`. Le garanzie dipendono dalla combinazione, non da un singolo flag.

## Operatività e scalabilità

Scelte di connessione, sicurezza, rollout e topologia determinano affidabilità e costo del database in esercizio. Separare responsabilità del server da quelle del driver e del servizio gestito.

### Connessioni, sicurezza ed evoluzione dello schema

Il `MongoClient` del driver gestisce pool e selezione dei server: riusarlo tra richieste, dimensionarlo rispetto al carico e chiuderlo allo shutdown. Timeout, errori di rete e write concern fallito richiedono gestione esplicita; una risposta persa non prova che la scrittura non sia avvenuta. Proteggere credenziali, rete e connessioni con autenticazione, ruoli a privilegi minimi e TLS (Transport Layer Security); validare input e operatori ammessi prima di costruire filtri e update.

Lo schema flessibile non elimina le migrazioni: aggiungere campi e validazioni compatibili con vecchie versioni dell'app, eseguire backfill misurati, poi rendere obbligatori i nuovi requisiti. Pianificare la creazione degli indici per spazio e carico. In un servizio gestito parte di backup, patching e monitoraggio è fornita dal provider; verificare comunque configurazione e restore. Per [pattern Repository e deployment](../engineering/15-architecture-devops.md#pattern-e-struttura) vedi Engineering.

### Replica, sharding e ripristino

Un **replica set** mantiene un primary per le scritture e secondary che applicano asincronamente l'**oplog** delle operazioni. Un'elezione può scegliere un nuovo primary quando il precedente non è disponibile; c'è un intervallo di indisponibilità e il lag può rendere stale le letture dai secondary. Monitorare latenza, errori, pool, query lente e lag; un failover non equivale a zero perdita di dati per ogni write concern.

Backup coerenti e restore provati servono per errori umani e corruzione; il ripristino a una data desiderata richiede anche le modifiche successive allo snapshot. Una replica può propagare una cancellazione e non è un backup indipendente. La procedura dipende da topologia e strumenti self-managed o gestiti. Per i principi di [observability](../engineering/15-architecture-devops.md#observability) vedi Engineering.

Lo **sharding** distribuisce collection fra shard per scalare capacità e volume. La *shard key* determina collocazione e routing: una distribuzione sbilanciata o una chiave crescente può concentrare scritture su pochi shard; query senza chiave adatta possono coinvolgerne molti. Aggiunge costo operativo e non sostituisce indici, replica o una modellazione delle query.
