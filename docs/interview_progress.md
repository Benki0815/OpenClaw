# OpenClaw – Interview-Fortschritt

> Diese Datei trackt den Fortschritt des Setup-Interviews.  
> Bei Unterbrechung: Copilot liest diese Datei und setzt beim letzten offenen Punkt fort.

## Status: ABGESCHLOSSEN ✅

## Themenblöcke & Fortschritt

### A) Grundverständnis – Was ist OpenClaw?
- [x] A1: Was ist OpenClaw? (Zweck, Funktionsumfang, Tech-Stack) ✅
- [x] A2: Eigenes Projekt oder existierendes Open-Source-Projekt? ✅

### B) NAS & Zugriff
- [x] B1: NAS-Modell, DSM-Version ✅ (per SSH ermittelt)
- [x] B2: Docker & Compose installiert? Versionen? ✅ (per SSH ermittelt)
- [x] B3: Bereits laufende Container (Portbelegung)? ✅ (per SSH ermittelt)
- [x] B4: Verfügbare Ordnerpfade / Volumes ✅ (per SSH ermittelt)

### C) Netzwerk & Erreichbarkeit
- [x] C1: LAN-IP, gewünschte Domain/Subdomain ✅
- [x] C2: Reverse Proxy vorhanden? → Nicht nötig, Tailscale Serve ✅
- [x] C3: TLS/Let's Encrypt → Tailscale Serve macht HTTPS ✅
- [x] C4: Tailscale – genauer Einsatzzweck & aktuelle Konfiguration ✅
- [x] C5: Firewall-Regeln → Gateway auf Loopback, nur Tailnet-Zugriff ✅

### D) OpenClaw Laufzeit
- [x] D1: Persistente Daten → Standard OpenClaw-Pfade ✅
- [x] D2: Log-Retention, Restart-Policy → restart: unless-stopped ✅
- [x] D3: Update-Strategie → Watchtower (automatisch) ✅

### E) Integrationen
- [x] E1: LLM-Provider → OpenRouter (verschiedene Modelle, wechselbar) ✅
- [x] E2: Channels → Telegram Bot (Start), ggf. eigene Entwicklung später ✅
- [x] E3: Home Assistant → Alles (Steuern+Sensoren+Automatisierung, später) ✅

### F) Sicherheitsanforderungen
- [x] F1: Authentifizierung → Tailscale Identity (automatisch) ✅
- [x] F2: MFA → Nicht nötig (Tailscale = Device-Auth) ✅
- [x] F3: IP-Allowlist → Nur Tailnet, kein öffentlicher Zugriff ✅
- [x] F4: Secret-Rotation → Manuell bei Bedarf ✅
- [x] F5: Backup-Strategie → Hyper Backup (tägliche Sicherung) ✅

### G) Betrieb & Monitoring
- [x] G1: Monitoring → Uptime Kuma (vorhanden, einbinden) ✅
- [x] G2: Alerting → über Uptime Kuma / Telegram ✅
- [x] G3: Rollback-Strategie → Watchtower + Hyper Backup ✅

---

## Bisherige Antworten

### A1/A2: Was ist OpenClaw?
- Bestehendes Open-Source-Projekt: https://github.com/openclaw/openclaw
- Persönlicher KI-Assistent, TypeScript/Node.js ≥22
- Gateway (WebSocket Control Plane), Default-Port 18789
- Offizielles Docker-Image + docker-compose.yml + docker-setup.sh vorhanden
- Unterstützt WhatsApp, Telegram, Slack, Discord, Signal u.v.m.
- Eingebaute Tailscale-Integration (Serve/Funnel)
- Läuft als non-root `node` User (UID 1000)
- Umfangreiche Security-Features (Sandboxing, DM-Pairing, Tool-Policies)
- Offizielle Doku: https://docs.openclaw.ai
- Docker-Guide: https://docs.openclaw.ai/install/docker
- Security-Guide: https://docs.openclaw.ai/gateway/security
- Tailscale-Guide: https://docs.openclaw.ai/gateway/tailscale

### B1: NAS-System
- **Modell**: Synology DS220+ (Gemini Lake)
- **OS**: DSM 7.3.2 (Build 86009, 2026-01-29)
- **Kernel**: 4.4.302+ x86_64
- **RAM**: 18 GB total, ~12 GB verfügbar
- **Speicher**: 3.5 TB Volume, 1.1 TB frei (71% belegt)

### B2: Docker & Compose
- **Docker**: 24.0.2 (via ContainerManager-Paket)
- **Compose**: v2.20.1
- **Pfad**: `/var/packages/ContainerManager/target/usr/bin/docker`
- **Hinweis**: `docker` nicht im PATH, muss mit vollem Pfad oder via sudo aufgerufen werden

### B3: Tailscale
- **Version**: 1.58.2
- **NAS Tailscale-IP**: 100.66.139.14 / fd7a:115c:a1e0::c001:8b0e
- **MagicDNS**: benki-nas3.tail73ca5d.ts.net
- **Peers**: Windows-Notebook, Samsung Android, VPS01

### B4: Laufende Container & Portbelegung
| Container           | Image                          | Ports              |
|---------------------|--------------------------------|--------------------|
| homeassistant       | homeassistant/home-assistant   | host-Netzwerk      |
| ballistixg-frontend | ballistixg-frontend            | 3030→3000          |
| ballistixg-backend  | ballistixg-backend             | 8008→8000          |
| ballistixg-db       | postgres:15-alpine             | 5433→5432          |
| postgres            | postgres:latest                | 55432→5432         |
| grafana             | grafana/grafana                | 3000→3000          |
| grafana-copy        | grafana/grafana                | 32768→3000         |
| uptime-kuma         | louislam/uptime-kuma:1         | 3001→3001          |
| flask-app           | python:3.11                    | 5050→5000          |
| milvus-attu         | zilliz/attu                    | 8000→3000          |
| milvus-minio        | minio/minio                    | 9000-9001→9000-9001|
| pgadmin             | dpage/pgadmin4                 | 5580→80, 5443→443  |
| Flowise             | flowiseai/flowise              | 3333→3000          |
| nodered             | nodered/node-red               | 1880→1880          |
| mosquitto           | eclipse-mosquitto              | 1883→1883          |
| influxdb            | influxdb                       | 8086→8086          |
| python              | python:latest                  | keine              |

**Belegte Host-Ports**: 1880, 1883, 3000, 3001, 3030, 3333, 5050, 5433, 5443, 5580, 8000, 8008, 8086, 9000, 9001, 32768, 55432
**Port 18789 (OpenClaw Default)**: FREI ✅

### B5: Docker-Netzwerke
- bridge (default), host, none
- ballistixg_ballistixg-net, milvus_milvus-net, uptime-kuma_default

### B6: Existierende OpenClaw-Verzeichnisse
- `/volume1/docker/openclaw/` existiert (leer, nur Unterordner `config/`)
- Owner: stephan:users

### C1-C5: Netzwerk & Erreichbarkeit
- **Zugriff**: Nur Tailnet (Handy, Notebook, zuhause+unterwegs)
- **Einzelbenutzer**: Nur User Benki/Stephan
- **Lösung**: Tailscale Serve (Gateway auf Loopback + HTTPS über Tailscale)
- **URL**: `https://benki-nas3.tail73ca5d.ts.net/`
- **Kein Reverse Proxy nötig**: Tailscale Serve übernimmt TLS + Routing
- **Kein öffentlicher Zugriff**: Kein Funnel, kein Port-Forwarding
- **Home Assistant**: Integration vorbereiten (spätere Phase), läuft bereits auf gleicher NAS im host-Netzwerk
- **Gateway Config**: `bind: "loopback"`, `tailscale.mode: "serve"`, `auth.allowTailscale: true`

### D: Laufzeit
- **Restart-Policy**: `unless-stopped`
- **Updates**: Automatisch via Watchtower
- **Persistente Daten**: Standard OpenClaw-Pfade unter `/volume1/docker/openclaw/`

### E: Integrationen
- **LLM-Provider**: OpenRouter (verschiedene Modelle, wechselbar)
  - Benötigt: `OPENROUTER_API_KEY` in `.env`
- **Messaging**: Telegram Bot (Phase 1)
  - Benötigt: `TELEGRAM_BOT_TOKEN` in `.env`
- **Home Assistant**: Vorbereiten (spätere Phase)
  - HA läuft bereits auf gleicher NAS im host-Netzwerk
  - Scope: Vollzugriff (Steuern, Sensoren, Automatisierungen)
  - Benötigt später: Long-Lived Access Token von HA
- **Eigene Entwicklung**: Ggf. später eigene Channels/Tools

### F: Sicherheit
- **Browser-Auth**: Tailscale Identity (automatisch, kein extra Passwort)
- **Einzelbenutzer**: Nur Benki/Stephan über Tailnet
- **Kein MFA nötig**: Tailscale = Device-basierte Authentifizierung
- **Netzwerk**: Nur Tailnet-Zugriff, kein öffentliches Expose

### G: Betrieb & Monitoring
- **Monitoring**: Uptime Kuma (läuft bereits auf Port 3001), OpenClaw dort einbinden
- **Backups**: Hyper Backup sichert täglich alles unter `/volume1/docker/`
  → Alle OpenClaw-Daten müssen unter `/volume1/docker/openclaw/` gemountet werden!
- **Updates**: Watchtower (automatisch)
- **Telegram-Bot**: Bereits erstellt, Token vorhanden (kommt in .env)

