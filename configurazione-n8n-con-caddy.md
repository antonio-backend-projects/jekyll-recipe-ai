n8n/Caddy con namecheap


# Namecheap configurazion

Su namecheap su manage advanced DNS del proprio dominio c'è una sezione dynamic DNS in basso

|Type|Host|Value|TTL|
|A + Dynamic DNS Record|n8n(o il proprio sottodominio @ se dominio di primo livello)|proprio ip del router|Automatic|

Creare un file di invio dati a nameheap sul proprio raspberry da inserire nel cron ogni 5 min

Si trova la password su namecheap 


Dynamic DNS

Dynamic DNS Password

``` bash
#!/bin/bash
curl -s "https://dynamicdns.park-your-domain.com/update?host=n8n&domain=antoniotrento.net&password=PASSWORDCHE-TROVI-SU-NAMECHEAP" > /dev/null
```

Ora impostare il cron per mantenere connesso il dns al router anmche se si riavvia e cambia IP

``` bash
crontab -e
```
Inserire la righa in basso

``` bash
# Edit this file to introduce tasks to be run by cron.
#
# Each task to run has to be defined through a single line
# indicating with different fields when the task will be run
# and what command to run for the task
#
# To define the time you can provide concrete values for
# minute (m), hour (h), day of month (dom), month (mon),
# and day of week (dow) or use '*' in these fields (for 'any').
#
# Notice that tasks will be started based on the cron's system
# daemon's notion of time and timezones.
#
# Output of the crontab jobs (including errors) is sent through
# email to the user the crontab file belongs to (unless redirected).
#
# For example, you can run a backup of all your user accounts
# at 5 a.m every week with:
# 0 5 * * 1 tar -zcf /var/backups/home.tgz /home/
#
# For more information see the manual pages of crontab(5) and cron(8)
#
# m h  dom mon dow   command
#*/5 * * * * ~/duckdns/duck.sh >/dev/null 2>&1
#*/5 * * * * /root/duckdns/duck.sh >/dev/null 2>&1

*/5 * * * * /home/antonio/Documents/n8n/update-namecheap-ddns.sh >> /home/antonio/Documents/n8n/ddns.log 2>&1
```

# docker-compose.yml (ha sestension)

``` json
version: "3.8"

services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    container_name: n8n
    restart: unless-stopped
    environment:
      - N8N_SECURE_COOKIE=true
      - WEBHOOK_URL=https://n8n.antoniotrento.net
      - N8N_FILESYSTEM_ALLOW_LIST=/data
      - NODE_FUNCTION_ALLOW_BUILTIN=fs
      - TZ=Europe/Rome
      - N8N_RUNNERS_ENABLED=true
    volumes:
      - n8n_data:/home/node/.n8n
      - /home/antonio/n8n:/data
    expose:
      - "5678" # porta interna per Caddy

  caddy:
    image: caddy:2
    container_name: caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
      - caddy_config:/config

volumes:
  n8n_data:
  caddy_data:
  caddy_config:
```


# Caddyfile (non ha estension)

``` bash
{
  email antonio@tuaemail.com
}

n8n.antoniotrento.net {
  reverse_proxy n8n:5678
}
```