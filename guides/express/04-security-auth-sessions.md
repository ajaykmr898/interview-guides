# Sicurezza autenticazione e sessioni Express

La sicurezza di un'app Express dipende dai confini tra richiesta HTTP (Hypertext Transfer Protocol), identità verificata e permessi applicativi. Express offre routing e middleware, ma non include autenticazione, autorizzazione o uno store di sessione pronti per l'applicazione.

## Indice

- [Confini di sicurezza](#confini-di-sicurezza)
- [Autenticazione e autorizzazione](#autenticazione-e-autorizzazione)
- [Difese delle operazioni](#difese-delle-operazioni)

## Confini di sicurezza

### Input, header e proxy

`req.params`, `req.query`, `req.body`, cookie e header provengono dal client: validarli prima di usarli per query, redirect, risposte o decisioni di sicurezza. Impostare limiti ai body e produrre output adatto al contesto; un middleware per header di sicurezza, come Helmet, aggiunge difese del browser ma non sostituisce validazione e autorizzazione. Per il [confine di fiducia dell'input](02-request-response-validation.md#input-della-richiesta) vedi il capitolo precedente.

Proteggere il trasporto con HTTPS (HTTP su TLS, Transport Layer Security), direttamente o tramite un reverse proxy configurato correttamente. `app.disable('x-powered-by')` riduce l'esposizione del nome del framework, senza costituire una misura di accesso. Non costruire URL di redirect da valori arbitrari della richiesta: ammettere solo destinazioni previste.

`trust proxy` è disattivato per default. Dietro un proxy, configurarlo secondo indirizzi o numero di hop realmente fidati: influenza `req.ip`, `req.hostname` e `req.protocol`, usati anche per rate limiting e cookie `Secure`. Se il proxy non elimina o sovrascrive `X-Forwarded-For`, `X-Forwarded-Host` e `X-Forwarded-Proto`, il client può falsare questi valori; fidarsi indiscriminatamente di tutti i proxy è quindi pericoloso.

### CORS e rate limiting

**CORS (Cross-Origin Resource Sharing)** indica al browser quali origini possono leggere una risposta cross-origin. Il middleware `cors` configura gli header, ma non autentica né blocca client non browser. Se si permettono credenziali cross-origin, autorizzare origini esplicite: il browser non accetta `Access-Control-Allow-Origin: *` per richieste con credenziali. Per il comportamento del browser vedi [isolamento e sicurezza Web](../../topics/web/01-html-web.md#isolamento-e-sicurezza).

Il **rate limiting** limita richieste per chiave e finestra temporale, per esempio per account o indirizzo IP (Internet Protocol); è particolarmente utile su login e operazioni costose. Applicarlo tramite middleware o gateway, restituendo `429` quando la quota è superata e, se utile, `Retry-After`. Con più istanze serve uno stato condiviso o un limite al gateway; una chiave basata su `req.ip` è affidabile solo con proxy configurati correttamente. Non trattarlo come sostituto dell'autorizzazione.

## Autenticazione e autorizzazione

**Autenticazione** verifica chi presenta le credenziali; **autorizzazione** decide se quell'identità può compiere una specifica operazione su una specifica risorsa. Il middleware può condividere l'identità verificata con gli handler, ma Express non crea automaticamente `req.user`.

### Sessioni e cookie

Con il middleware esterno `express-session`, il cookie contiene l'identificatore della sessione, mentre i dati di sessione risiedono in uno store server-side. Usare uno store condiviso e affidabile quando l'app ha più processi o istanze: il `MemoryStore` predefinito serve allo sviluppo, non alla produzione. La sessione permette revoca e stato lato server, ma richiede gestione dello store; un token bearer evita il lookup della sessione solo se il formato e la verifica lo consentono, e ha trade-off di revoca propri.

Configurare cookie di sessione con `Secure` su HTTPS, `HttpOnly` per impedirne la lettura da JavaScript e `SameSite` secondo i flussi cross-site richiesti; `SameSite=None` richiede `Secure` nei browser. Definire scadenza e ambito appropriati. `HttpOnly` non ferma richieste inviate dal browser con il cookie. Dopo il login rigenerare l'identificatore di sessione per ridurre il rischio di session fixation; al logout invalidare la sessione sul server. Dietro proxy, verificare `trust proxy` prima di basarsi sul rilevamento di connessione sicura per impostare il cookie.

### Token e middleware di autorizzazione

Un client può inviare un token bearer nell'header `Authorization`. Il middleware di autenticazione deve verificarlo **sul server** prima di associare un'identità alla richiesta: per un JWT (JSON Web Token) firmato verificare firma, algoritmo consentito, scadenza e claim pertinenti come issuer e audience; per un token opaco consultare l'autorità o lo stato server-side previsto. Decodificare un token non equivale a verificarlo. Un bearer rubato può essere riutilizzato, quindi proteggerlo nel trasporto e nei log.

Ordine tipico: verificare credenziali, caricare la risorsa, controllare il permesso sulla risorsa, quindi eseguire la modifica. Un controllo di ruolo sulla sola route non prova la proprietà o l'accesso al singolo record. Per credenziali bearer mancanti o non valide usare `401` con `WWW-Authenticate: Bearer`; per un'identità verificata ma priva del permesso usare `403`, secondo il [contratto di autenticazione HTTP](../../topics/api/16-restful-apis.md#sicurezza-e-verifica-del-contratto).

```js
function requireAdmin(req, res, next) {
  if (!req.user) return res.status(401).set('WWW-Authenticate', 'Bearer').json({ error: 'Autenticazione richiesta' });
  if (!Array.isArray(req.user.roles) || !req.user.roles.includes('admin')) {
    return res.status(403).json({ error: 'Accesso negato' });
  }
  next();
}
```

Questo middleware presuppone che un precedente autenticatore bearer abbia impostato `req.user` con dati verificati; controlla un ruolo, mentre l'autorizzazione per risorsa resta a carico dell'handler che la carica.

## Difese delle operazioni

### CSRF e dati sensibili

**CSRF (Cross-Site Request Forgery)** sfrutta credenziali inviate automaticamente dal browser, soprattutto cookie di sessione, per indurre una modifica non voluta. Per operazioni che cambiano stato, verificare un token CSRF legato alla sessione o l'origine della richiesta secondo il flusso; un'origine assente o non attendibile non prova la legittimità. `SameSite` è una difesa aggiuntiva, non l'unico controllo. CORS non impedisce da solo l'invio di una richiesta con effetti. Un bearer inviato esplicitamente in `Authorization` non è una credenziale ambientale, ma resta esposto ad altri rischi se uno script malevolo può leggerlo.

Conservare segreti e chiavi fuori dal codice e dai log, limitare l'accesso ai dati personali e non includere token, sessioni, stack trace o dettagli interni nelle risposte d'errore. Loggare il contesto necessario alla diagnosi senza copiare l'intera richiesta. Per i meccanismi generali di [cookie, CSRF e sicurezza browser](../../topics/web/01-html-web.md#isolamento-e-sicurezza) vedi HTML e Web.
