# 🛠️ Guida all’installazione di n8n (Docker, persistente, fs-enabled)

Questa guida spiega come installare **n8n** su una macchina Linux (es. Ubuntu) usando Docker, con:

* **Persistenza dei dati**
* **Accesso al filesystem host** (`/data`)
* **Esecuzione di script `fs` (Node.js built-in)**
* **Autostart all’avvio**
* **Tunneling webhook diretto (no n8n.cloud)**

---

### 🔧 Requisiti

* Docker installato e funzionante (`docker --version`)
* Una cartella host dove leggere/scrivere i file (es. `~/n8n/jekyll`)
* Un IP pubblico accessibile per i webhook (es. server VPS con porta 5678 aperta)

---

### 📁 1. Prepara la cartella dati

```bash
mkdir -p ~/n8n/jekyll
```

Questa cartella verrà montata all’interno del container come `/data`, ed è qui che il workflow leggerà/scriverà i file CSV, Markdown ecc.

---

### 🚀 2. Avvia il container n8n

> ⚠️ **Non modificare il comando sottostante**: è configurato per garantire esecuzione persistente, supporto a `fs`, accesso a `/data`, e webhook tunnel con IP custom.

```bash
docker stop n8n && docker rm n8n   # ferma container esistente

docker run -d --name n8n \
  --restart unless-stopped \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  -v /home/ubuntu/n8n/jekyll:/data \
  -e N8N_SECURE_COOKIE=false \
  -e WEBHOOK_TUNNEL_URL=http://indirizzo-web:5678 \
  -e N8N_FILESYSTEM_ALLOW_LIST="/data" \
  -e NODE_FUNCTION_ALLOW_BUILTIN=fs \
  docker.n8n.io/n8nio/n8n
```

📌 **Modifica se necessario solo**:

* `-v /home/ubuntu/n8n/jekyll:/data` → sostituisci con il tuo path locale
* `WEBHOOK_TUNNEL_URL` → IP/URL pubblico del tuo server Docker

---

### 🌐 3. Accedi all’interfaccia n8n

Apri il browser su:

```
http://[TUO_IP]:5678
```

Esegui la configurazione iniziale. Dopo il primo avvio, i dati saranno salvati in volume (`n8n_data`) e la configurazione sarà persistente.

---

### 📂 4. Verifica montaggio `/data`

Per assicurarti che `/data` sia montato correttamente nel container:

```bash
docker exec -it n8n ls /data
```

Dovresti vedere i file presenti nella cartella host (es. `posts.csv`).

---

### ✅ A questo punto sei pronto per:

* Importare o costruire workflow
* Usare nodi `Read Binary File`, `Write Binary File`, ecc.
* Eseguire codice JS che legge da `/data` via `fs.readFileSync`
* Committare file via GitHub API con credenziali sicure
* Automatizzare task quotidiani

---

### 🧠 Suggerimenti avanzati

* Se usi `n8n` dietro Nginx/HTTPS, configura `WEBHOOK_TUNNEL_URL` di conseguenza (es. `https://n8n.miodominio.it`)
* Per **backup**: esegui `docker volume inspect n8n_data` e salva quel path
* Per deploy cloud: puoi usare lo stesso comando anche su **Render**, **DigitalOcean**, **Scaleway**, ecc.

---


# 🚀 Installazione n8n su Raspberry Pi 5 con DuckDNS e HTTPS

Questa guida ti permette di esporre n8n in internet con HTTPS gratuito, senza dover installare un web server. L'intera infrastruttura si basa su Docker, Docker Compose, DuckDNS e Traefik come reverse proxy.

---

## 📋 Requisiti

| Requisito                                      | Descrizione                                                  |
| ---------------------------------------------- | ------------------------------------------------------------ |
| Raspberry Pi 5                                 | Qualsiasi sistema operativo Linux (es. Raspberry Pi OS)      |
| Docker + Docker Compose                        | Installati e funzionanti                                     |
| Account su [DuckDNS](https://www.duckdns.org/) | Gratuito, per ottenere dominio dinamico                      |
| Porte 80 e 443 aperte nel NAT                  | Port forwarding configurato nel router verso il Raspberry Pi |


---

## 🧠 Obiettivo

Esporre **n8n** su internet in modo **sicuro (HTTPS)** e **stabile**, anche se l’IP pubblico cambia o se il Raspberry si riavvia.

Esempio finale di accesso:

```
https://tuodominio.duckdns.org
```

---

## 🧱 Requisiti

| Cosa            | Versione consigliata          |
| --------------- | ----------------------------- |
| Raspberry Pi    | Qualsiasi (64bit consigliato) |
| Docker          | ≥ 20.x                        |
| Docker Compose  | ≥ 1.29                        |
| DuckDNS account | ✅ creato                      |
| Traefik         | v2.11 (usato nel compose)     |

---

## 🗂️ Struttura delle cartelle consigliata

```bash
/home/antonio/Documents/n8n/
├── data/                        → dati personalizzati di n8n
├── letsencrypt/                → contiene il certificato SSL
│   └── acme.json               → certificati Let’s Encrypt
├── duck.sh                     → script aggiornamento DuckDNS
├── docker-compose.yml          → configurazione completa n8n + traefik
└── configurazione.n8n.md       → eventuali note personali
```

---

## ⚙️ Step 1 — Configura DuckDNS

### 1.1. Registra il dominio su: [https://www.duckdns.org](https://www.duckdns.org)

* Domini: `antoniotrento`
* Token: `IL_TUO_TOCKEN` (⚠️ non condividere in pubblico!)

### 1.2. Crea uno script per aggiornare l’IP ogni 5 minuti

File: `/home/antonio/Documents/n8n/duck.sh`

```bash
#!/bin/bash
echo url="https://www.duckdns.org/update?domains=antoniotrento&token=IL_TUO_TOCKEN&ip=" | curl -k -o ~/duckdns/duck.log -K -
```

Rendilo eseguibile:

```bash
chmod +x duck.sh
```

### 1.3. Aggiungi il cron job per aggiornarlo ogni 5 minuti

```bash
crontab -e
```

Aggiungi:

```bash
*/5 * * * * /home/antonio/Documents/n8n/duck.sh >/dev/null 2>&1
```

---

## 🐳 Step 2 — Docker Compose

File: `/home/antonio/Documents/n8n/docker-compose.yml`

```yaml
version: "3.8"

services:
  traefik:
    image: traefik:v2.11
    container_name: traefik
    restart: unless-stopped
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.duckdns.acme.dnschallenge=true"
      - "--certificatesresolvers.duckdns.acme.dnschallenge.provider=duckdns"
      - "--certificatesresolvers.duckdns.acme.email=lantoniotrento@gmail.com"
      - "--certificatesresolvers.duckdns.acme.storage=/letsencrypt/acme.json"
      - "--log.level=INFO"
    environment:
      - DUCKDNS_TOKEN=IL_TUO_TOCKEN
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./letsencrypt:/letsencrypt

  n8n:
    image: docker.n8n.io/n8nio/n8n
    container_name: n8n
    restart: unless-stopped
    environment:
      - N8N_SECURE_COOKIE=true
      - WEBHOOK_TUNNEL_URL=https://tuodominio.duckdns.org
      - N8N_FILESYSTEM_ALLOW_LIST=/data
      - NODE_FUNCTION_ALLOW_BUILTIN=fs
      - TZ=Europe/Rome
    volumes:
      - n8n_data:/home/node/.n8n
      - /home/antonio/n8n:/data
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.n8n.rule=Host(`tuodominio.duckdns.org`)"
      - "traefik.http.routers.n8n.entrypoints=websecure"
      - "traefik.http.routers.n8n.tls.certresolver=duckdns"
      - "traefik.http.routers.n8n.entrypoints=web,websecure"
      - "traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https"
      - "traefik.http.routers.n8n.middlewares=redirect-to-https"

volumes:
  n8n_data:
```

### 2.1. Assicurati che `acme.json` esista e abbia i permessi giusti

```bash
touch ./letsencrypt/acme.json
chmod 600 ./letsencrypt/acme.json
```

---

## 🌍 Step 3 — Configura il Port Forwarding

Sul tuo **router WindTre**, crea queste regole di NAT:

| Porta esterna  | Porta interna | IP interno   | Protocollo | Descrizione         |
| -------------- | ------------- | ------------ | ---------- | ------------------- |
| 80             | 80            | 192.168.1.23 | TCP        | HTTP → Traefik      |
| 443            | 443           | 192.168.1.23 | TCP        | HTTPS → Traefik     |
| 5678           | 5678          | 192.168.1.23 | TCP        | n8n (solo se serve) |
| 22 (opzionale) | 22            | 192.168.1.23 | TCP        | SSH                 |

---

## 🚀 Step 4 — Avvia tutto

```bash
cd /home/antonio/Documents/n8n
docker compose up -d
```

Controlla lo stato:

```bash
docker ps
```

Verifica il certificato:

```bash
curl -I https://tuodominio.duckdns.org
```

---

## 🔁 Riavvio: Cosa succede?

| Evento               | Azione automatica                  |
| -------------------- | ---------------------------------- |
| IP pubblico cambia   | DuckDNS aggiorna tramite `cron`    |
| Raspberry si riavvia | Docker riavvia `n8n` + `traefik`   |
| Certificato scade    | Traefik lo rinnova automaticamente |

---

## ✅ Checklist finale

* [x] DNS dinamico aggiornato via DuckDNS
* [x] Certificati HTTPS con Let’s Encrypt automatici
* [x] Accesso remoto sicuro via `https://tuodominio.duckdns.org`
* [x] Tutto autostart dopo reboot

---

Vuoi che te lo salvi in un file `README.md` già pronto da tenere nella cartella?


Perfetto! Ecco una **mini guida all’uso di Docker per il tuo progetto n8n con Traefik e DuckDNS** sul tuo Raspberry Pi. La guida è pensata per essere **concreta, essenziale e adatta al tuo setup attuale**.

---

## 📦 STRUTTURA DEL PROGETTO

La tua directory è:

```
/home/antonio/Documents/n8n/
│
├── docker-compose.yml       # File principale
├── duck.sh                  # Script aggiornamento IP DuckDNS
├── letsencrypt/             # Dove Traefik salva i certificati SSL
│   └── acme.json
├── data/                    # Dati n8n persistenti (workflow, credenziali)
```

---

## ⚙️ COMANDI BASE DOCKER COMPOSE

### ▶️ Avviare n8n + Traefik

```bash
cd /home/antonio/Documents/n8n
docker compose up -d
```

### ⏹️ Fermare i container

```bash
docker compose down
```

### 🔁 Riavviare tutto (es. dopo modifiche al file)

```bash
docker compose down
docker compose up -d
```

### 📝 Vedere i log live (es. errore HTTPS o avvio n8n)

```bash
docker compose logs -f
```

### 🧼 Aggiornare le immagini (manutenzione)

```bash
docker compose pull
docker compose up -d
```

---

## 🔐 SCRIPT DUCKDNS

Assicurati che lo script `duck.sh` abbia i permessi e sia in esecuzione ogni 5 minuti:

### 1. Rendi eseguibile:

```bash
chmod +x duck.sh
```

### 2. Aggiungi a `crontab`:

```bash
crontab -e
```

Aggiungi in fondo:

```
*/5 * * * * /home/antonio/Documents/n8n/duck.sh >/dev/null 2>&1
```

---

## 🌐 ACCESSO

Una volta avviato tutto, accedi al tuo n8n da **qualsiasi parte del mondo**:

🔗 `https://tuodominio.duckdns.org`

> ⚠️ Il primo caricamento può richiedere qualche secondo dopo il riavvio.

---

## 💾 BACKUP UTILI

### Backup dati n8n:

Tutto è dentro:

```
/home/antonio/n8n/      # montato in /data nel container
```

oppure:

```
docker volume inspect n8n_data
```

---

## 🛠️ RISOLUZIONE PROBLEMI

| Problema                      | Soluzione                                                |
| ----------------------------- | -------------------------------------------------------- |
| Non accedi in HTTPS           | Verifica porta 443 nel router e `acme.json` non vuoto    |
| DuckDNS non aggiorna IP       | Controlla `duck.sh` e crontab                            |
| n8n non si avvia              | `docker compose logs n8n`                                |
| Certificati non si rigenerano | Rimuovi `acme.json` e riavvia traefik (verrà rigenerato) |

---

## ✅ PRONTI E OPERATIVI!

Ora hai un setup **completo**, sicuro (HTTPS), **accessibile da remoto con DNS dinamico**, e che si **ripristina da solo** al riavvio del Raspberry o del router.

Se vuoi posso prepararti anche:

* uno script di backup automatico dei dati n8n
* una dashboard monitoring leggera per Raspberry

Fammi sapere!



Perfetto! Ecco una guida completa per **usare Cloudflare con un dominio Namecheap per ottenere DNS dinamico gratuito e HTTPS** (alternativa a DuckDNS).

---

## 🌐 Obiettivo

Collegare un dominio Namecheap a Cloudflare, aggiornare dinamicamente l’IP del tuo router con uno script (Dynamic DNS), e usare HTTPS su Raspberry Pi con Traefik + Docker.

---

## ✅ Requisiti

* Un dominio acquistato su **Namecheap**
* Un account gratuito su **Cloudflare**
* **Raspberry Pi** (o altro server Linux) con Docker + Docker Compose
* **Traefik** già configurato come reverse proxy

---

## 1. 🔁 Sposta il dominio da Namecheap a Cloudflare

### 1.1 Crea un account gratuito su Cloudflare

Vai su [https://dash.cloudflare.com](https://dash.cloudflare.com) e registrati.

### 1.2 Aggiungi il tuo dominio

1. Clic su “**Add a Site**”
2. Inserisci il tuo dominio Namecheap (es. `miodominio.it`)
3. Scegli il piano **Free**
4. Cloudflare ti mostrerà due **nameserver da usare su Namecheap**

### 1.3 Cambia nameserver su Namecheap

1. Vai su [https://www.namecheap.com](https://www.namecheap.com) e fai login
2. Vai su “**Domain List**”, clicca “Manage” accanto al dominio
3. Nella sezione “Nameservers” scegli **Custom DNS**
4. Inserisci i 2 nameserver forniti da Cloudflare
5. Salva e attendi propagazione (da 10 minuti a 24h)

---

## 2. 🔄 Configura Dynamic DNS su Raspberry

### 2.1 Crea un token su Cloudflare

1. Vai su [https://dash.cloudflare.com/profile/api-tokens](https://dash.cloudflare.com/profile/api-tokens)
2. Clic “Create Token”
3. Scegli template “**Edit zone DNS**”
4. Dai accesso solo alla zona del tuo dominio
5. Salva il token

### 2.2 Script per aggiornare l’IP

Crea il file `cloudflare-ddns.sh` sul Raspberry:

```bash
#!/bin/bash

# CONFIGURA QUESTI
CF_API_TOKEN="INSERISCI_IL_TUO_TOKEN"
ZONE_NAME="miodominio.it"
RECORD_NAME="home.miodominio.it"

# NON TOCCARE QUESTO
IP=$(curl -s https://api.ipify.org)
ZONE_ID=$(curl -s -X GET "https://api.cloudflare.com/client/v4/zones?name=$ZONE_NAME" \
     -H "Authorization: Bearer $CF_API_TOKEN" -H "Content-Type: application/json" | jq -r '.result[0].id')

RECORD_ID=$(curl -s -X GET "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records?name=$RECORD_NAME" \
     -H "Authorization: Bearer $CF_API_TOKEN" -H "Content-Type: application/json" | jq -r '.result[0].id')

curl -s -X PUT "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records/$RECORD_ID" \
     -H "Authorization: Bearer $CF_API_TOKEN" \
     -H "Content-Type: application/json" \
     --data "{\"type\":\"A\",\"name\":\"$RECORD_NAME\",\"content\":\"$IP\",\"ttl\":120,\"proxied\":false}"
```

### 2.3 Rendi eseguibile e automatizzalo

```bash
chmod +x cloudflare-ddns.sh
```

Aggiungi a `crontab`:

```bash
crontab -e
```

E inserisci:

```bash
*/5 * * * * /path/to/cloudflare-ddns.sh
```

---

## 3. 🔐 Configura HTTPS in Traefik

Nel tuo `docker-compose.yml`, modifica così (esempio):

```yaml
services:
  traefik:
    image: traefik:v2.11
    command:
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--providers.docker=true"
      - "--certificatesresolvers.cloudflare.acme.dnschallenge=true"
      - "--certificatesresolvers.cloudflare.acme.dnschallenge.provider=cloudflare"
      - "--certificatesresolvers.cloudflare.acme.email=tuo@email.it"
      - "--certificatesresolvers.cloudflare.acme.storage=/letsencrypt/acme.json"
    environment:
      - CF_API_TOKEN=IL_TUO_TOKEN
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./letsencrypt:/letsencrypt
```

Nel servizio `n8n`:

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.n8n.rule=Host(`home.miodominio.it`)"
  - "traefik.http.routers.n8n.entrypoints=websecure"
  - "traefik.http.routers.n8n.tls.certresolver=cloudflare"
```

---

## ✅ Vantaggi rispetto a DuckDNS

| DuckDNS                      | Cloudflare                        |
| ---------------------------- | --------------------------------- |
| Dominio `.duckdns.org` fisso | Dominio personalizzato            |
| Solo 1 IP/dominio            | Multi-record + TTL personalizzato |
| Meno controllo DNS           | API avanzate, DNSSEC, firewall    |
| Può chiudere in futuro       | È un colosso con base solida      |

---

## 🌟 Test finale

* Vai su `https://home.miodominio.it` → Deve funzionare e mostrare N8N
* Prova `curl -I https://home.miodominio.it` → Deve restituire `200 OK` con certificato HTTPS valido

---

Ottima domanda! Il setup che hai creato è già **funzionante e sicuro**, ma se vuoi avere **n8n al massimo delle sue funzionalità**, ecco cosa puoi aggiungere/modificare per completarlo al **100% in produzione**:

---

## ✅ **1. Email per notifiche e recupero password**

n8n supporta notifiche via email e invio mail dai workflow. Devi configurare un provider SMTP.

### Da aggiungere alla sezione `environment:` del servizio `n8n`:

```yaml
- N8N_SMTP_HOST=smtp.gmail.com
- N8N_SMTP_PORT=465
- N8N_SMTP_USER=la-tua-email@gmail.com
- N8N_SMTP_PASS=la-tua-password-o-app-password
- N8N_EMAIL_MODE=smtp
- N8N_EMAIL_FROM=la-tua-email@gmail.com
```

> ⚠️ Gmail richiede una *App password* se hai l'autenticazione a due fattori.

---

## ✅ **2. Crittografia dati sensibili**

Per proteggere credenziali/API token salvati, usa una chiave di cifratura.

Aggiungi:

```yaml
- N8N_ENCRYPTION_KEY=una-chiave-di-32-caratteri-supersegreta
```

> ⚠️ Una volta impostata, **non cambiarla mai** o perderai l'accesso ai dati criptati.

---

## ✅ **3. Utente admin e accesso protetto**

Per usare n8n con login e ruoli, attiva la modalità di autenticazione:

```yaml
- N8N_BASIC_AUTH_ACTIVE=true
- N8N_BASIC_AUTH_USER=admin
- N8N_BASIC_AUTH_PASSWORD=una-password-sicura
```

> 🔒 Questo impedisce l’accesso anonimo all’interfaccia di n8n.

---

## ✅ **4. Persistenza avanzata (es. PostgreSQL)**

Attualmente i dati sono salvati localmente nel volume Docker. Se vuoi un setup scalabile:

* Usa un database PostgreSQL per salvare i workflow e la cronologia in modo più robusto
* Esempio:

```yaml
- DB_TYPE=postgresdb
- DB_POSTGRESDB_HOST=postgres
- DB_POSTGRESDB_PORT=5432
- DB_POSTGRESDB_DATABASE=n8n
- DB_POSTGRESDB_USER=n8n
- DB_POSTGRESDB_PASSWORD=passwordsicura
```

> 🔧 Ti serve anche un servizio `postgres` nel docker-compose.

---

## ✅ **5. Backup automatici**

Implementa uno di questi:

* Script cron per salvare `/data` e `~/.n8n`
* Oppure un volume montato su un NAS/Nextcloud
* Oppure backup su S3 con un container come `restic` o `duplicity`

---

## ✅ **6. Trigger via Webhook da esterno**

Questo è **già configurato** grazie a Traefik e DuckDNS. Ma per webhook in entrata che richiedono IP statico affidabile, considera:

* usare Cloudflare (proxy + DNS failover)
* o un VPS economico per proxy reverse

---

## ✅ **7. Aggiornamento automatico container**

Per rimanere aggiornato con le nuove versioni di n8n:

* Usa [Watchtower](https://containrrr.dev/watchtower/)
* Oppure un cronjob che fa `docker pull && docker compose up -d`

---

## ✅ **8. Accesso in SSH via dominio**

Già fattibile se hai fatto il port forwarding sulla porta 22 e usi:

```bash
ssh pi@antoniotrento.duckdns.org
```

---

## ✅ **9. Log & monitoraggio**

Per produzione seria puoi:

* Usare Prometheus + Grafana per metriche (esportate da Traefik)
* Inviare log a Loggly, Datadog, Loki, ecc.

---

## ✅ **10. Disabilitare interfaccia API pubblica (opzionale)**

Per sicurezza, puoi disabilitare completamente l’API REST pubblica di n8n se non la usi.

---

## Esempio finale di `n8n` completo (env semplificato):

```yaml
environment:
  - N8N_BASIC_AUTH_ACTIVE=true
  - N8N_BASIC_AUTH_USER=admin
  - N8N_BASIC_AUTH_PASSWORD=una-password-sicura
  - N8N_SMTP_HOST=smtp.gmail.com
  - N8N_SMTP_PORT=465
  - N8N_SMTP_USER=la-tua-email@gmail.com
  - N8N_SMTP_PASS=app-password
  - N8N_EMAIL_MODE=smtp
  - N8N_EMAIL_FROM=la-tua-email@gmail.com
  - N8N_ENCRYPTION_KEY=una-chiave-super-sicura-32-caratteri
  - TZ=Europe/Rome
  - WEBHOOK_TUNNEL_URL=https://antoniotrento.duckdns.org
```

---

Eccoti un **docker-compose.yml completo e pronto all’uso** per n8n su Raspberry Pi con:

* Traefik con DuckDNS e HTTPS automatico
* n8n con autenticazione base (utente+password)
* SMTP configurato per email
* Crittografia dati con chiave sicura
* Volumi persistenti
* Variabili di ambiente essenziali

---

```yaml
version: "3.8"

services:
  traefik:
    image: traefik:v2.11
    container_name: traefik
    restart: unless-stopped
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.duckdns.acme.dnschallenge=true"
      - "--certificatesresolvers.duckdns.acme.dnschallenge.provider=duckdns"
      - "--certificatesresolvers.duckdns.acme.email=lantoniotrento@gmail.com"
      - "--certificatesresolvers.duckdns.acme.storage=/letsencrypt/acme.json"
      - "--log.level=INFO"
    environment:
      - DUCKDNS_TOKEN=d129aa4f-bd67-488e-978c-bfaec76521c6
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./letsencrypt:/letsencrypt

  n8n:
    image: docker.n8n.io/n8nio/n8n
    container_name: n8n
    restart: unless-stopped
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=una-password-sicura
      - N8N_SMTP_HOST=smtp.gmail.com
      - N8N_SMTP_PORT=465
      - N8N_SMTP_USER=la-tua-email@gmail.com
      - N8N_SMTP_PASS=app-password-o-password
      - N8N_EMAIL_MODE=smtp
      - N8N_EMAIL_FROM=la-tua-email@gmail.com
      - N8N_ENCRYPTION_KEY=una-chiave-super-sicura-32-caratteri!!!
      - N8N_FILESYSTEM_ALLOW_LIST=/data
      - NODE_FUNCTION_ALLOW_BUILTIN=fs
      - TZ=Europe/Rome
      - WEBHOOK_TUNNEL_URL=https://antoniotrento.duckdns.org
    volumes:
      - n8n_data:/home/node/.n8n
      - ./data:/data
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.n8n.rule=Host(`antoniotrento.duckdns.org`)"
      - "traefik.http.routers.n8n.entrypoints=websecure"
      - "traefik.http.routers.n8n.tls.certresolver=duckdns"
      - "traefik.http.routers.n8n.entrypoints=web,websecure"
      - "traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https"
      - "traefik.http.routers.n8n.middlewares=redirect-to-https"

volumes:
  n8n_data:
```

---

### Come usarlo

1. Metti questo file in `/home/antonio/Documents/n8n/docker-compose.yml`
2. Crea la cartella `letsencrypt` nello stesso percorso (per i certificati)
3. Crea la cartella `data` nello stesso percorso (per i dati di n8n persistenti)
4. Cambia le variabili:

   * `N8N_BASIC_AUTH_USER` e `N8N_BASIC_AUTH_PASSWORD` (metti username e password sicuri)
   * SMTP (user, pass, host, porta) se vuoi usare email
   * `N8N_ENCRYPTION_KEY`: 32 caratteri esatti, super random
   * `WEBHOOK_TUNNEL_URL` con il tuo dominio DuckDNS
   * `DUCKDNS_TOKEN` con il tuo token DuckDNS
   * `--certificatesresolvers.duckdns.acme.email` con la tua email reale
5. Avvia con:

```bash
docker compose up -d
```

---


Certo! Ecco uno **script bash semplice** per fare backup periodici dei dati persistenti di n8n (cartelle `./data` e volume Docker `n8n_data`) in un archivio `.tar.gz` con timestamp, da salvare in una cartella `backups`:

---

```bash
#!/bin/bash

# Configurazione backup
BACKUP_DIR="/home/antonio/Documents/n8n/backups"
DATA_DIR="/home/antonio/Documents/n8n/data"
TIMESTAMP=$(date +'%Y%m%d_%H%M%S')
BACKUP_NAME="n8n_backup_${TIMESTAMP}.tar.gz"

# Crea cartella backup se non esiste
mkdir -p "$BACKUP_DIR"

echo "Backup di n8n in corso..."

# Esporta il volume Docker n8n_data in una cartella temporanea
TEMP_VOL_DIR="/tmp/n8n_docker_volume_backup"
mkdir -p "$TEMP_VOL_DIR"
docker run --rm -v n8n_data:/volume -v "$TEMP_VOL_DIR":/backup busybox sh -c "cd /volume && tar czf /backup/n8n_data.tar.gz ."

# Crea archivio backup con dati volume + cartella data
tar czf "$BACKUP_DIR/$BACKUP_NAME" -C "$TEMP_VOL_DIR" n8n_data.tar.gz -C "$DATA_DIR" .

# Pulisci temporanei
rm -rf "$TEMP_VOL_DIR"

echo "Backup completato: $BACKUP_DIR/$BACKUP_NAME"
```

---

### Come usarlo

1. Salva questo script in `/home/antonio/Documents/n8n/backup_n8n.sh`
2. Rendi eseguibile:

```bash
chmod +x /home/antonio/Documents/n8n/backup_n8n.sh
```

3. Esegui manualmente o metti in cron per backup automatico:

```bash
# Apri cron tab:
crontab -e

# Esempio: backup ogni giorno alle 2 di notte
0 2 * * * /home/antonio/Documents/n8n/backup_n8n.sh >> /home/antonio/Documents/n8n/backup.log 2>&1
```

---

### Nota

* Lo script fa un backup **sia della cartella dati montata (./data)** sia del **volume Docker n8n\_data**.
* I backup saranno archivi compressi con timestamp, facili da spostare o ripristinare.

Se vuoi ti faccio anche lo script di restore!
