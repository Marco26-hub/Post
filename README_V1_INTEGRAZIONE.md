# Social Automation V1+ — Integrazione parti mancanti

Estende V1 con: SETTINGS, ACCOUNT_SOCIAL, PROMO, BACKUP_LOG, lock anti-doppione, dry_run, validazioni umane/media/stock/promo, retry, UTM auto, error handler, backup.

## Nuovi fogli da importare

```
sheets_csv/SETTINGS.csv
sheets_csv/ACCOUNT_SOCIAL.csv
sheets_csv/PROMO.csv
sheets_csv/BACKUP_LOG.csv
```

Compila `ACCOUNT_SOCIAL` con `platform_account_id` reali da Blotato (sostituisci `*_REPLACE_ME`).

## Fogli aggiornati (sovrascrivere)

- `PRODOTTI.csv` — aggiunte colonne: `prodotto_attivo, stock_status, stock_quantity, data_ultimo_controllo_stock`
- `CALENDARIO.csv` — aggiunte colonne: `platform_account_id, publish_lock_id, media_type, media_validato, retry_count, last_retry_at, errore_tecnico, checked_copy, checked_media, checked_link, checked_price, checked_by, checked_at, utm_source, utm_medium, utm_campaign, utm_content, link_prodotto_finale, promo_id, promo_codice, promo_validata, fonte_media, consenso_utilizzo`

Strategia consigliata: cancella foglio esistente in Sheets e re-importa CSV aggiornato (mantieni le righe vecchie in backup).

## Workflow aggiornati / nuovi

| File | Stato | Note |
|---|---|---|
| `SOCIAL_C_PUBBLICA_APPROVATI.json` | **AGGIORNATO** | Tutte validazioni + lock + dry_run + retry + UTM |
| `SOCIAL_H_ERROR_HANDLER.json` | NUOVO | Error workflow globale + Telegram |
| `SOCIAL_I_BACKUP_GIORNALIERO.json` | NUOVO | CSV daily 03:00 in Drive |
| `SOCIAL_J_VALIDA_MEDIA.json` | NUOVO | HEAD check media → `media_validato=SI/NO` |

## Variabili n8n richieste

```
SHEET_ID = <id Google Sheet>
BACKUP_FOLDER_ID = <id cartella Drive 10_Backup>
TELEGRAM_CHAT_ID = <chat id>
```

## Credentials n8n richieste

- `GSHEETS` (Google Sheets OAuth2)
- `GDRIVE` (Google Drive OAuth2) — per backup
- `BLOTATO` (HTTP Header Auth)
- `ANTHROPIC` (HTTP Header Auth)
- `TELEGRAM` (Telegram Bot API)

## Configurazione error workflow

In n8n: **Settings → Workflows → Error Workflow** = `SOCIAL_H_ERROR_HANDLER` (default per tutti).

Oppure su ogni workflow → Settings tab → Error workflow.

## Logica workflow C aggiornato (sintesi)

```
Schedule 15min
  ↓ (parallelo) Read SETTINGS / CALENDARIO / ACCOUNT_SOCIAL / PRODOTTI / PROMO
  ↓
Validate All (Code Node)
  - controlla automation_enabled
  - filtra status=APPROVATO + publish_lock_id vuoto + data/ora valide
  - valida campi obbligatori
  - valida revisione umana (checked_copy/media/link)
  - valida media_validato
  - valida consenso (se fonte sensibile)
  - lookup ACCOUNT_SOCIAL → account attivo + formato consentito
  - lookup PRODOTTI → prodotto_attivo + stock
  - lookup PROMO → date/canali/prodotti
  - build UTM link_prodotto_finale
  - genera publish_lock_id
  ↓
IF Valid
  ├ NO → status=ERRORE + errore_tecnico=lista errori
  └ SI → Lock + UTM (status=IN_PUBBLICAZIONE, salva lock + UTM + account_id)
         ↓
       IF Dry Run
         ├ SI → Log DRY_RUN_OK
         └ NO → Blotato Publish
                ↓
              IF Success
                ├ SI → status=PUBBLICATO + blotato_post_id + Log
                └ NO → Compute Retry
                       - retry_count < max → status=APPROVATO (riprova)
                       - retry_count >= max → status=ERRORE_MANUALE
                       ↓
                     Mark Retry/Errore + Log
```

## Test

1. Imposta `SETTINGS.dry_run=TRUE` → run C → verifica log `DRY_RUN_OK`, niente post reale
2. `dry_run=FALSE` + riga C001 con tutti `checked_*=SI` e `media_validato=SI` → pubblica
3. Forza errore: cambia `link_media_1` con URL non raggiungibile → run C → `ERRORE` + `errore_tecnico` popolato
4. Test retry: `max_retry=2`. Riga fallisce → torna APPROVATO con retry_count=1 → fallisce ancora → ERRORE_MANUALE
5. Test stock: `prodotto_attivo=NO` su P001 → run C → status=ERRORE con "prodotto non attivo"
6. Test promo: imposta `promo_id=PR001` su riga e `promo_attiva=NO` → ERRORE
7. Test J: aggiungi riga DA_APPROVARE → run J → `media_validato` aggiornato
8. Test I: run manuale → CSV in cartella Drive + riga in BACKUP_LOG

## Regola finale aggiornata

```
Claude propone.
Umano controlla (checked_copy/media/link/price = SI).
Google Sheet approva (status = APPROVATO).
n8n valida (SETTINGS, ACCOUNT_SOCIAL, PRODOTTI, PROMO, media).
n8n blocca la riga (IN_PUBBLICAZIONE + publish_lock_id).
Blotato pubblica.
n8n registra (LOG_PUBBLICAZIONI + blotato_post_id).
Errore → retry o ERRORE_MANUALE.
SOCIAL_H_ERROR_HANDLER cattura crash workflow.
```

## Note critiche

- **Schema Blotato**: payload `accountId/platform/mediaType/mediaUrls/caption` è ipotetico. Verifica docs Blotato e adatta nodo `Blotato Publish` se diverso.
- **HEAD request su Drive**: Google Drive può non rispondere correttamente a HEAD su `uc?id=`. Se fallisce, sostituire con GET range=0-0 in `SOCIAL_J`.
- **Telegram opzionale**: rimuovi nodo se non usi bot.
- **Backup Drive folder**: crea manualmente `Social_Automation_Brand/10_Backup/` e copia ID in `BACKUP_FOLDER_ID`.
