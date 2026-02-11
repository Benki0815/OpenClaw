# OpenClaw – Aufgaben für Copilot (automatisch)

> Aufgaben, die Copilot automatisch in diesem Repo und auf dem NAS ausführt.
> Jede Aufgabe enthält die exakten Befehle.

---

## Status-Legende

- ⬜ Offen
- 🔄 In Arbeit
- ✅ Erledigt
- ⏸️ Wartet auf Benki (manuelle Aktion nötig)

---

## Phase 0: Repository vorbereiten

- ✅ **C-01**: Interview durchführen und dokumentieren
- ✅ **C-02**: Projektdokumentation erstellen (`docs/openClaw_project.md`)
- ✅ **C-03**: Setup-Anleitung erstellen (`docs/setup.md`)
- ✅ **C-04**: `.gitignore`, `.env.example`, `docker-compose.yml` erstellen
- ✅ **C-05**: `config/openclaw.json.example` erstellen
- ✅ **C-06**: Todo-Listen erstellen
- ⬜ **C-07**: `copilot-instructions.md` aktualisieren
- ⬜ **C-08**: Git-Commit + Push

---

## Phase 1: NAS-Verzeichnisse vorbereiten

- ⬜ **C-10**: Verzeichnisse erstellen und Rechte setzen

```bash
ssh Benki-NAS3 << 'EOF'
sudo mkdir -p /volume1/docker/openclaw/{config,workspace}
sudo chown -R 1000:1000 /volume1/docker/openclaw/config
sudo chown -R 1000:1000 /volume1/docker/openclaw/workspace
sudo chmod 700 /volume1/docker/openclaw/config
sudo chmod 755 /volume1/docker/openclaw/workspace
ls -la /volume1/docker/openclaw/
EOF
```

---

## Phase 2: Dateien auf NAS deployen

- ⬜ **C-20**: docker-compose.yml auf NAS kopieren

```bash
scp docker-compose.yml Benki-NAS3:/volume1/docker/openclaw/docker-compose.yml
```

- ⬜ **C-21**: .env-Grundgerüst auf NAS erstellen (Platzhalter)

```bash
ssh Benki-NAS3 << 'ENVEOF'
cat > /volume1/docker/openclaw/.env << 'EOF'
OPENCLAW_GATEWAY_TOKEN=PLACEHOLDER
OPENCLAW_GATEWAY_PORT=18789
OPENCLAW_GATEWAY_BIND=lan
OPENROUTER_API_KEY=PLACEHOLDER
TELEGRAM_BOT_TOKEN=PLACEHOLDER
OPENCLAW_CONFIG_DIR=/volume1/docker/openclaw/config
OPENCLAW_WORKSPACE_DIR=/volume1/docker/openclaw/workspace
EOF
sudo chmod 600 /volume1/docker/openclaw/.env
ENVEOF
```

- ⏸️ **C-22**: Warte auf Benki → `.env`-Werte eintragen (siehe `todo_benki.md`)

---

## Phase 3: Gateway-Token generieren

- ⬜ **C-30**: Token generieren und in .env eintragen

```bash
ssh Benki-NAS3 << 'EOF'
TOKEN=$(head -c 32 /dev/urandom | xxd -p | tr -d '\n')
echo "Gateway-Token: $TOKEN"
sed -i "s/OPENCLAW_GATEWAY_TOKEN=PLACEHOLDER/OPENCLAW_GATEWAY_TOKEN=$TOKEN/" /volume1/docker/openclaw/.env
grep OPENCLAW_GATEWAY_TOKEN /volume1/docker/openclaw/.env
EOF
```

---

## Phase 4: Docker Image ziehen

- ⬜ **C-40**: Image pullen

```bash
ssh Benki-NAS3 << 'EOF'
cd /volume1/docker/openclaw
sudo /var/packages/ContainerManager/target/usr/bin/docker compose pull
EOF
```

---

## Phase 5: Onboarding (interaktiv)

- ⏸️ **C-50**: Onboarding-Wizard – erfordert interaktive Eingabe von Benki

```bash
ssh -t Benki-NAS3 "cd /volume1/docker/openclaw && sudo /var/packages/ContainerManager/target/usr/bin/docker compose run --rm openclaw-cli onboard --no-install-daemon"
```

> **Hinweis**: `-t` für TTY (interaktiv). Benki muss Model/Provider-Fragen beantworten.

---

## Phase 6: Gateway starten

- ⬜ **C-60**: Container starten

```bash
ssh Benki-NAS3 << 'EOF'
cd /volume1/docker/openclaw
sudo /var/packages/ContainerManager/target/usr/bin/docker compose up -d
sudo /var/packages/ContainerManager/target/usr/bin/docker compose ps
EOF
```

- ⬜ **C-61**: Health Check lokal

```bash
ssh Benki-NAS3 "curl -sf http://127.0.0.1:18789/health && echo 'OK' || echo 'FAIL'"
```

---

## Phase 7: Tailscale Serve

- ⬜ **C-70**: Tailscale Serve aktivieren

```bash
ssh Benki-NAS3 << 'EOF'
sudo /var/packages/Tailscale/target/bin/tailscale serve --bg 18789
sudo /var/packages/Tailscale/target/bin/tailscale serve status
EOF
```

- ⬜ **C-71**: HTTPS-Zugriff über Tailnet testen

```bash
curl -sf https://benki-nas3.tail73ca5d.ts.net/health && echo 'Tailscale Serve OK' || echo 'FAIL'
```

---

## Phase 8: Telegram Bot einrichten

- ⬜ **C-80**: Telegram-Channel hinzufügen

```bash
ssh Benki-NAS3 << 'EOF'
cd /volume1/docker/openclaw
TELEGRAM_TOKEN=$(grep TELEGRAM_BOT_TOKEN .env | cut -d= -f2)
sudo /var/packages/ContainerManager/target/usr/bin/docker compose run --rm openclaw-cli channels add --channel telegram --token "$TELEGRAM_TOKEN"
sudo /var/packages/ContainerManager/target/usr/bin/docker compose restart openclaw-gateway
EOF
```

- ⏸️ **C-81**: Benki sendet Test-Nachricht → Pairing-Code

- ⬜ **C-82**: Pairing bestätigen

```bash
ssh Benki-NAS3 << 'EOF'
cd /volume1/docker/openclaw
sudo /var/packages/ContainerManager/target/usr/bin/docker compose run --rm openclaw-cli pairing list telegram
# Dann: sudo docker compose run --rm openclaw-cli pairing approve telegram <CODE>
EOF
```

---

## Phase 9: Monitoring

- ⏸️ **C-90**: Uptime Kuma – Benki richtet Monitor manuell ein (siehe `todo_benki.md`)

---

## Phase 10: Verifikation

- ⬜ **C-100**: Vollständiger Systemtest

```bash
ssh Benki-NAS3 << 'EOF'
echo "=== Container-Status ==="
cd /volume1/docker/openclaw
sudo /var/packages/ContainerManager/target/usr/bin/docker compose ps

echo "=== Health Check (lokal) ==="
curl -sf http://127.0.0.1:18789/health && echo "OK" || echo "FAIL"

echo "=== Logs (letzte 20 Zeilen) ==="
sudo /var/packages/ContainerManager/target/usr/bin/docker compose logs --tail 20 openclaw-gateway

echo "=== Tailscale Serve ==="
sudo /var/packages/Tailscale/target/bin/tailscale serve status

echo "=== .env vorhanden ==="
test -f /volume1/docker/openclaw/.env && echo "OK" || echo "FEHLT"

echo "=== Config-Rechte ==="
ls -la /volume1/docker/openclaw/config/
EOF
```
