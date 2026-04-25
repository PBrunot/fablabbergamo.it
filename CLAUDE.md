# CLAUDE.md — FabLab Bergamo

Guida editoriale e tecnica per la creazione e modifica di articoli sul sito **www.fablabbergamo.it** (WordPress + Yoast SEO).

---

## QUICK REFERENCE — Comandi API pronti all'uso

```bash
# Setup (copia-incolla in shell prima di iniziare)
WP_BASE="https://www.fablabbergamo.it/wp-json/wp/v2"
WP_USER="pascal"
WP_PASS="$(cat /home/pascal/fablabbergamo.it/.secret | tr -d '\n')"
wp_get()  { curl -s --user "${WP_USER}:${WP_PASS}" "${WP_BASE}/$1"; }
wp_patch(){ curl -s -X PATCH --user "${WP_USER}:${WP_PASS}" \
              -H "Content-Type: application/json" -d "$2" "${WP_BASE}/$1"; }
wp_post() { curl -s -X POST  --user "${WP_USER}:${WP_PASS}" \
              -H "Content-Type: application/json" -d "$2" "${WP_BASE}/$1"; }
```

```bash
# Leggere articoli
wp_get "posts?per_page=10&_fields=id,title,slug,date,status"     # lista recenti
wp_get "posts?search=KEYWORD&_fields=id,title,slug"              # cerca per keyword
wp_get "posts/ID?context=edit&_fields=id,title,content,excerpt,slug,yoast_head_json"  # articolo completo

# Aggiornare contenuto
wp_patch "posts/ID" '{"title":"Nuovo titolo","content":"<p>HTML...</p>","excerpt":"Estratto..."}'

# Cambiare stato
wp_patch "posts/ID" '{"status":"publish"}'   # pubblica
wp_patch "posts/ID" '{"status":"draft"}'     # porta in bozza

# Creare nuovo articolo in bozza
wp_post "posts" '{"title":"Titolo","content":"<p>...</p>","status":"draft","categories":[ID]}'

# Categorie e tag
wp_get "categories?per_page=50&_fields=id,name,slug"
wp_get "tags?per_page=100&_fields=id,name,slug"
```

```python
# Python helper per contenuti lunghi (evita problemi di escaping bash)
import requests, base64
from pathlib import Path

WP_PASS = Path("/home/pascal/fablabbergamo.it/.secret").read_text().strip()
_tok = base64.b64encode(f"pascal:{WP_PASS}".encode()).decode()
_hdrs = {"Authorization": f"Basic {_tok}", "Content-Type": "application/json"}
_base = "https://www.fablabbergamo.it/wp-json/wp/v2"

def wp_get(path):
    return requests.get(f"{_base}/{path}", headers=_hdrs).json()

def wp_update(post_id, **fields):
    r = requests.patch(f"{_base}/posts/{post_id}", headers=_hdrs, json=fields)
    r.raise_for_status()
    return r.json()

# Uso:
# post = wp_get("posts/5335?context=edit")
# wp_update(5335, title="Nuovo titolo", content="<p>...</p>", excerpt="...")
```

**Workflow tipico "sviluppa articolo":**
1. `wp_get "posts?search=TITOLO&_fields=id,title,slug"` → trova ID
2. `wp_get "posts/ID?context=edit"` → leggi contenuto attuale
3. Scrivi/espandi il testo in HTML rispettando §2 e §3
4. `wp_patch "posts/ID" '{"content":"<p>...</p>"}'` → aggiorna
5. Apri nel browser per verifica visiva
6. Aggiorna focus keyphrase + meta description in wp-admin (Yoast)

---

## 1. Missione del sito

**FabLab Bergamo APS** è uno spazio maker a Bergamo con il motto *"Costruirsi (quasi) qualsiasi cosa"*. Il sito ha due obiettivi:

1. **Attrarre nuovi maker**: far conoscere il laboratorio, gli strumenti (stampanti 3D, CNC, elettronica, moda/design) e la community.
2. **Documentare i progetti**: articoli tecnici e divulgativi che raccontano cosa si fa in FabLab, per fidelizzare la community esistente e farsi trovare su Google e AI.

L'orario di apertura (martedì, giovedì, sabato) e il link al **tesseramento** sono le CTA principali del sito.

---

## 2. Linee editoriali

### 2.1 Tono e voce

- **Didattico-entusiasta**: spiega come un collega appassionato che vuole trasmettere la gioia di fare. Non cattedratico.
- **Inclusivo e plurale**: usa "noi" e "vi aspettiamo in Lab". Il lettore è parte della community, non un utente passivo.
- **Accessibile ma non banale**: i concetti tecnici vanno spiegati (con analogie, tabelle, codice annotato) ma senza scendere sotto la dignità intellettuale del lettore.
- **Italiano corretto**: lingua italiana con accenti e apostrofi corretti. I termini tecnici anglofoni si lasciano in inglese (*breadboard*, *inference*, *object detection*) ma si spiegano al primo utilizzo.
- **Emoji con parsimonia**: solo per enfatizzare un passaggio chiave o una nota ironica, non come decorazione.

### 2.2 Target

| Pubblico | Caratteristiche | Come scrivere |
|---|---|---|
| Maker principianti | Curiosi, poche basi tecniche | Spiega acronimi, usa analogie quotidiane |
| Maker intermedi | Sanno usare Arduino/RPi, vogliono approfondire | Entra nel dettaglio tecnico, mostra codice |
| Esperti / ricercatori | Leggono per ispirazione o benchmarking | Dati precisi, riferimenti a librerie/paper |

La maggior parte degli articoli si rivolge a **maker intermedi**, con un'introduzione comprensibile anche ai principianti.

### 2.3 Struttura tipo di un articolo

```
H1: Titolo SEO (= keyword principale + promessa di valore)
    └── Paragrafo introduttivo (2-4 righe): aggancia il lettore con un problema o una curiosità

H2: Contesto / "Cos'è X?"
H2: Come funziona (teoria minima necessaria)
H2: Tutorial / Procedura passo-passo
    H3: Passo 1 — ...
    H3: Passo 2 — ...
H2: Risultati / Performance / Demo
H2: Conclusione e prossimi passi
    └── CTA: "Vieni in FabLab" oppure link alla serie successiva
```

Per **serie di articoli** (es. Fab-O-MAtic), ogni post si chiude con un rimando al successivo e si apre con un brevissimo richiamo al precedente.

### 2.4 Lunghezza

| Tipo articolo | Target parole |
|---|---|
| Annuncio evento / news | 200–400 |
| Articolo divulgativo | 800–1.200 |
| Tutorial tecnico completo | 1.200–2.500 |
| Puntata di una serie | 700–1.000 (più link agli altri) |

### 2.5 Elementi ricorrenti

- **Immagini**: almeno 1 immagine per sezione H2. La featured image deve essere significativa e avere alt text con la keyword.
- **Blocchi di codice**: formattati con il tag `<pre><code>`, annotati in italiano dove possibile.
- **Tabelle**: usare per confronti (es. modelli, performance, librerie).
- **Link interni**: sempre linkare articoli correlati dello stesso sito.
- **Link esterni**: GitHub, documentazione ufficiale, video YouTube per approfondimenti.
- **CTA finale**: ogni articolo si chiude con un invito esplicito (venire in FabLab, iscriversi, collaborare, visitare il repo).

### 2.6 Cosa evitare

- Toni promozionali autoreferenziali ("siamo i migliori...").
- Paragrafi troppo lunghi: spezza dopo 4-5 righe.
- Introduzioni generiche ("Nel mondo di oggi la tecnologia...").
- Ripetere il titolo esatto nel primo paragrafo — riformula.
- Elenchi puntati al posto di testo narrativo quando si racconta una storia.

---

## 3. Linee guida SEO (Yoast SEO)

### 3.1 Titolo (H1 / SEO title)

- **Formato consigliato**: `[Keyword principale]: [Sottotitolo descrittivo]`
  - Esempio: `Contatore Geiger IoT: Guida con ESPHome e Home Assistant`
- Lunghezza: 50–60 caratteri (Yoast mostra l'indicatore verde).
- La keyword principale deve essere nelle prime parole.
- NON ripetere la keyword esatta più di 2 volte nel titolo.

### 3.2 Focus Keyphrase (Yoast)

- Una frase di 2–4 parole che descrive esattamente il contenuto.
- Esempi: `contatore Geiger ESP32`, `Hailo-8L object detection`, `Fab-O-MAtic RFID Arduino`.
- Deve comparire in: titolo H1, primo paragrafo, almeno un H2, slug URL, meta description.

### 3.3 Meta Description

- Lunghezza: 120–155 caratteri.
- Deve contenere la focus keyphrase.
- Formula efficace: `[Cosa impari/fai] + [con quale strumento] + [beneficio/risultato concreto]`
  - Esempio: *"Scopri come trasformare il kit Geiger CAJOE in un sensore IoT permanente con ESPHome e Home Assistant su Raspberry Pi 5."*

### 3.4 Struttura URL (slug)

- Formato: `keyword-principale-breve` (kebab-case, tutto minuscolo, no articoli/preposizioni).
- Esempi: `fabomatic1`, `hailo-object-detection-raspberry-pi`, `contatore-geiger-esphome`.

### 3.5 Headings e keyword density

- La focus keyphrase deve comparire in almeno **1 H2**.
- Usa variazioni semantiche (LSI): se la keyphrase è `contatore Geiger`, usa anche `rilevatore di radioattività`, `sensor IoT`, `CPM`.
- Densità consigliata da Yoast: 0.5%–2.5% del testo.

### 3.6 Immagini

- **Alt text**: descrizione letterale + keyword se pertinente.
  - Esempio: `Schema elettrico ESP32 con tubo Geiger J305 per monitoraggio radioattività`
- **Nome file**: `keyword-descrittiva.jpg` (no spazi, no caratteri speciali).
- **Featured image**: obbligatoria, dimensione consigliata 1200×628px (formato OG).

### 3.7 Link interni

- Almeno 2 link interni per articolo.
- Usa anchor text descrittivi, non "clicca qui".
  - Sì: `l'articolo sul prototipo breadboard Fab-O-MAtic`
  - No: `clicca qui per saperne di più`

### 3.8 Readability Yoast

- Frasi: max 20 parole in media.
- Paragrafi: max 150 parole.
- Parole di transizione (Yoast le conta): usa "inoltre", "tuttavia", "in conclusione", "di conseguenza".
- Distribuzione H2/H3: un heading ogni 300 parole circa.
- Evita la forma passiva quando possibile.

---

## 4. WordPress API — Setup

### 4.1 Credenziali

```
Base URL : https://www.fablabbergamo.it
Username : pascal
Password : vedi file .secret nella root del progetto
Endpoint : /wp-json/wp/v2/
```

> **Nota**: il dominio senza `www` fa un redirect 301 che strip le credenziali Basic Auth. Usa sempre `https://www.fablabbergamo.it`.

### 4.2 Helper function (bash)

Aggiungi al tuo ambiente o usa inline:

```bash
WP_BASE="https://www.fablabbergamo.it/wp-json/wp/v2"
WP_USER="pascal"
WP_PASS="$(cat /home/pascal/fablabbergamo.it/.secret | tr -d '\n')"

wp_get()  { curl -s --user "${WP_USER}:${WP_PASS}" "${WP_BASE}/$1"; }
wp_patch(){ curl -s -X PATCH --user "${WP_USER}:${WP_PASS}" \
              -H "Content-Type: application/json" \
              -d "$2" "${WP_BASE}/$1"; }
```

### 4.3 Operazioni comuni

**Leggere un post per ID:**
```bash
wp_get "posts/5335" | python3 -m json.tool | grep -E '"title|"slug|"status"'
```

**Leggere lista articoli recenti:**
```bash
wp_get "posts?per_page=10&_fields=id,title,slug,date,status" | python3 -m json.tool
```

**Cercare un post per titolo/slug:**
```bash
wp_get "posts?search=geiger&_fields=id,title,slug,date" | python3 -m json.tool
```

**Leggere il contenuto completo (per editing):**
```bash
wp_get "posts/5335?context=edit&_fields=id,title,content,excerpt,slug,status,meta,yoast_head_json" \
  | python3 -m json.tool
```

**Aggiornare titolo e contenuto:**
```bash
wp_patch "posts/5335" '{
  "title": "Nuovo titolo articolo",
  "content": "<p>Contenuto HTML aggiornato...</p>"
}'
```

**Aggiornare excerpt:**
```bash
wp_patch "posts/5335" '{"excerpt": "Breve descrizione dell'\''articolo per gli archivi."}'
```

**Pubblicare una bozza:**
```bash
wp_patch "posts/5335" '{"status": "publish"}'
```

### 4.4 Yoast SEO via API

I campi Yoast (`_yoast_wpseo_focuskw`, `_yoast_wpseo_metadesc`, `_yoast_wpseo_title`) **non sono esposti in scrittura** di default dalla REST API. Per abilitarli, aggiungere al tema `functions.php`:

```php
add_action('init', function() {
    foreach (['_yoast_wpseo_focuskw','_yoast_wpseo_metadesc','_yoast_wpseo_title'] as $key) {
        register_post_meta('post', $key, [
            'show_in_rest' => true, 'single' => true, 'type' => 'string'
        ]);
    }
});
```

Dopo questa modifica, aggiornare i metadati Yoast via:
```bash
wp_patch "posts/5335" '{
  "meta": {
    "_yoast_wpseo_focuskw": "contatore Geiger ESP32",
    "_yoast_wpseo_metadesc": "Trasforma il kit Geiger CAJOE in sensore IoT con ESPHome su Raspberry Pi 5.",
    "_yoast_wpseo_title": "Contatore Geiger IoT: Guida ESPHome %%sep%% FabLab Bergamo"
  }
}'
```

Fino all'abilitazione, i metadati Yoast si modificano **manualmente nell'editor WordPress** dopo l'aggiornamento del contenuto via API.

### 4.5 Struttura JSON risposta (campi utili)

```json
{
  "id": 5335,
  "slug": "arduino-tostato-...",
  "status": "publish",
  "title": { "rendered": "Titolo HTML", "raw": "Titolo raw" },
  "content": { "rendered": "<p>HTML...</p>", "raw": "HTML grezzo" },
  "excerpt": { "rendered": "<p>...</p>", "raw": "..." },
  "yoast_head_json": {
    "title": "Titolo SEO attuale",
    "description": "Meta description attuale",
    "og_image": [{ "url": "https://..." }]
  },
  "meta": {}
}
```

---

## 5. Workflow: "Sviluppa il testo di questo articolo"

Quando ricevi il task **"sviluppa il testo di questo articolo"**, segui questo protocollo:

### Step 1 — Recupera l'articolo

```bash
wp_get "posts/<ID>?context=edit&_fields=id,title,content,excerpt,slug,yoast_head_json"
```

Se non conosci l'ID, cerca per slug o titolo:
```bash
wp_get "posts?search=<parola-chiave>&_fields=id,title,slug"
```

### Step 2 — Fai le domande giuste

Prima di scrivere, chiedi (se non è già chiaro dal contesto):

1. **Pubblico target**: principianti, maker intermedi, esperti?
2. **Obiettivo dell'articolo**: tutorial pratico, racconto di un progetto, annuncio evento?
3. **Focus keyphrase SEO**: quale keyword vuoi posizionare?
4. **Lunghezza desiderata**: breve (~500 parole), medio (~1.000), lungo (~2.000)?
5. **Tono**: più tecnico o più narrativo/divulgativo?
6. **C'è codice/hardware da documentare?**: se sì, quali snippet includere?
7. **CTA principale**: venire in FabLab, GitHub, iscrizione newsletter?

### Step 3 — Scrivi o espandi il contenuto

Rispetta le linee editoriali (§2) e SEO (§3). Genera HTML semantico valido per WordPress:
- Usa `<h2>`, `<h3>` (non `<h1>` nel corpo — c'è già il titolo del post).
- Wrappa ogni paragrafo in `<p>`.
- Codice in `<pre><code class="language-python">...</code></pre>`.
- Grassetto con `<strong>` per termini tecnici al primo utilizzo.

### Step 4 — Aggiorna il post via API

```bash
wp_patch "posts/<ID>" "{
  \"content\": \"$(echo '<p>Contenuto...</p>' | sed 's/"/\\"/g')\"
}"
```

Per contenuti lunghi, usa Python per evitare problemi di escaping:

```python
import requests, base64, json
from pathlib import Path

WP_PASS = Path("/home/pascal/fablabbergamo.it/.secret").read_text().strip()
token = base64.b64encode(f"pascal:{WP_PASS}".encode()).decode()
headers = {"Authorization": f"Basic {token}", "Content-Type": "application/json"}

def wp_update(post_id, **fields):
    r = requests.patch(
        f"https://www.fablabbergamo.it/wp-json/wp/v2/posts/{post_id}",
        headers=headers, json=fields
    )
    r.raise_for_status()
    return r.json()

# Esempio
wp_update(5335,
    title="Nuovo titolo",
    content="<p>Contenuto sviluppato...</p>",
    excerpt="Breve descrizione per archivi."
)
```

### Step 5 — Verifica e SEO checklist

Dopo l'aggiornamento, verifica nella risposta:
- [ ] `title.rendered` corretto
- [ ] `content.rendered` non contiene HTML rotto
- [ ] `status` è `publish` (o `draft` se voluto)
- [ ] Apri l'articolo nel browser per controllo visivo
- [ ] Aggiorna manualmente in wp-admin: focus keyphrase, meta description (finché non abilitato via API)

---

## 6. Workflow: "Crea un nuovo articolo"

### Step 1 — Raccolta info (domande obbligatorie)

1. Titolo provvisorio?
2. Categoria WordPress? (vedi lista: `wp_get "categories?_fields=id,name,slug"`)
3. Focus keyphrase SEO?
4. C'è una featured image da caricare?

### Step 2 — Crea la bozza

```bash
wp_patch "posts" '{
  "title": "Titolo articolo",
  "content": "<p>Contenuto...</p>",
  "excerpt": "Meta description / excerpt",
  "status": "draft",
  "categories": [ID_CATEGORIA]
}'
```

> Usa `status: "draft"` durante la stesura, `"publish"` solo a revisione completata.

---

## 7. Categorie e tag principali

Recupera la lista aggiornata con:
```bash
wp_get "categories?per_page=50&_fields=id,name,slug" | python3 -m json.tool
wp_get "tags?per_page=100&_fields=id,name,slug" | python3 -m json.tool
```

Categorie tipiche del sito: **Progetti**, **Tutorial**, **Eventi**, **Fab-O-MAtic**, **AI/Machine Learning**, **Arduino**, **Elettronica**.

---

## 8. Note tecniche WordPress

- **Dominio**: sempre `https://www.fablabbergamo.it` (con `www`)
- **Autenticazione**: Basic Auth con Application Password (vedi `.secret`)
- **Editor**: Gutenberg (blocchi) — via API il contenuto va come HTML grezzo
- **Yoast**: versione 27.4 — campi SEO in sola lettura via API (vedi §4.4 per abilitarli in scrittura)
- **Rate limiting**: non configurato lato server, ma evita burst di molte richieste in pochi secondi

---

*Aggiornato: 2026-04-25*
