# Configurazione Pipeline CI/CD
## GitHub Actions - CI/CD Pipeline

Sono state create e utilizzate pipeline all-in-one come da best practice ma anche pipeline più brevi per piccole modifiche a distanza ravvicinata, per ridurre tempistiche (in particolar modo dopo l'implementazione di SNYK sulla Repository GitHub del progetto).

L'automazione del deployment segue un flusso lineare e protetto: ogni modifica al ramo main della repository innesca una serie di test funzionali (Pytest) e una scansione di sicurezza (Trivy). Solo in caso di successo, l'immagine viene compilata per architetture multi-platform (AMD64/ARM64) e distribuita sul server on-premises. L'integrità del deployment è garantita dall'uso di tag univoci basati sull'hash del commit (Git SHA), eliminando le ambiguità legate all'uso del tag latest.

## Gestione dei Tag: latest vs SHA
Se si usasse solo il tag latest, ogni nuovo deploy sovrascrive l'immagine precedente sul server.

In caso di errore:

- **Con latest:** L'immagine precedente è andata perduta (o è diventata "dangling"). Per tornare indietro, bisognerebbe ricompilare il vecchio codice, perdendo minuti preziosi.
- **Con il tag SHA...:** Sul Raspberry Pi (e su Docker Hub) convivono diverse versioni (es. sha-abc, sha-def). L'immagine precedente è ancora presente, intatta e pronta all'uso per un rollback immediato.

**Comportamento su Docker Hub e Raspberry Pi**
- Su Docker Hub: Ogni volta che la pipeline esegue il push, il tag :latest viene staccato dalla vecchia immagine e collegato alla nuova. La vecchia immagine diventa orfana (senza nome), rendendo difficile il recupero senza conoscere l'hash specifico.
- Sul Raspberry Pi: Quando si esegue docker compose pull, Docker controlla se l'immagine :latest sul server ha lo stesso Digest (impronta digitale) di quella sul Registry. Se è diversa, la scarica. La vecchia immagine perde il nome :latest e appare nei log come <none>:<none> (immagine dangling).

```yaml
YAML
name: CI/CD Web-API 

on:
  push:
    branches:
      - main

jobs:

# 1. TEST & SECURITY SCAN 
  test-and-scan:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"
          cache: 'pip'

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
          pip install pytest

      - name: Run Pytest
        run: pytest tests

      - name: Scan Repository (FS) with Trivy
        uses: aquasecurity/trivy-action@0.28.0
        with:
          scan-type: 'fs'
          scan-ref: '.'
          severity: 'HIGH,CRITICAL'
          exit-code: '1' # Blocca se il codice o le dipendenze sono vulnerabili
 
# 2. BUILD & PUSH 
  build-and-push:
    needs: test-and-scan
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - name: Build and Push Image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          platforms: linux/amd64,linux/arm64
          tags: |
            enrisox/web-api-pw:latest
            enrisox/web-api-pw:sha-${{ github.sha }}
 
# 3. DEPLOY 
  deploy:
    needs: build-and-push
    runs-on: self-hosted
    steps:
      - name: Docker login on Raspberry
        run: echo "${{ secrets.DOCKERHUB_TOKEN }}" | docker login -u "${{ secrets.DOCKERHUB_USERNAME }}" --password-stdin
      - name: Deploy with Docker Compose
        working-directory: /home/enrico/web-api-pw
        run: |
          echo "IMAGE_TAG=sha-${{ github.sha }}" > .env
          
          # Pull specifico dell'immagine appena buildata
          docker compose pull web-api
          
          # Update del solo servizio web-api
          docker compose up -d --remove-orphans web-api
      - name: Cleanup old images
        run: |
          docker image prune -f --filter "until=48h"
```

## Analisi delle Strategie di Deployment e Sicurezza
- **L'architettura**della pipeline si basa su quattro pilastri tecnici:
- **Sicurezza Predittiva (Shift-Left Security)**: Anticipare i controlli alle fasi precoci del ciclo di sviluppo (SDLC). L'analisi di Trivy viene eseguita sul file system prima della creazione dell'immagine Docker. Questo approccio "fail fast" permette di intercettare vulnerabilità nel codice o nei requirements.txt con costi di correzione minimi rispetto a un intervento in produzione.
- **Determinismo del Deployment e Immutabilità**: La generazione dinamica del file .env forza Docker Compose a utilizzare l'hash univoco del commit (Git SHA). Questo garantisce una corrispondenza biunivoca tra il codice su GitHub e l'istanza in esecuzione, eliminando ambiguità legate alla cache.
- **Ottimizzazione delle Risorse e Continuità**: Il comando docker compose up -d web-api aggiorna solo il servizio applicativo. In un ambiente limitato come il Raspberry Pi, questo permette di mantenere attivi il database (MariaDB) e gli altri servizi, evitando micro-interruzioni o errori di riconnessione.
- **Manutenzione Automatica e Rollback**: La pulizia temporizzata (until=48h) bilancia lo spazio libero sull'SSD con la sicurezza operativa. Mantenere i layer delle ultime 48 ore consente un ripristino della versione precedente in pochi secondi in caso di anomalie critiche.

## SNYK: Integrazione nel Progetto
L'integrazione di **Snyk** nella repository **GitHub** aggiunge un monitoraggio 24/7 sugli asset software, complementare all'azione di Trivy nella pipeline.

- **Scansione Statica (SCA)**: Analisi automatizzata del file requirements.txt per identificare librerie vulnerabili, indicando la versione minima sicura per il fix.
- **Monitoraggio Continuo e Alerting**: Snyk opera anche post-push. Se emerge una nuova vulnerabilità in una libreria già distribuita, invia un alert immediato. Nelle Pull Request, impedisce il merge di nuovo codice che introduce dipendenze insicure.
- **Automatizzazione dei Fix**: Generazione automatica di Fix Pull Requests per aggiornare le versioni delle librerie vulnerabili, accelerando il tempo medio di risoluzione (MTTR).
- **Snyk Code (SAST) & Container**: Analisi del codice Python per individuare pattern insicuri (es. SQL Injection) e raccomandazioni sulle immagini base (es. suggerendo il passaggio da "bullseye" a "bookworm") prima ancora del processo di build.

**L'uso combinato di Snyk (monitoraggio continuo sulla repository) e Trivy (blocco preventivo nella pipeline)** crea un sistema di sicurezza a doppio fattore.
