![n8n → Jekyll workflow](images/n8n-workflow.png)

# jekyll-recipe-ai  
Ricettario statico basato su **Jekyll** alimentato da un workflow **n8n** che genera, programma e pubblica articoli tramite **OpenAI** a partire da un semplice file CSV.

---

## 🗂️ Cosa troverai in questo repository

| Percorso              | Contenuto                                                           |
|-----------------------|---------------------------------------------------------------------|
| `_config.yml`         | Configurazione Jekyll (tema minima, lingua IT, future posts off)    |
| `Gemfile`             | Dipendenze per lo sviluppo locale (`github-pages`)                  |
| `_layouts/`           | Layout HTML minimi (`default`, `post`)                              |
| `_includes/`          | Header comune                                                       |
| `index.html`          | Homepage che elenca tutti i post                                    |
| `_posts/`             | Cartella in cui il workflow n8n committa i markdown delle ricette   |
| `images/n8n-workflow.png` | Diagramma del flusso (visualizzato qui sopra)                   |
| `_n8n/`               | JSON del workflow n8n per import rapido                           |

---

## ⚙️ Come funziona

1. **CSV di input** – L’editor inserisce (o aggiorna) le righe in `posts.csv` con:  
   `titolo;prompt_descrizione;keyword_principale;keyword_secondarie;data_pubblicazione`
2. **n8n Workflow**  
   - Legge il CSV e processa una riga alla volta  
   - Usa GPT-4o-mini per scrivere ~800-1000 parole (no front-matter)  
   - Costruisce il file Markdown + front-matter Jekyll  
   - Si mette in pausa fino alla `data_pubblicazione`  
   - Effettua il commit su `_posts/` del ramo `main` tramite PAT GitHub  
   - (opzionale) Pubblica un tweet e un post LinkedIn col link alla ricetta
3. **GitHub Pages** ricostruisce il sito, rendendo live la nuova pagina di ricetta.

---

## 📋 Schema CSV

| Colonna              | Descrizione                                                   | Esempio                                  |
|----------------------|---------------------------------------------------------------|------------------------------------------|
| `titolo`             | Titolo leggibile                                              | Risotto ai Funghi Porcini                |
| `prompt_descrizione` | Brief per il copywriter AI                                    | “Ricetta tradizionale… tecniche crema”   |
| `keyword_principale` | Keyword SEO primaria                                          | risotto ai funghi porcini                |
| `keyword_secondarie` | Keyword secondarie (virgola-separate)                         | risotto cremoso, funghi porcini freschi  |
| `data_pubblicazione` | Data/ora ISO 8601 (timezone Europe/Rome)                      | 2025-06-17 10:00                         |

---

## 🚀 Avvio rapido

```bash
# 1. Clona il repo
git clone https://github.com/antonio-backend-projects/jekyll-recipe-ai.git
cd jekyll-recipe-ai

# 2. (facoltativo) Servi il sito in locale
bundle install          # richiede Ruby + Bundler
bundle exec jekyll serve
```

### Impostare n8n

1. Importa `workflow/jekyll-recipe-workflow.json` (oppure crea il flusso a mano).
2. Crea queste credenziali in **n8n**:

   * **GitHub OAuth2 API** – PAT con scope `repo` su `antonio-backend-projects`.
   * **OpenAI API** – chiave valida.
   * (opz.) Twitter / LinkedIn token per i nodi social.
3. Aggiorna nel nodo GitHub:

   * **Owner:** `antonio-backend-projects`
   * **Repo:** `jekyll-recipe-ai`
4. Metti il CSV nel percorso indicato dal nodo “Read CSV”.
5. Avvia manualmente o pianifica un trigger.

---

## 🛠️ Personalizzazioni

| Esigenza                   | Modifica                                              |
| -------------------------- | ----------------------------------------------------- |
| Aggiungere tag / categorie | Estendi il front-matter nel nodo “Build Markdown”.    |
| Cambiare struttura URL     | Adatta `markdownPath` o `permalink` in `_config.yml`. |
| Disabilitare social        | Scollega / rimuovi i nodi Twitter & LinkedIn.         |
| Pubblicare anche immagini  | Riattiva i nodi immagine nel workflow n8n.            |

---


run


```bash
docker stop n8n && docker rm n8n   # ferma e rimuove il vecchio container

docker run -d --name n8n \
  --restart unless-stopped \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  -v /home/ubuntu/n8n/jekyll:/data \
  -e N8N_SECURE_COOKIE=false \
  -e WEBHOOK_TUNNEL_URL=http://51.75.133.84:5678 \
  -e N8N_FILESYSTEM_ALLOW_LIST="/data" \
  -e NODE_FUNCTION_ALLOW_BUILTIN=fs \
  docker.n8n.io/n8nio/n8n
```

## 🤝 Contributi

1. Fai fork e crea un branch descrittivo (`feature/nuova-ricetta`).
2. Commit & push; apri una **pull request**.
3. Un mantenitore revisionerà il contenuto (test kitchen-approved!).

---

## Licenza

Questo progetto è distribuito con licenza **MIT**. Vedi `LICENSE` per i dettagli.
