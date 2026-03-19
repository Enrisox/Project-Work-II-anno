# Sviluppo del Frontend (HTML/JavaScript/CSS)
L'interfaccia utente è un'applicazione web **single-page** che interagisce con il backend tramite chiamate API.

## **Struttura HTML (index.html)**
Suddivisa in sezioni logiche che vengono mostrate o nascoste dinamicamente.

* **Sicurezza:** Include un tag meta *Content-Security-Policy* (CSP) per mitigare attacchi XSS e controllare le fonti di script, stili e font.
    ```html
    <meta http-equiv="Content-Security-Policy" 
          content="default-src 'self'; script-src 'self' 'unsafe-inline' https://cdn.jsdelivr.net; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com https://cdn.jsdelivr.net; font-src 'self' https://fonts.gstatic.com https://cdn.jsdelivr.net; connect-src 'self' http://localhost:5001 http://127.0.0.1:5001;">
    ```
* **Sezione Pubblica:** Comprende la "Home" (landing page) e la sezione "About" con la presentazione del team di Data Analyst & DevOps.
* **Gestione Autenticazione:** Contiene i moduli per il Login e la Registrazione, con spiegazioni in linea sui requisiti della password.
* **Dashboard Protetta:** Una vista complessa accessibile solo post-login, contenente una sidebar di navigazione, KPI, grafici multipli e una "Legenda Analisi" testuale.
* **Overlay Modali:** Definisce componenti a comparsa globale per la gestione degli errori, le notifiche di sistema e l'avviso di scadenza della sessione.

---

### **Logica JavaScript (`main.js`)**
Gestisce le chiamate API, l'autenticazione, il routing interno e il rendering dei dati.

#### **Gestione dell'Autenticazione e Sicurezza**
* **Sanificazione Input:** Implementa la funzione `sanitizeInput()` per convertire i tag in testo semplice, offrendo una prima barriera lato client contro il Cross-Site Scripting (XSS).
* **Validazione Registrazione:** Applica una Regex per forzare password sicure (minimo 8 caratteri, una maiuscola, un numero, un carattere speciale).
    ```javascript
    // Almeno un uppercase, un numero, un carattere speciale e lunghezza min 8 
    const passRegex = /^(?=.*[A-Z])(?=.*[0-9])(?=.*[!@#$%^&*(),.?":{}|<>]).{8,}$/; 

    if (!passRegex.test(pass)) { 
        return showNotification("Errore", "email o password inseriti nel modo errato", true); 
    }
    ```
* **Gestione Sessione:** Decodifica il payload del JWT tramite `parseJwt()` per leggere la data di scadenza.
    ```javascript
    function parseJwt(token) { 
         try {
             return JSON.parse(atob(token.split('.')[1])); 
         } catch (e) {
             return null; 
         } 
    }
    ```
* **Rinnovo Token:** Attiva un timer `startTokenTimer()` che mostra un conto alla rovescia modale quando mancano 60 secondi alla scadenza, permettendo all'utente di estendere la sessione tramite l'endpoint `/refresh`.
    ```javascript
    const timeLeft = payload.exp - Math.floor(Date.now() / 1000); 

    if (timeLeft <= 60 && timeLeft > 0 && !isRefreshing) { 
        isRefreshing = true;
        overlay.classList.remove('hidden'); 
    }
    ```

#### **Rendering dei Dati**
* **Recupero Dati:** Richiama l'endpoint protetto `/api/csv/summary` includendo il token di autorizzazione per scaricare il dataset analizzato.
* **Gestione Chart.js:** Distrugge e ricrea dinamicamente le istanze dei grafici (`severityChart`, `cweChart`, `impactChart`) per evitare sovrapposizioni visive durante gli aggiornamenti.

---

### **Design del sito (`style.css`)**

* **Variabili CSS Centralizzate:** Definisce nel `:root` una palette di colori, sfumature e gradienti che rendono il design scalabile e facilmente manutenibile.
* **Gestione Temi (Light/Dark):** Il tema predefinito è scuro (Dark Mode). L'attivazione del tema chiaro avviene modificando l'attributo `data-theme="light"` sul tag `<html>`, che sovrascrive istantaneamente le variabili CSS di base.
* **Effetto Vetro (Glassmorphism):** Utilizza proprietà come `backdrop-filter: blur(20px)` e sfondi semitrasparenti (`rgba`) per i pannelli principali, creando un effetto di profondità.
* **Layout Flessibile:** Utilizza estensivamente **CSS Grid** (es. `.dashboard-layout`, `.analysis-grid`) e **Flexbox** per garantire la responsività su desktop e dispositivi mobili.
* **Feedback Visivo:** Applica transizioni morbide (`transition: all 0.2s ease`) su tutti gli elementi e animazioni fade-in per la comparsa dei blocchi di contenuto.
