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

Se vuoi ti posso generare automaticamente i file pronti per Cloudflare, oppure aiutarti a trasferire direttamente la configurazione passo-passo. Vuoi che prepari qualcosa?
