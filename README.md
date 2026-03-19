# DevSecOps Web API & Data Analytics 

This repository contains my **Second-Year Group Project** for the **IFOA ITS Olivetti**'s course **“Information Systems Security & Integration Specialist – DevSecOps”**. 

It represents a comprehensive implementation of secure software development, automated infrastructure, and data science.

## Our Team
RE-boot

## My Role
In this project, I served as the **System Administrator and DevSecOps Engineer**. 
My key responsibilities included:
* **Team Management:** Coordinating five specialized roles and ensuring project milestones were met.
* **Infrastructural Design:** Architecting the on-premises ARM64 environment (Raspberry Pi 5) and Ubuntu Server hardening.
* **DevSecOps Implementation:** Building the CI/CD pipeline, integrating security scanners (Trivy, Snyk), and managing container orchestration.
* **Network Security:** Configuring Caddy Reverse Proxy, Cloudflare WAF, and IDS/IPS systems (CrowdSec/Fail2ban).

## 1. Project Overview
The project focuses on building a secure ecosystem for delivering data analytics via REST APIs. Developed in **Python/Flask**, the service is hosted in a **Docker** environment and processes complex datasets using **Pandas**. 

### Key Features
* **Secure REST API:** JWT-based authentication and strict input sanitization.
* **Data Intelligence:** Advanced processing of CVE and business datasets.
* **Hardened Infrastructure:** Hosted on-premises with optimized energy efficiency and performance/cost ratio.
* **Shift-Left Security:** Automated vulnerability scanning at every commit.

## 2. Tech Stack & Architecture

### Infrastructure (On-Premises & ARM64)
* **Hardware:** Raspberry Pi 5 (8GB RAM, 250GB SATA SSD).
* **OS:** Ubuntu Server 24.04.4 LTS (Headless, automated security updates).
* **Containerization:** Docker & Docker Compose.
* **Orchestration & UI:** Portainer CE.

### Backend & Data Analysis
* **Language:** Python 3.13-slim.
* **Framework:** Flask (Gunicorn WSGI).
* **Library:** Pandas (Data Cleaning, KPI calculation, Risk Scoring).
* **Database:** MariaDB 11 (Isolated in internal Docker network).

### Security & Networking
* **Edge Proxy:** Cloudflare (Anti-DDoS, WAF, Global CDN).
* **Reverse Proxy:** Caddy (Automated TLS 1.3 with Let's Encrypt).
* **Defense in Depth:** Fail2ban, CrowdSec, UFW (Firewall), and Wireguard VPN for remote admin.

## 3. CI/CD Pipeline (GitHub Actions)
The deployment automation ensures that only secure and tested code reaches the production server:

1.  **Phase 1: Security & Logic Test**
    * **Pytest:** Validates API endpoints and Pandas logic.
    * **Trivy (FS Scan):** Blocks the pipeline if `requirements.txt` or source code contains HIGH/CRITICAL CVEs.
2.  **Phase 2: Build & Push**
    * Compiles Multi-platform images (AMD64/ARM64).
    * Pushes to Docker Hub using **Git SHA** tags to ensure deployment determinism.
3.  **Phase 3: Automated Deploy**
    * The self-hosted runner triggers an update on the Raspberry Pi.
    * Uses a **Read-Only** filesystem strategy for the container runtime to prevent unauthorized modifications.

## 4. Functional Modules

### Data Analysis (pd_analysis.py)
* **Risk Score Calculation:** `risk_score = cvss + exploitability` (Network access, No Auth, Low complexity).
* **KPI Extraction:** Automated calculation of average CVSS, critical percentage, and network-exploitable trends.
* **Trend Analysis:** Monthly publication trends for vulnerabilities.

### Middleware & Logging
* **Request Fingerprinting:** Unique hash generation for request correlation.
* **Sensitive Data Masking:** Automatic obfuscation of `Authorization` and `Cookie` headers in logs.
* **Log Rotation:** Automated cleanup to prevent SSD storage exhaustion.


## 5. Testing & Validation
The project includes a robust suite of automated tests:
* **Mocking:** Uses `MagicMock` to simulate Database responses during CI, ensuring tests run without a live MariaDB instance.
* **Security Validation:** Tests for weak passwords (Regex), duplicate users, and unauthorized API access.

## 📂 Project Documentation Index

Click on the links below to explore the detailed technical documentation for each module:

### 🏛️ General & Infrastructure
* [🇮🇹 Project Description (Italiano)](./Descrizione%20del%20Progetto%20(Italiano).md) - Overview e obiettivi del Project Work.
* [🖥️ System Infrastructure](./Infratruttura.md) - Hardware (Raspberry Pi 5) and OS (Ubuntu Server) specifications.
* [🐳 Docker Containerization](./Docker.md) - Container strategy, isolation, and orchestration.

### 💻 Development & Logic
* [⚙️ Backend API](./Backend.md) - Flask development, JWT authentication, and secure endpoints.
* [📊 Data Analysis (Pandas)](./Pandas.md) - Data cleaning, KPI calculation, and risk scoring logic.
* [🎨 Frontend UI](./Frontend.md) - Single Page Application, Bootstrap design, and JS integration.

### 🛡️ DevSecOps & QA
* [🧪 Pytest Suite](./Pytest.md) - Automated testing, mocking, and validation procedures.
* [🚀 CI/CD Pipeline (GitHub Actions)](./Pipeline_Github_Actions.md) - Automated deployment and Shift-Left security scanning.

### ⚖️ Legal
* [📄 LICENSE](./LICENSE) - Project licensing information.

---

# About Me
My name is Enrico and I am a second-year student at the ITS Academy Olivetti, enrolled in the course **“Information Systems Security & Integration Specialist – DevSecOps”.**
I am passionate about experimenting and building hands-on labs with the tools at my disposal.

This project was developed as part of my training to practice and consolidate skills in DevSecOps tools and methodologies, including automation, infrastructure management, CI/CD pipelines, and containerized environments. 

⭐ If you liked my project, give it a star!

enricosoci@protonmail.com
