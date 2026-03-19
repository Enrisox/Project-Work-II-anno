## Data Analysis (Pandas)

### 1. **data_sources.py**
Questo modulo fornisce una semplice interfaccia per caricare dati dal database MySQL all'interno di DataFrame di pandas. L'obiettivo è semplificare l'esecuzione di query SQL e restituire i risultati in una struttura dati adatta all'analisi e alla manipolazione dei dati.

Il modulo gestisce automaticamente:
* La connessione al database
* L'esecuzione della query
* La conversione dei risultati in un `pandas.DataFrame`
* La chiusura della connessione al termine dell'operazione

#### **Funzione `load_data`**
Esegue una query SQL su un database MySQL e restituisce i risultati sotto forma di `pandas.DataFrame`. Questa funzione gestisce internamente:
* L'apertura della connessione al database
* L'esecuzione della query
* La lettura dei risultati tramite `pandas.read_sql`
* La chiusura della connessione (gestita tramite un blocco `try / finally`)

### 2. **pd_analysis.py**
Questo modulo fornisce una serie di funzioni per pulire, analizzare e aggregare dati relativi alle CVE utilizzando pandas DataFrame.

Le funzioni incluse nel modulo permettono di:
* Pulire e normalizzare dataset di vulnerabilità
* Classificare le vulnerabilità in base alla severità CVSS
* Identificare le vulnerabilità più comuni
* Calcolare KPI statistici sulla sicurezza
* Analizzare trend temporali delle CVE
* Individuare le vulnerabilità più rischiose
* Analizzare la distribuzione degli impatti sulla sicurezza

#### **2.1 clean_data**
La funzione `clean_data` esegue operazioni di pulizia e normalizzazione del dataset prima dell'analisi. Le principali operazioni effettuate sono:
* Normalizzazione dei nomi delle colonne
* Rimozione dei duplicati
* Sostituzione dei valori mancanti

#### **2.2 cvss_severity**
Questa funzione classifica le vulnerabilità in base al punteggio CVSS e restituisce il numero di vulnerabilità per ciascun livello di severità.

| Range CVSS | Severity |
| :--- | :--- |
| 0 – 4 | Low |
| 4 – 7 | Medium |
| 7 – 9 | High |
| 9 – 10 | Critical |

#### **2.3 top_cwe**
Questa funzione identifica le vulnerabilità più frequenti per tipo di debolezza (CWE). Le vulnerabilità vengono raggruppate per:
* Codice CWE
* Nome della vulnerabilità
* Ordinamento in base alla frequenza

#### **2.4 cve_kpi_extended**
Questa funzione calcola indicatori KPI avanzati relativi al dataset di vulnerabilità. Gli indicatori restituiti includono:
* `avg_cvss`: punteggio CVSS medio
* `median_cvss`: mediana del punteggio CVSS
* `high_or_critical_percentage`: percentuale di vulnerabilità con CVSS ≥ 7
* `critical_percentage`: percentuale di vulnerabilità con CVSS ≥ 9
* `network_exploitable_percentage`: vulnerabilità sfruttabili via rete
* `no_auth_required_percentage`: vulnerabilità che non richiedono autenticazione
* `network_noauth_percentage`: vulnerabilità sfruttabili via rete senza autenticazione

#### **2.5 cve_trend_by_month**
Questa funzione analizza il trend temporale delle vulnerabilità in base al mese di pubblicazione.

#### **2.6 top_riskiest_cves**
Questa funzione identifica le vulnerabilità più rischiose combinando la severità CVSS e le caratteristiche di exploitabilità. Per ogni vulnerabilità viene calcolato un risk score.

Il punteggio viene calcolato come:
`risk_score = cvss + exploitability`

Dove `exploitability` è la somma di tre condizioni (ogni condizione aggiunge 1 punto):
1. Accesso via rete
2. Nessuna autenticazione richiesta
3. Bassa complessità dell'exploit

#### **2.7 impact_distribution**
Questa funzione calcola la distribuzione percentuale degli impatti di sicurezza nel dataset analizzando tre tipi di impatto:
* Confidentiality
* Integrity
* Availability

Per ogni tipo di impatto:
* Viene calcolata la frequenza dei valori
* La frequenza viene normalizzata in percentuale
* I risultati vengono arrotondati a due decimali

### **Dependencies**
Il modulo utilizza le seguenti librerie:
* **pandas**: Utilizzata per caricare i risultati delle query SQL in un DataFrame.
* **mysql.connector**: Libreria per stabilire una connessione con il database MySQL.
