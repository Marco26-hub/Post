# Social Automation V1 — Pacchetto setup

Artefatti pronti per partire. Spec sorgente: `~/Downloads/workflow_automazione_social_abbigliamento_n8n_claude.md`.

## Contenuto

```
sheets_csv/         — 8 CSV da importare come fogli in Google Sheet
n8n_workflows/      — 3 JSON da importare in n8n (Import from File)
prompts/            — prompt Claude separati (riusabili)
drive_structure/    — istruzioni Drive
```

## Setup ordine (Giorno 1)

### 1. Google Drive
Creare cartella `Social_Automation_Brand/` con 9 sottocartelle (vedi `drive_structure/README.txt`). Caricare 3+ immagini prodotto test con permesso "Anyone with link".

### 2. Google Sheet
Creare Sheet `Social Automation - Brand Abbigliamento`. Per ogni CSV in `sheets_csv/`:
- File → Import → Upload CSV → "Insert new sheet(s)"
- Rinomina foglio = nome file senza `.csv`
Risultato: 8 fogli (BRAND, PRODOTTI, CALENDARIO, CAROSELLI, VIDEO_REEL, PROMPT, LOG_PUBBLICAZIONI, REPORT).

Copia SHEET_ID dall'URL (`docs.google.com/spreadsheets/d/<SHEET_ID>/edit`).

### 3. n8n
- Crea istanza (cloud o self-hosted)
- Settings → Timezone: `Europe/Rome`
- Variables: aggiungi `SHEET_ID` = <tuo id>
- Credentials da creare:
  - `GSHEETS` (Google Sheets OAuth2) — autorizza account Google
  - `BLOTATO` (HTTP Header Auth) — header `Authorization: Bearer <BLOTATO_API_KEY>`
  - `ANTHROPIC` (HTTP Header Auth) — header `x-api-key: <ANTHROPIC_API_KEY>`
- Importa i 3 JSON: Workflows → Import from File → seleziona
- Adatta i nomi credentials se diversi

### 4. Blotato
- Sottoscrivi piano
- Collega account social (IG, FB, TikTok, Pinterest, YT Shorts)
- Genera API key → salva in credential `BLOTATO`
- IMPORTANTE: verifica endpoint reale API Blotato e adatta `SOCIAL_C` nodo `Blotato Publish` (URL + payload). Lo schema usato qui è generico — controlla docs Blotato per nome campi esatti (`platform`/`accountId`/`mediaUrls`/`caption`/`scheduledAt`).

### 5. Anthropic
- Crea API key su console.anthropic.com → credential `ANTHROPIC`
- Modello default workflow: `claude-sonnet-4-5`

## Test V1 (cuore sistema)

1. Riga test già pronta in `sheets_csv/CALENDARIO.csv` (C001, status=APPROVATO). Aggiorna `data_pubblicazione`/`ora_pubblicazione` a ora corrente + 2 min e sostituisci `link_media_1` con URL Drive reale.
2. Apri workflow `SOCIAL_C_PUBBLICA_APPROVATI` in n8n → Activate
3. Click "Execute Workflow" per test immediato
4. Verifica:
   - Status CALENDARIO: APPROVATO → IN_PUBBLICAZIONE → PUBBLICATO
   - `blotato_post_id` popolato
   - Riga in LOG_PUBBLICAZIONI
   - Post visibile su IG profile

## Test errore
Riga con `link_media_1` vuoto → filtro la esclude (resta APPROVATO, non passa). Per testare path ERRORE: passa caption valida ma URL media non scaricabile → Blotato risponde error → status=ERRORE + log.

## Anti-doppione
Lock `IN_PUBBLICAZIONE` settato PRIMA della chiamata Blotato. Filtro iniziale richiede `status==APPROVATO` quindi righe in IN_PUBBLICAZIONE/PUBBLICATO non rientrano.

## Roadmap

- Giorno 1: workflow C live + 1 post test
- Giorno 2-3: attiva B (gen contenuti da IDEA) + A (piano settimanale)
- Settimana 2: aggiungi D (caroselli), E (reel), F (report), G (Telegram)
- Mese 2: V2 — import Shopify/Woo, video auto, analytics

## Note critiche

- **Blotato API schema**: payload usato è ipotetico. Conferma su docs Blotato e aggiorna nodo HTTP `Blotato Publish` in workflow C.
- **Drive URL format**: `https://drive.google.com/uc?id=FILE_ID` funziona per immagini singole pubbliche. Per video pesanti considera CDN (Cloudinary, Bunny.net).
- **Quota Anthropic**: monitora token usage. Sonnet ~$3/M input, $15/M output.
- **Approvazione manuale**: cambia `status` a `APPROVATO` direttamente in Sheet, niente UI custom in V1.
