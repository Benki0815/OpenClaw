# OpenClaw – Aufgaben für Benki (manuell)

> Aufgaben, die Benki manuell erledigen muss.
> Copilot kann diese nicht automatisch ausführen (Accounts, Keys, Bestätigungen etc.).

---

## Status-Legende

- ⬜ Offen
- ✅ Erledigt

---

## Vor dem Setup

### B-01: OpenRouter API-Key besorgen ⬜

1. Gehe zu https://openrouter.ai
2. Erstelle einen Account (falls noch nicht vorhanden)
3. Navigiere zu https://openrouter.ai/keys
4. Erstelle einen neuen API-Key
5. Kopiere den Key (beginnt mit `sk-or-...`)
6. **Scope**: Der Key braucht Zugriff auf die gewünschten Modelle (z.B. Claude Opus 4.6)
7. **Budget**: Setze ein monatliches Limit (empfohlen als Sicherheitsnetz)

> ⚠️ Den Key NICHT in Chat/Repo einfügen! Direkt in `.env` auf dem NAS eintragen.

### B-02: Telegram Bot-Token bereithalten ✅

- Bot wurde bereits über @BotFather erstellt
- Token liegt bereit

### B-03: Tailscale 2FA prüfen (empfohlen) ⬜

1. Gehe zu https://login.tailscale.com/admin/settings/general
2. Prüfe, ob 2FA/MFA aktiviert ist
3. Falls nicht: Aktiviere es (schützt den gesamten Tailnet-Zugriff)

### B-04: Tailscale HTTPS-Zertifikate aktivieren ⬜

1. Gehe zu https://login.tailscale.com/admin/dns
2. Prüfe, ob "HTTPS Certificates" aktiviert ist
3. Falls nicht: Aktiviere es (nötig für Tailscale Serve)

---

## Während des Setups

### B-10: `.env`-Werte auf NAS eintragen ⬜

Nachdem Copilot die `.env`-Datei mit Platzhaltern erstellt hat:

```bash
ssh Benki-NAS3
nano /volume1/docker/openclaw/.env
```

Ersetze die `PLACEHOLDER`-Einträge:

| Variable                   | Wert                              | Woher                         |
|----------------------------|-----------------------------------|-------------------------------|
| `OPENCLAW_GATEWAY_TOKEN`    | (wird automatisch generiert)     | Copilot generiert das         |
| `OPENROUTER_API_KEY`        | `sk-or-...`                      | https://openrouter.ai/keys    |
| `TELEGRAM_BOT_TOKEN`        | `123456:ABCDEF...`              | @BotFather auf Telegram       |

### B-11: Onboarding-Wizard durchlaufen ⬜

Copilot startet den Wizard, aber du musst interaktiv antworten:

- **Model Provider**: OpenRouter / OpenAI-kompatibel
- **Base URL**: `https://openrouter.ai/api/v1`
- **API Key**: Deinen OpenRouter-Key eingeben
- **Default Model**: z.B. `anthropic/claude-opus-4-6`
- **Gateway Auth**: Token (wird aus .env gelesen)
- **Tailscale**: Überspringen (machen wir manuell)
- **Install Daemon**: Nein

### B-12: Telegram Pairing bestätigen ⬜

1. Sende eine Nachricht an deinen Bot auf Telegram
2. Du erhältst einen Pairing-Code (z.B. `4721`)
3. Teile den Code mit Copilot, oder bestätige selbst:
   ```bash
   ssh Benki-NAS3
   cd /volume1/docker/openclaw
   sudo docker compose run --rm openclaw-cli pairing approve telegram <CODE>
   ```

---

## Nach dem Setup

### B-20: Gateway-Token im Browser eingeben ⬜

1. Öffne `https://benki-nas3.tail73ca5d.ts.net/` im Browser
2. Gehe zu Settings → Token
3. Trage den Gateway-Token ein (aus `.env`)
4. Die Control UI sollte sich verbinden

### B-21: Uptime Kuma – OpenClaw Monitor anlegen ⬜

1. Öffne Uptime Kuma: `http://192.168.178.142:3001/`
2. Neuer Monitor:
   - **Type**: HTTP(s)
   - **Name**: OpenClaw Gateway
   - **URL**: `http://127.0.0.1:18789/health`
   - **Interval**: 60s
   - **Retries**: 3
3. Speichern

### B-22: Test-Konversation mit dem Bot ⬜

1. Sende eine Nachricht über Telegram an den Bot
2. Prüfe, ob eine KI-Antwort kommt
3. Teste auch die Browser-UI unter `https://benki-nas3.tail73ca5d.ts.net/`

---

## Spätere Phase (Home Assistant)

### B-30: Home Assistant Long-Lived Access Token erstellen ⬜

1. Öffne Home Assistant: `http://192.168.178.142:8123/`
2. Gehe zu: Profil → Security → Long-Lived Access Tokens
3. Erstelle einen neuen Token
4. Trage ihn in `.env` ein:
   ```
   HOMEASSISTANT_TOKEN=dein-token-hier
   HOMEASSISTANT_URL=http://host.docker.internal:8123
   ```
5. Teile Copilot mit, dass der Token bereit ist → Copilot konfiguriert die Integration

---

## Notfall / Wartung

### B-90: Secret-Rotation (bei Verdacht auf Kompromittierung) ⬜

1. Neuen Gateway-Token generieren: `openssl rand -hex 32`
2. In `.env` eintragen
3. Gateway neu starten: `sudo docker compose restart openclaw-gateway`
4. Token im Browser neu eingeben

### B-91: OpenRouter-Key rotieren ⬜

1. Alten Key auf https://openrouter.ai/keys löschen
2. Neuen Key erstellen
3. In `.env` eintragen
4. Gateway neu starten
