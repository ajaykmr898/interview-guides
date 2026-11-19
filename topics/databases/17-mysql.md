# MySQL

MySQL è un sistema di gestione di database relazionali che usa SQL (Structured Query Language) per definire, leggere e modificare dati. Qui il riferimento è MySQL 8.4 con lo storage engine InnoDB; controllare versione e `sql_mode` dell'installazione prima di applicare i default descritti.

## Indice

- [Modello relazionale e schema](#modello-relazionale-e-schema)
- [Query e modifiche](#query-e-modifiche)
- [Indici e piani di esecuzione](#indici-e-piani-di-esecuzione)
- [Transazioni e concorrenza](#transazioni-e-concorrenza)
- [Operatività ed evoluzione](#operatività-ed-evoluzione)

## Modello relazionale e schema

Un database organizza dati in tabelle con righe e colonne; in MySQL `DATABASE` e `SCHEMA` sono sinonimi. Lo schema definisce tipi, chiavi e vincoli che il server applica indipendentemente dal codice chiamante.

### MySQL e InnoDB

MySQL Server interpreta query e gestisce connessioni, permessi e ottimizzazione; lo **storage engine** realizza memorizzazione e accesso alle tabelle. In MySQL 8.4 InnoDB è il default: supporta transazioni, lock a livello di riga e foreign key. Una tabella può usare un engine diverso, quindi le garanzie non vanno attribuite al solo linguaggio SQL.

MyISAM è ancora supportato, ma è non transazionale e non applica foreign key: `ROLLBACK` non annulla le sue scritture. È rilevante soprattutto nei sistemi esistenti; verificare l'engine delle tabelle prima di ragionare su atomicità e integrità.

### Tabelle, chiavi e vincoli

La **primary key** identifica ogni riga e in InnoDB determina normalmente l'indice clustered; una **foreign key** collega una riga alla chiave di un'altra tabella e impedisce riferimenti non validi. `UNIQUE` impedisce duplicati non nulli, `NOT NULL` richiede un valore e `CHECK` verifica un predicato. In MySQL 8.4 `CHECK` è applicato per default; prima di 8.0.16 la sintassi era accettata ma ignorata. Una condizione `CHECK` che valuta `UNKNOWN` per `NULL` passa: aggiungere `NOT NULL` quando il valore è obbligatorio.

Esempio uno-a-molti, da `customers` a `orders`:

```sql
CREATE TABLE customers (
  id INT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  email VARCHAR(255) NOT NULL UNIQUE
) ENGINE=InnoDB;
CREATE TABLE orders (
  id INT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  customer_id INT UNSIGNED NOT NULL,
  total DECIMAL(12,2) NOT NULL CHECK (total >= 0),
  FOREIGN KEY (customer_id) REFERENCES customers(id)
) ENGINE=InnoDB;
```

`AUTO_INCREMENT` genera chiavi, non numeri contigui garantiti dopo rollback o cancellazioni e non sostituisce l'identità di dominio. Separare clienti e ordini evita di ripetere email in ogni ordine e riduce anomalie di modifica: è il principio pratico della normalizzazione. Denormalizzare un valore derivato può aiutare letture misurate, ma richiede una regola per mantenerlo coerente. `ON DELETE`/`ON UPDATE` della foreign key vanno scelti secondo il dominio, non aggiunti automaticamente.

### Tipi, NULL e collazioni

| Scelta | Comportamento e uso |
| --- | --- |
| `INT`/`BIGINT`, `DECIMAL` vs `FLOAT`/`DOUBLE` | Scegliere intervallo e segno adeguati; `DECIMAL` rappresenta valori decimali esatti, utile per importi, mentre il floating point è approssimato. |
| `VARCHAR` vs `TEXT` | `VARCHAR(n)` impone una lunghezza massima; `TEXT` serve a testi più grandi e ha vincoli diversi per indicizzazione e storage. |
| `DATETIME` vs `TIMESTAMP` | `DATETIME` conserva data e ora senza conversione automatica di fuso; `TIMESTAMP` converte tramite il fuso della sessione da/verso UTC (Coordinated Universal Time) e in MySQL 8.4 ha un intervallo più ristretto, fino al 2038. |
| `JSON` (JavaScript Object Notation) | Memorizza documenti strutturati validi; utile per attributi variabili, senza sostituire colonne e vincoli per dati che si interrogano o relazionano spesso. |

`NULL` significa valore assente o sconosciuto: `col = NULL` non è vero; usare `IS NULL`/`IS NOT NULL`. I confronti possono produrre `UNKNOWN`, che `WHERE` non seleziona; in MySQL un indice `UNIQUE` su una colonna nullable ammette più `NULL`. Il charset `utf8mb4` rappresenta Unicode; la **collation** decide confronto e ordinamento delle stringhe, inclusa sensibilità a maiuscole e accenti. La scelta influisce anche sull'unicità di valori testuali.

`sql_mode` modifica controlli e conversioni: MySQL 8.4 include normalmente `STRICT_TRANS_TABLES` e `ONLY_FULL_GROUP_BY`, ma la configurazione può cambiarli. In modalità strict molti valori invalidi producono errori invece di conversioni con warning; non affidarsi a coercizioni silenziose né supporre che ogni warning sia un successo corretto.

## Query e modifiche

Una query combina righe, applica filtri e produce un risultato; modifiche e letture devono rispettare tipi, vincoli e permessi della connessione.

### SELECT, JOIN e aggregazioni

`SELECT` proietta colonne, `WHERE` filtra righe, `JOIN` combina tabelle, `GROUP BY` forma gruppi e `HAVING` filtra i gruppi. `INNER JOIN` conserva solo coppie corrispondenti; `LEFT JOIN` conserva anche righe della tabella sinistra senza corrispondenza, riempiendo di `NULL` le colonne di destra. Un rapporto uno-a-molti può ripetere la riga sinistra nel risultato; `SELECT` non elimina i duplicati senza `DISTINCT`.

```sql
SELECT c.id, c.email, COUNT(o.id) AS order_count,
       COALESCE(SUM(o.total), 0) AS total
FROM customers AS c
LEFT JOIN orders AS o ON o.customer_id = c.id
WHERE c.email LIKE 'a%'
GROUP BY c.id, c.email
ORDER BY total DESC, c.id
LIMIT 10;
```

Qui `COUNT(o.id)` è zero per un cliente senza ordini; `COUNT(*)` sarebbe uno per la riga prodotta dal `LEFT JOIN`. `SUM` su quel cliente è `NULL`, quindi `COALESCE` restituisce zero. Un filtro come `o.total > 0` in `WHERE` eliminerebbe le righe senza ordine: per mantenerle, collocarlo nella condizione `ON`. `HAVING` filtra dopo l'aggregazione; `WHERE` prima. `ORDER BY` è necessario per un risultato ordinato e un criterio di spareggio rende stabile `LIMIT`; il [contratto di paginazione](../api/16-restful-apis.md#filtri-ordinamento-e-paginazione) è un tema distinto.

Una subquery usa il risultato di un'altra query; una CTE (Common Table Expression) nomina un risultato intermedio, utile per rendere leggibile una query complessa. Una funzione finestra calcola per partizione senza collassare le righe come `GROUP BY`. Sono strumenti per casi analitici, non scorciatoie di performance: controllare piano e volume dei dati. In MySQL 8.4 `ONLY_FULL_GROUP_BY` rifiuta aggregazioni che selezionano colonne né aggregate né determinate funzionalmente dai gruppi.

### Scritture, UPSERT e query sicure

`INSERT` crea righe; `UPDATE` e `DELETE` modificano o eliminano quelle selezionate dal `WHERE`. Verificare vincoli e numero di righe interessate, ma non equiparare sempre zero righe modificate a riga inesistente: un `UPDATE` che imposta il valore già presente può non contarla come modificata, secondo driver e opzioni della connessione. Per più statement che devono riuscire insieme, aprire una transazione.

`INSERT ... ON DUPLICATE KEY UPDATE` esegue l'aggiornamento se una nuova riga viola una primary key o un indice `UNIQUE`; non è un `UPDATE` generico e più vincoli unici possono rendere ambiguo quale conflitto lo attivi. Il conteggio delle righe interessate differisce fra inserimento, aggiornamento e nessun cambiamento e può dipendere dall'opzione `CLIENT_FOUND_ROWS`. `INSERT IGNORE` può trasformare errori in warning o saltare righe: controllare l'esito, soprattutto con `sql_mode` non strict.

I valori non attendibili vanno passati come parametri del driver, mai concatenati nel testo SQL. Esempio con `mysql2/promise`, stringa di connessione configurata fuori dal codice e tabella `customers` già creata:

```js
const mysql = require('mysql2/promise');
const pool = mysql.createPool(process.env.MYSQL_URL);
async function findCustomer(email) {
  const [rows] = await pool.execute(
    'SELECT id, email FROM customers WHERE email = ?', [email]);
  return rows[0] ?? null;
}
```

`execute()` prepara la query e restituisce una Promise della coppia `[rows, fields]`; qui si usa solo `rows` e `null` segnala assenza. L'errore di connessione o query rifiuta la Promise. Il placeholder `?` rappresenta **valori**, non nomi di tabelle o colonne: scegliere identificatori dinamici da una allowlist. Chiudere il pool con `await pool.end()` allo shutdown. Parametri SQL non sostituiscono validazione né autorizzazione applicativa; per il [confine tra repository e service](../engineering/15-architecture-devops.md#pattern-e-struttura) vedi Engineering.

## Indici e piani di esecuzione

Un indice è una struttura ausiliaria che accelera alcuni accessi ma richiede spazio e manutenzione a ogni scrittura. L'ottimizzatore sceglie un piano in base a query e statistiche, non al solo fatto che un indice esista.

### Indici InnoDB e compositi

InnoDB usa normalmente la primary key come **indice clustered**: le righe sono organizzate secondo quella chiave. Un indice secondario contiene anche la primary key della riga, necessaria per recuperare colonne non presenti nell'indice. Per questo chiavi primarie molto grandi aumentano anche il costo degli indici secondari.

Un indice composito `(customer_id, total)` può sostenere ricerche sul prefisso sinistro `customer_id` e su entrambi i campi; una ricerca sul solo `total` non usa quel prefisso per il lookup. Rispetto a un indice singolo su `customer_id`, occupa più spazio ma può servire anche il filtro con ordinamento su `total`. Un indice **covering** contiene tutte le colonne richieste e può evitare la lettura della riga; l'utilità effettiva dipende da filtri, ordinamento, selettività e piano scelto. Indici ridondanti rallentano `INSERT`/`UPDATE`/`DELETE`: progettarli sulle query misurate, non uno per ogni colonna.

### EXPLAIN e ottimizzazione

`EXPLAIN` mostra il piano previsto: ordine delle tabelle, tipo di accesso, indice scelto e righe stimate. `EXPLAIN ANALYZE` **esegue** la query e aggiunge tempi e righe effettive: usarlo con cautela su dati reali, soprattutto per istruzioni che scrivono. Il piano cambia con dati e statistiche; una scansione completa può essere corretta per tabelle piccole o filtri poco selettivi.

```sql
CREATE INDEX idx_orders_customer_total ON orders (customer_id, total);
EXPLAIN SELECT id, total FROM orders
WHERE customer_id = 42 ORDER BY total DESC;
EXPLAIN ANALYZE SELECT id, total FROM orders
WHERE customer_id = 42 ORDER BY total DESC;
```

Confrontare stime e righe effettive prima di cambiare indici. Un predicato non **sargable**, per esempio una funzione applicata alla colonna indicizzata, può impedire un accesso per intervallo: quando possibile riscriverlo come confronto diretto. Offset elevati richiedono comunque di attraversare molte righe; un filtro sull'ultima chiave letta può essere più efficiente, se ordinamento e indice lo permettono. Evitare hint finché statistiche e query non sono state verificate.

## Transazioni e concorrenza

Una transazione raggruppa operazioni da confermare o annullare come unità. Le proprietà **ACID (Atomicity, Consistency, Isolation, Durability)** indicano: tutto o niente per le scritture transazionali; rispetto dei vincoli e delle invarianti applicative; isolamento fra operazioni concorrenti; persistenza dei commit secondo le garanzie e la configurazione del server.

### Autocommit, isolamento e MVCC

Con `autocommit=1`, default di MySQL, ogni statement completato è una transazione se non ne è stata aperta una esplicita. `START TRANSACTION` avvia un'unità di lavoro sulla **stessa connessione**; `COMMIT` conferma, `ROLLBACK` annulla le modifiche ancora non confermate. Una transazione breve limita l'attesa sui lock. Un cambio schema con molti statement DDL (Data Definition Language) può introdurre commit impliciti, quindi non è annullabile come normali scritture InnoDB.

InnoDB usa **MVCC (Multi-Version Concurrency Control)**: le normali letture consistenti possono leggere una versione precedente senza bloccare le scritture; letture con lock e modifiche usano lo stato corrente. Il livello di isolamento determina la visibilità:

| Livello | Comportamento rilevante |
| --- | --- |
| `READ UNCOMMITTED` | Può vedere dati non confermati (*dirty read*); raro per invarianti applicative. |
| `READ COMMITTED` | Ogni lettura consistente usa un nuovo snapshot; due `SELECT` nella stessa transazione possono vedere commit diversi. |
| `REPEATABLE READ` | Default InnoDB 8.4: le letture consistenti condividono lo snapshot stabilito dalla prima lettura; una lettura con lock può invece vedere stato più recente. |
| `SERIALIZABLE` | Aumenta le garanzie di isolamento mediante lock aggiuntivi, con più attese e minore concorrenza. |

Non interpretare `REPEATABLE READ` come assenza universale di phantom o lost update: dipende da tipo di lettura e operazione. Per modifiche condizionate da una lettura, usare lock espliciti o un aggiornamento condizionale e verificarne l'esito, invece di fare affidamento sul solo snapshot.

### Lock, deadlock e retry

`SELECT ... FOR UPDATE` è una **locking read** da eseguire nella transazione che userà i dati; blocca le righe o gli intervalli letti finché la transazione termina. Un lookup tramite indice unico può bloccare il solo record; scansioni per intervallo in `REPEATABLE READ` possono usare **gap/next-key lock** e impedire inserimenti nel range. Indici e predicati influiscono quindi anche sulla contesa, non solo sulla velocità. Un `UPDATE` con condizione di versione e verifica dell'esito è spesso più semplice di leggere e poi scrivere.

Esempio di due scritture correlate con `mysql2/promise`: `pool` è quello creato sopra, `email` e `total` sono già validati. La stessa connessione mantiene la transazione e viene sempre rilasciata:

```js
async function createCustomerWithOrder(pool, email, total) {
  const conn = await pool.getConnection();
  try {
    await conn.beginTransaction();
    const [customer] = await conn.execute('INSERT INTO customers (email) VALUES (?)', [email]);
    await conn.execute('INSERT INTO orders (customer_id, total) VALUES (?, ?)',
      [customer.insertId, total]);
    await conn.commit();
    return customer.insertId;
  } catch (error) {
    await conn.rollback();
    throw error;
  } finally {
    conn.release();
  }
}
```

Un **deadlock** è un ciclo di attese fra transazioni: InnoDB annulla l'intera transazione scelta come vittima, quindi ritentare l'unità completa se sicuro. Un **lock wait timeout**, per default, annulla solo lo statement che ha atteso; l'app deve decidere se fare `ROLLBACK` del resto, come nell'esempio. Retry limitati e con backoff evitano di amplificare la contesa; non ripetere ciecamente effetti esterni o commit dall'esito incerto.

## Operatività ed evoluzione

L'operatività mantiene accessibili e recuperabili i dati mentre schema, carico e deployment cambiano. Le procedure specifiche dipendono da versione, topologia e obiettivi di ripristino.

### Connessioni, sicurezza e migrazioni

Un **pool** riusa connessioni ma va dimensionato rispetto al limite server e al numero di processi: troppe connessioni consumano risorse e possono peggiorare la latenza. Le transazioni devono usare una sola connessione acquisita; rilasciarla in `finally` e chiudere il pool allo shutdown. Gestire timeout e connessioni perse come errori, senza considerarli un successo o presumere che un'operazione già inviata non abbia avuto effetto.

Usare utenti con privilegi minimi, credenziali fuori dal codice e TLS (Transport Layer Security) quando la connessione attraversa una rete non fidata. Le migrazioni versionano lo schema: progettare cambi compatibili fra vecchia e nuova app, misurare lock e durata di `ALTER TABLE`, e predisporre una strategia di ripristino. In MySQL molte istruzioni **DDL**, fra cui `CREATE TABLE` e `ALTER TABLE`, causano commit impliciti: un `ROLLBACK` applicativo non le annulla. Per [rilasci e rollback applicativi](../engineering/15-architecture-devops.md#pipeline-e-rilascio) vedi Engineering.

### Backup, ripristino e osservabilità

Un backup **logico** esporta oggetti e dati in forma ricostruibile; uno **fisico** copia i file del database con un metodo che garantisca consistenza. La scelta dipende da dimensione, tempi e strumenti disponibili: non basta produrre il file, occorre provare periodicamente il restore. Con backup completo e binary log conservati, il **point-in-time recovery** può riprodurre le modifiche fino a un punto successivo al backup.

**RPO (Recovery Point Objective)** è la perdita massima di dati tollerata; **RTO (Recovery Time Objective)** è il tempo massimo di ripristino. Guidano frequenza dei backup, conservazione dei log e prove di recovery. Slow query log, metriche e `performance_schema` aiutano a individuare query lente, lock e saturazione delle connessioni; usare poi `EXPLAIN` sul caso misurato. Una replica può propagare anche errori e cancellazioni, quindi non sostituisce un backup indipendente.

### Replica e consistenza delle letture

La replica MySQL tradizionale trasferisce modifiche dal **binary log** della sorgente a una o più repliche in modo asincrono: una lettura dalla replica può essere indietro rispetto a una scrittura appena confermata sulla sorgente. Per garantire *read-after-write*, leggere dalla sorgente o attendere esplicitamente che la replica abbia applicato la transazione richiesta; non assumere che il failover sia immediato o senza perdita.

I **GTID (Global Transaction Identifier)** identificano transazioni replicate e semplificano il tracciamento delle posizioni, ma non eliminano il lag. La gestione di replica, failover e backup richiede una topologia coerente con RPO/RTO; per i principi generali di [deployment e osservabilità](../engineering/15-architecture-devops.md#observability) vedi Engineering.
