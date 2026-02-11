# OpenClaw – Setup-Anleitung

> Exakte Schritte und Befehle für NAS-Setup, Docker, OpenClaw, Tailscale Serve und Verifikation.

---

## Voraussetzungen

| Komponente         | Status   | Details                                         |
|--------------------|----------|-------------------------------------------------|
| Synology DSM 7.3.2 | ✅       | Benki-NAS3                                      |
| ContainerManager   | ✅       | Docker 24.0.2 + Compose v2.20.1                |
| Tailscale          | ✅       | v1.58.2, IP: 100.66.139.14                     |
| SSH-Zugang         | ✅       | User `*new*`, sudo-fähig                        |
| Port 18789         | ✅       | Frei                                            |
| OpenRouter API-Key | ❓       | Muss Benki bereitstellen                        |
| Telegram Bot-Token | ✅       | Vorhanden                                       |

### Docker-Befehl auf Synology

Docker ist auf Synology nicht im PATH. Voller Pfad:
```bash
DOCKER="sudo /var/packages/ContainerManager/target/usr/bin/docker"
```

Alle Befehle in dieser Anleitung verwenden diesen Pfad mit `sudo`.

---

## Phase 1: NAS-Verzeichnisse vorbereiten

```bash
# SSH auf NAS
ssh Benki-NAS3

# Verzeichnisstruktur erstellen
sudo mkdir -p /volume1/docker/openclaw/{config,workspace}

# Berechtigungen: UID 1000 (node-User im Container)
sudo chown -R 1000:1000 /volume1/docker/openclaw/config
sudo chown -R 1000:1000 /volume1/docker/openclaw/workspace

# Verzeichnis-Rechte: nur Owner
sudo chmod 700 /volume1/docker/openclaw/config
sudo chmod 755 /volume1/docker/openclaw/workspace

# Prüfen
ls -la /volume1/docker/openclaw/
```

---

## Phase 2: Dateien deployen

### 2.1 docker-compose.yml auf NAS kopieren

```bash
# Vom lokalen Rechner (VS Code Terminal):
scp docker-compose.yml Benki-NAS3:/volume1/docker/openclaw/docker-compose.yml
```

### 2.2 .env erstellen (auf dem NAS)

```bash
ssh Benki-NAS3

# .env erstellen (Secrets direkt auf dem NAS, NIE im Repo!)
cat > /volume1/docker/openclaw/.env << 'ENVEOF'
# OpenClaw Gateway
OPENCLAW_GATEWAY_TOKEN=HIER_TOKEN_EINTRAGEN
OPENCLAW_GATEWAY_PORT=18789
OPENCLAW_GATEWAY_BIND=lan

# OpenRouter (LLM-Provider)
OPENROUTER_API_KEY=HIER_KEY_EINTRAGEN

# Telegram Bot
TELEGRAM_BOT_TOKEN=HIER_TOKEN_EINTRAGEN

# Pfade
OPENCLAW_CONFIG_DIR=/volume1/docker/openclaw/config
OPENCLAW_WORKSPACE_DIR=/volume1/docker/openclaw/workspace
ENVEOF

# Berechtigungen einschränken
sudo chmod 600 /volume1/docker/openclaw/.env
```

> **⚠️ WICHTIG**: Die Platzhalter `HIER_..._EINTRAGEN` müssen durch echte Werte ersetzt werden!
> Siehe [todo_benki.md](todo_benki.md) für die vollständige Liste.

### 2.3 Gateway-Token generieren

```bash
# Zufälligen Token generieren
TOKEN=$(openssl rand -hex 32)
echo "Generierter Token: $TOKEN"

# In .env eintragen
sed -i "s/HIER_TOKEN_EINTRAGEN/$TOKEN/" /volume1/docker/openclaw/.env
```

> Hinweis: Wenn `openssl` nicht verfügbar: `head -c 32 /dev/urandom | xxd -p`

---

## Phase 3: Docker Image pullen & Onboarding

```bash
ssh Benki-NAS3
cd /volume1/docker/openclaw

# Image ziehen
sudo docker compose pull

# Onboarding-Wizard starten (interaktiv!)
# Der Wizard fragt nach: Model-Provider, API-Keys, Channel-Setup
sudo docker compose run --rm openclaw-cli onboard --no-install-daemon
```

> **Während des Onboardings:**
> - Model: OpenRouter wählen / OpenAI-kompatibel mit Base-URL `https://openrouter.ai/api/v1`
> - API-Key: OpenRouter-Key eingeben
> - Gateway Auth: Token (wird aus .env gelesen)
> - Tailscale: Skip (wir konfigurieren Serve manuell auf dem Host)
> - Install Daemon: Nein (Docker übernimmt das)

---

## Phase 4: Gateway starten

```bash
cd /volume1/docker/openclaw

# Gateway + Watchtower starten
sudo docker compose up -d

# Status prüfen
sudo docker compose ps

# Logs (live)
sudo docker compose logs -f openclaw-gateway
```

### Health Check

```bash
# Lokaler Health Check (auf dem NAS)
curl -s http://127.0.0.1:18789/health

# Gateway-Status
sudo docker compose exec openclaw-gateway node dist/index.js health --token "$(grep OPENCLAW_GATEWAY_TOKEN .env | cut -d= -f2)"
```

---

## Phase 5: Tailscale Serve konfigurieren

Tailscale Serve macht den Gateway per HTTPS über das Tailnet erreichbar, ohne Ports zu öffnen.

```bash
ssh Benki-NAS3

# Tailscale Serve aktivieren: HTTPS → localhost:18789
sudo /var/packages/Tailscale/target/bin/tailscale serve --bg 18789

# Status prüfen
sudo /var/packages/Tailscale/target/bin/tailscale serve status
```

### Verifikation (von einem Tailnet-Gerät)

```bash
# Vom Notebook (mit Tailscale aktiv):
curl -s https://benki-nas3.tail73ca5d.ts.net/health

# Oder im Browser öffnen:
# https://benki-nas3.tail73ca5d.ts.net/
```

### Tailscale Serve entfernen (falls nötig)

```bash
sudo /var/packages/Tailscale/target/bin/tailscale serve --remove 18789
```

---

## Phase 6: Telegram Bot einrichten

```bash
ssh Benki-NAS3
cd /volume1/docker/openclaw

# Telegram-Channel hinzufügen
sudo docker compose run --rm openclaw-cli channels add --channel telegram --token "DEIN_BOT_TOKEN"

# Gateway neu starten, damit der Channel aktiv wird
sudo docker compose restart openclaw-gateway

# Prüfen ob Telegram verbunden ist
sudo docker compose logs openclaw-gateway | grep -i telegram
```

### Telegram DM-Pairing

1. Schreibe dem Bot eine Nachricht auf Telegram
2. Du erhältst einen Pairing-Code
3. Bestätige den Code:
   ```bash
   sudo docker compose run --rm openclaw-cli pairing list telegram
   sudo docker compose run --rm openclaw-cli pairing approve telegram <code>
   ```

---

## Phase 7: Uptime Kuma – Monitoring einrichten

Uptime Kuma läuft bereits auf `http://192.168.178.142:3001/`.

1. Neuen Monitor anlegen:
   - **Type**: HTTP(s)
   - **Name**: OpenClaw Gateway
   - **URL**: `http://127.0.0.1:18789/health`
   - **Interval**: 60 Sekunden
   - **Retry**: 3

2. Optional: Externen Monitor (via Tailscale):
   - **URL**: `https://benki-nas3.tail73ca5d.ts.net/health`
   - (Nur wenn Uptime Kuma auch per Tailscale erreichbar ist)

---

## Phase 8: Verifikation – Runbook-Checkliste

| #  | Check                                      | Befehl / Aktion                                           | Erwartung       |
|----|--------------------------------------------|-----------------------------------------------------------|-----------------|
| 1  | Container läuft                            | `sudo docker compose ps`                                  | `Up`            |
| 2  | Health Check lokal                         | `curl http://127.0.0.1:18789/health`                      | `200 OK`        |
| 3  | Health Check Tailscale                     | `curl https://benki-nas3.tail73ca5d.ts.net/health`        | `200 OK`        |
| 4  | Control UI erreichbar                      | Browser: `https://benki-nas3.tail73ca5d.ts.net/`          | UI lädt         |
| 5  | Gateway-Token funktioniert                 | Token in Control UI eingeben                               | Verbunden       |
| 6  | Telegram Bot reagiert                      | Nachricht an Bot senden                                    | Pairing-Code    |
| 7  | Telegram Pairing bestätigt                 | `openclaw-cli pairing approve telegram <code>`            | Approved        |
| 8  | Telegram Chat funktioniert                 | Nachricht an Bot → KI-Antwort                              | Antwort kommt   |
| 9  | Watchtower läuft                           | `sudo docker compose logs openclaw-watchtower`            | Polling-Log     |
| 10 | Uptime Kuma Monitor grün                   | Uptime Kuma Dashboard                                      | ✅ Up           |
| 11 | Logs sauber                                | `sudo docker compose logs openclaw-gateway --tail 50`     | Keine Errors    |

---

## Troubleshooting

### Container startet nicht

```bash
# Logs prüfen
sudo docker compose logs openclaw-gateway

# Container-Details
sudo docker compose ps -a

# Neustart erzwingen
sudo docker compose down && sudo docker compose up -d
```

### Tailscale Serve funktioniert nicht

```bash
# Tailscale-Status
sudo /var/packages/Tailscale/target/bin/tailscale status

# Serve-Status
sudo /var/packages/Tailscale/target/bin/tailscale serve status

# HTTPS auf Tailnet aktiviert?
# → DSM → Tailscale Package Settings prüfen
# → https://login.tailscale.com/admin/dns → HTTPS Certificates
```

### Permission-Fehler

```bash
# Rechte prüfen
ls -la /volume1/docker/openclaw/
ls -la /volume1/docker/openclaw/config/

# Rechte reparieren
sudo chown -R 1000:1000 /volume1/docker/openclaw/config
sudo chown -R 1000:1000 /volume1/docker/openclaw/workspace
```

### Gateway-Token vergessen

```bash
# Aus .env auslesen
grep OPENCLAW_GATEWAY_TOKEN /volume1/docker/openclaw/.env

# Dashboard-URL mit Token anzeigen
sudo docker compose run --rm openclaw-cli dashboard --no-open
```
