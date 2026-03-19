# Sviluppo del Backend

# 1. File Principale: app.py

Il file app.py è il punto d'ingresso principale (entry point) dell'applicazione Flask. 

**Le sue responsabilità principali sono:**

- Configurazione iniziale e variabili d'ambiente: Carica le variabili dall'ambiente (o dal file .env) tramite blocco load_dotenv() per configurare parametri critici come DB_HOST e APP_SECRET_KEY ancor prima di importare altri moduli. Include accurati controlli di sicurezza che bloccano l'avvio (sollevando una RuntimeError) se queste chiavi non sono inizializzate.
- Inizializzazione di Flask e CORS: Crea l'istanza Flask(__name__) e configura in modo sicuro il Cross-Origin Resource Sharing (CORS) per permettere chiamate API autorizzate solo da domini frontend controllati (ALLOWED_ORIGINS).
- Inizializzazione dei Middleware: Inizializza il middleware custom di logging (init_logging_middleware) per tracciare il traffico in ingresso, disabilitando al contempo il rumoroso log predefinito di werkzeug (salvo errori) e garantendo la pulizia dei log obsoleti.
- Registrazione dei Blueprint (Controllers): Scompone il routing agganciando all'app principale le varie sezioni o percorsi delle API definite nei controller: user_login_controller, user_signup_controller e csv_controller.
- Avvio dell'Applicativo: Fornisce un endpoint di base / che renderizza la homepage index.html. Per lo sviluppo locale, avvia il server sulla porta 6700 in ascolto su tutte le interfacce (0.0.0.0).

## 2. Controllers
I controllers sono implementati come Blueprint logici di Flask. Gestiscono lo smistamento delle route per l'API (i vari endpoint esposti), processano il payload ed emettono i dati in formato JSON gestendo i previsti codici di stato (HTTP status code).

### 2.1 user_login_controller.py
Gestisce l'autenticazione, la login list e l'emissione dei token validi (JWT) necessari ad accedere alle risorse protette dell'API.
**- Endpoint /api/login (POST):**
  Estrae username e password dal corpo della request. Valida la presenza di tali campi obbligatori pena un errore 400.
  Verifica le credenziali mediante verify_password().
  Verifica inoltre lo stato di attivazione dell'account (user.is_verified), impedendo l'accesso e lanciando errore 403 con messaggio se la mail non risulta ancora confermata.
Se le verifiche passano, genera e restituisce al client un token di sessione firmato (generate_token()). Ritorna un 401 in caso di credenziali fallite.

- **Endpoint /api/refresh (POST):**
Consente di ottenere un nuovo token a partire da uno ancora valido (o appena scaduto) prolungando così la sessione senza costringere l'utente ad effettuare un nuovo accesso manuale prolungato.
Estrae e verifica il token intercettato nell'header Authorization, confermando l'utente e generando al volo un new_token.

### 2.2 user_signup_controller.py
Contiene la logica per la registrazione dei nuovi prospect e il delicato ciclo di verifica/attivazione account (onboarding).
- **Endpoint /api/signup (POST):**
Registra l'account tramite username, password ed email.
Implementa protezioni anti-errori con stringenti espressioni regolari (regex). Nello specifico, la password deve avere lunghezza minima (8), almeno una maiuscola, un numero e un carattere speciale. Controlla, analogamente, la corretta composizione standard della mail.
Previene duplicazioni incrociate consultando il database su account o mail omonimi già occupati (HTTP 409 Conflict).
Assicura un salvataggio blindato criptando la password con un hash bcrypt e inoltra una mail asincrona reale all'utente tramite send_verification_email(), equipaggiata con un token URL sicuro a 32 byte. Ritorna il payload 201 confirm e lo user_id.
- **Endpoint /api/verify-email (GET):**
Genera una vista HTML a seguito del click, da parte dell'utente, del link di notifica ricevuto per email.
Raccoglie dalla parametrizzazione dell'URL il token utente transitorio per consumarlo, convertendolo come verificato ad opera di verify_user_by_token(). Gestisce con interfacce grafiche sia i successi sia gli errori, offrendo poi un comodo switch-back (Via alla Home).

### 2.3 csv_controller.py
Questo modulo orientato sui dati gestisce, principalmente appoggiandosi alle librerie d'analisi (come pandas), l'aggregazione di KPI e metriche mirate sui CVE (Common Vulnerabilities and Exposures) pronte per esser passate graficamente alla Dashboard backend/frontend.
**Endpoint /api/csv/summary (GET):**
Endpoint totalmente protetto dal decoratore @require_auth (riservato agli autenticati).
Pone delle prime interrogazioni di selezione massive al DB SQL, richiedendo tutte le CVE esposte a "NETWORK".
Sfrutta funzioni importate esterne (cvss_severity, top_cwe, cve_kpi_extended ecc…) per estrarre il livello di severità globale, le vulnerabilità dominanti, matrici KPI complete e i difetti top rischiosi.
Calcola internamente e real-time un raggruppamento storico (per la timeline line chart) sfruttando la data di pubblicazione (pub_date), componendo i conteggi CVE storici su base annua. Ritorna l'agglomerato completo elaborato direttamente in JSON.


## 3. Helpers
Il gruppo degli helper incapsula classi delegate puramente alla manipolazione della sicurezza e dell'astrazione dell'I/O (database e crittografie), facilitando notevolmente il codice dei controllers.

**3.1 auth_helper.py**
Motore del livello middleware personalizzato su autorizzazione e protezione passiva/attiva.
Decoratore @require_auth: Un interruttore architetturale protettivo (wrapper). Iniettato in calce sopra le chiamate agli endpoint, disseziona autonomamente le intestazioni Authorization per scovare token Bearer, verificandone crittografia, esistenza e consistenza ed abortendo (401) silente in caso non corrisponda, evitando esecuzioni illecite di root.
Gestione Password (hash_password, verify_password_hash, verify_password): Sfrutta integralmente lo strato Python bcrypt. Crittografare (hashing) irreversibilmente le chiavi segrete avviandosi per un sale crittografico di base in scrittura (bytes). Assicura test di comparazione univoca in decodifica in lettura (con gestione solida dell'eccezione se la computazione fallisce o crasha). Identifica dinamicamente le profilazioni intercettando o l'username o la email e restituendo is_ok e lo sblocco dello User.
Motore JWT (generate_token, verify_token): Configura la spina dorsale della generazione ed emissione logica su libreria open PyJWT su protocollo ad asimmetria HS256 in combinata con chiave proprietaria .env. Produce Payload certificati marchiati via uuid4 dotati di metadati di tempo e bloccante scadenza a minuti limitati, ma estraibile in maniera ignorata da timestamp di blocco (verify_exp: False bypass) in casi di auto-refresh programmati protetti.

**3.2 user_helper.py**
Il ponte essenziale di disaccoppiamento del Data Access Object (DAO) che isola il driver query MySQL (ORM-like proxy) dalla logica business.
Resilienza della Connessione (get_db_connection): Affina la delicatezza in fase di avvio gestendo nativamente un blocco strutturato di tolleranza all'errore e ritentativi a loop (re-try multiplo a colpi distanziati in secondi time.sleep) contro fastidiosi lag operativi in contesti dockerizzati dove MySQL fatica inizialmente in boot e si rifiuta di connettersi prematuramente.
Parsing Intelligente (row_to_user): Tramuta serializzazioni grezze strutturate di riga SQL DB (dictionary-mapped) a comodi e dotati oggetti User istanziati su dataclass.

### Istradamento Manipolativo SQL:
- **create_user():** Inizializa query di INSERT esplicite passate da tuple parametrale, offrendo robustezza totale alle varianti SQL Injection, occupandosi pure dei rollback preventivi con cancellazione istruzione base sul try-catch se per un qualunque imprevisto l'immissione o la persistenza crasha. Ritorna id generati in last row.
- **get_user_by_username_or_email(identifier)**: Costruisce filtri stringenti flessibili (BINARY forzata per il case sensitivity di stringhe SQL match) catturando tuple di corrispondenze su più vie (se metto nome trovo utente, se metto mail lo trovo comunque).
- **verify_user_by_token():** Assicura procedure veloci di aggancio UPDATE, consumando il contatore e azzerandolo (verification_token = NULL, is_verified=TRUE) in blocco unico di sicurezza su singola transazione (commit()), ritornando vero su affettazione positiva della rowcount.

### Integrazione Invio Email (Resend)
Il sistema applicativo integra l'API del provider Resend per la gestione e l'invio automatizzato delle comunicazioni transazionali. L'implementazione primaria riguarda l'invio delle email di benvenuto e di verifica dell'account per l'onboarding di nuovi utenti. L'intera logica è incapsulata all'interno del modulo dedicato helpers/email_helper.py.
Architettura e funzionalità Implementate
**Astrazione dell'invio (send_verification_email):** Il modulo espone una singola funzione che prende in carico l'indirizzo email di destinazione, il nome dell'utente e il token crittografico univoco generato in fase di registrazione per l'attivazione dell'account.
Gestione dinamica dell'ambiente e Dummy Mode: Le configurazioni sensibili, quali ad esempio RESEND_API_KEY e l'anagrafica del mittente (RESEND_FROM_EMAIL), vengono estratte in tempo reale dalle variabili d'ambiente in totale sicurezza. È stato inoltre implementato un meccanismo di fallback protettivo: nel caso in cui la chiave API non venga configurata o utilizzi chiavi mockate (es. valori che iniziano per re_dummy), il sistema non arresta in modo anomalo l'esecuzione dell'app, bensì simula l'invio registrandolo internamente e garantendo così l'operatività del flusso in locale o durante i test end-to-end senza intaccare le quote mensili di Resend.
Dinamicità delle rotte di re-indirizzamento: I link di verifica generati all'interno delle email utilizzano URL radice dinamici dedotti dalla variabile ALLOWED_ORIGINS. Tale architettura rende il componente agnostico rispetto all'ambiente (Sviluppo Locale, Staging o Produzione) adattandosi in automatico senza necessità di ri-compilazioni.
Design Template e UI coerente: Il corpo dell'email viene generato tramite un template HTML inline formattato con regole CSS responsive. Lo stile rispecchia fedelmente l'interfaccia utente del frontend, garantendo coerenza estetica all'esperienza utente. Viene inoltre fornito sempre un link di recupero in testo puro nel caso i client di posta blocchino temporaneamente i contenuti multimediali o l'HTML.
Error Handling e resilienza: L'effettiva comunicazione asincrona col server Resend (resend.Emails.send) è racchiusa all'interno di un blocco try-except. Tale configurazione assicura un'alta resilienza della piattaforma: un eventuale down dell'API esterna di Resend si traduce in un log di errore specifico, ma l'iscrizione a database procede impedendo il congelamento (crash) della richiesta originaria dell'utente.
