# OpenClaw – Projektdokumentation

> Konzept, Architektur, Sicherheitsmodell, Abhängigkeiten, Netzwerk, Datenpfade, Backup/Restore

---

## 1. Projektübersicht

**OpenClaw** ist ein persönlicher KI-Assistent (Open Source), der als Docker-Container auf einem Synology NAS betrieben wird. Er kommuniziert über Telegram, ist per Browser erreichbar und kann Home Assistant steuern.

| Eigenschaft       | Wert                                            |
|--------------------|------------------------------------------------|
| Upstream-Repo      | https://github.com/openclaw/openclaw           |
| Offizielle Doku    | https://docs.openclaw.ai                       |
| Tech-Stack         | TypeScript / Node.js ≥22                       |
| Container-Image    | `ghcr.io/openclaw/openclaw:latest`             |
| Gateway-Port       | 18789                                           |
| LLM-Provider       | OpenRouter (verschiedene Modelle, wechselbar)  |
| Messaging-Channel  | Telegram Bot (Phase 1)                         |
| Zugriffsmethode    | Tailscale Serve (HTTPS, Tailnet-only)          |

---

## 2. Architektur

```
  Handy / Notebook (Tailscale-Client)
         │
         │  WireGuard-Tunnel (verschlüsselt)
         ▼
  ┌─────────────────────────────────────────────┐
  │  Synology NAS (Benki-NAS3)                  │
  │  Tailscale v1.58.2 (100.66.139.14)         │
  │                                              │
  │  tailscale serve → 127.0.0.1:18789         │
  │       │                                      │
  │       ▼                                      │
  │  ┌────────────────────────────────┐          │
  │  │  Docker: openclaw_net (bridge) │          │
  │  │                                │          │
  │  │  ┌──────────────────────────┐  │          │
  │  │  │  openclaw-gateway        │  │          │
  │  │  │  Port 18789 (intern)     │  │          │
  │  │  │  non-root: node (1000)   │  │          │
  │  │  │  Volumes:                │  │          │
  │  │  │   /volume1/docker/       │  │          │
  │  │  │    openclaw/config       │  │          │
  │  │  │    openclaw/workspace    │  │          │
  │  │  └──────────────────────────┘  │          │
  │  │                                │          │
  │  │  ┌──────────────────────────┐  │          │
  │  │  │  openclaw-watchtower     │  │          │
  │  │  │  (Auto-Updates, täglich) │  │          │
  │  │  └──────────────────────────┘  │          │
  │  └────────────────────────────────┘          │
  │                                              │
  │  homeassistant (host-Netzwerk, Port 8123)   │
  │       ▲                                      │
  │       │ host.docker.internal (spätere Phase) │
  │       └──────── openclaw-gateway ────────────┘
  └─────────────────────────────────────────────┘
```

### Zugriffspfade

| Von              | Über                                    | URL                                         |
|------------------|-----------------------------------------|---------------------------------------------|
| Handy (unterwegs)| Tailscale VPN → Serve                   | `https://benki-nas3.tail73ca5d.ts.net/`     |
| Notebook (Büro)  | Tailscale VPN → Serve                   | `https://benki-nas3.tail73ca5d.ts.net/`     |
| Notebook (LAN)   | Tailscale (lokal) → Serve               | `https://benki-nas3.tail73ca5d.ts.net/`     |
| Telegram         | Telegram-Server → Bot-API → Gateway     | Polling (kein Webhook nötig)                |

---

## 3. Deployment-Varianten

### Variante 1: Maximale Sicherheit – nur Tailnet (GEWÄHLT ✅)

- Gateway bindet auf **127.0.0.1** (Loopback)
- **Tailscale Serve** stellt HTTPS bereit (nur Tailnet-Geräte)
- Kein Port-Forwarding, kein öffentliches Expose
- Authentifizierung: Tailscale Identity Headers + Gateway Token
- Telegram-Bot: Polling-Modus (kein eingehender Webhook nötig)

**Vorteile:**
- Kein einziger Port nach außen offen
- WireGuard-Verschlüsselung (Tailscale)
- Tailscale-MagicDNS → automatisches HTTPS-Zertifikat
- Geringste Angriffsfläche

**Nachteile:**
- Zugriff nur mit Tailscale-Client möglich
- Kein Zugriff aus dem Internet ohne Tailscale

### Variante 2: Reverse Proxy + TLS (NICHT gewählt)

- Gateway hinter Synology Reverse Proxy oder Nginx Proxy Manager
- Let's Encrypt TLS-Zertifikat
- Öffentliche Domain + HTTPS
- Zusätzliche Authentifizierung (Authelia/OAuth)

**Warum nicht gewählt:** Unnötige Komplexität, größere Angriffsfläche, einzelner Benutzer = Tailnet reicht.

---

## 4. Sicherheitsmodell

### 4.1 Netzwerk-Segmentierung

| Schicht                | Maßnahme                                           |
|------------------------|----------------------------------------------------|
| Internet → NAS         | Kein Port offen, nur Tailscale-Tunnel              |
| Tailnet → Gateway      | Tailscale Serve (HTTPS), nur authentifizierte Peers |
| Docker Host → Container| `127.0.0.1:18789`, nicht LAN-gebunden              |
| Container-Netzwerk     | Eigenes `openclaw_net` (bridge), isoliert           |
| Container → Host       | Nur `host.docker.internal` für HA (spätere Phase)  |

### 4.2 Container-Hardening

| Maßnahme                  | Umsetzung                                    |
|---------------------------|----------------------------------------------|
| Non-root                  | User `node` (UID 1000)                       |
| no-new-privileges         | `security_opt: [no-new-privileges:true]`     |
| Kein privileged-Modus     | Nicht gesetzt (default: false)               |
| Kein Host-PID/IPC         | Nicht gesetzt (default: false)               |
| Kein Docker-Socket-Mount  | Nur Watchtower hat Socket-Zugriff            |
| tmpfs für /tmp            | Verhindert persistente Temp-Dateien          |
| mDNS deaktiviert          | `discovery.mdns.mode: "off"` in Config       |

### 4.3 Secrets-Management

| Secret                    | Speicherort         | Im Repo?  |
|---------------------------|---------------------|-----------|
| Gateway-Token             | `.env` auf NAS      | ❌ Nein   |
| OpenRouter API-Key        | `.env` auf NAS      | ❌ Nein   |
| Telegram Bot-Token        | `.env` auf NAS      | ❌ Nein   |
| HA Access-Token (später)  | `.env` auf NAS      | ❌ Nein   |
| Template                  | `.env.example`      | ✅ Ja     |
| OpenClaw Config           | `openclaw.json`     | ❌ Nein   |
| Config-Template           | `openclaw.json.example` | ✅ Ja |

### 4.4 DM-Sicherheit (Telegram)

- **DM-Policy**: `pairing` (Default) – unbekannte Absender müssen einen Pairing-Code bestätigen
- **Nur Single-User**: Nur der Besitzer darf mit dem Bot interagieren
- Pairing-Codes verfallen nach 1 Stunde

---

## 5. Bedrohungsmodell (Threat Model)

### 5.1 Angriffsvektoren

| #  | Vektor                           | Risiko | Gegenmaßnahme                                            |
|----|----------------------------------|--------|-----------------------------------------------------------|
| T1 | Fremder Zugriff auf Gateway-UI   | Mittel | Tailnet-only, kein öffentlicher Port                      |
| T2 | Prompt Injection via Telegram    | Hoch   | DM-Pairing, Tool-Einschränkungen, Anthropic Opus 4.6     |
| T3 | API-Key-Leak aus Repo            | Hoch   | `.env` gitignored, `.env.example` als Template            |
| T4 | Container-Ausbruch               | Gering | no-new-privileges, non-root, eigenes Netzwerk             |
| T5 | Watchtower Supply-Chain-Angriff  | Gering | Label-Filter (nur openclaw), GHCR als vertrauensw. Quelle|
| T6 | Session-Daten auf Disk           | Mittel | Verzeichnis-Rechte 700, Hyper Backup verschlüsselt       |
| T7 | HA-Missbrauch über OpenClaw      | Mittel | Spätere Phase: Tool-Einschränkungen, Read-only als Default|
| T8 | Tailscale-Account-Kompromittierung| Gering | Device-basierte Auth, 2FA auf Tailscale-Account empfohlen |

### 5.2 Maßnahmen-Matrix

| Maßnahme                  | Schützt gegen | Priorität |
|---------------------------|---------------|-----------|
| Tailscale Serve           | T1, T8        | Kritisch  |
| .env + .gitignore         | T3            | Kritisch  |
| DM-Pairing (Telegram)     | T2            | Hoch      |
| Container-Hardening       | T4            | Hoch      |
| mDNS deaktiviert          | T1            | Mittel    |
| Hyper Backup (täglich)    | T6            | Mittel    |
| Watchtower Label-Filter   | T5            | Mittel    |

---

## 6. Abhängigkeiten

| Komponente             | Version / Image                      | Quelle                |
|------------------------|--------------------------------------|-----------------------|
| Synology DSM           | 7.3.2                                | Synology              |
| Docker (ContainerMgr)  | 24.0.2                               | Synology-Paket        |
| Docker Compose          | v2.20.1                             | Synology-Paket        |
| Tailscale              | 1.58.2                               | Synology-Paket        |
| OpenClaw Gateway       | `ghcr.io/openclaw/openclaw:latest`   | GitHub Container Reg. |
| Watchtower             | `containrrr/watchtower:latest`       | Docker Hub            |
| Uptime Kuma            | `louislam/uptime-kuma:1` (vorhanden) | Existierender Container|

---

## 7. Netzwerk & Ports

| Port  | Dienst               | Bind            | Zugriff           |
|-------|----------------------|-----------------|-------------------|
| 18789 | OpenClaw Gateway     | 127.0.0.1       | Nur Tailscale Serve|
| 443   | Tailscale Serve      | Tailnet-only    | HTTPS via MagicDNS |
| 8123  | Home Assistant       | host-Netzwerk   | LAN + Tailscale   |
| 3001  | Uptime Kuma          | 0.0.0.0         | LAN               |

---

## 8. Datenpfade

```
/volume1/docker/openclaw/          ← Hyper Backup sichert dies täglich
├── docker-compose.yml             ← Container-Definition
├── .env                           ← Secrets (NUR auf NAS, nicht im Repo)
├── config/                        ← → /home/node/.openclaw (im Container)
│   ├── openclaw.json              ← Gateway-Konfiguration
│   ├── credentials/               ← Channel-Credentials (WhatsApp, etc.)
│   ├── agents/                    ← Agent-Konfiguration + Sessions
│   │   └── <agentId>/
│   │       ├── agent/
│   │       │   └── auth-profiles.json  ← API-Keys + OAuth-Tokens
│   │       └── sessions/
│   │           └── *.jsonl        ← Session-Transkripte
│   └── extensions/                ← Installierte Plugins
└── workspace/                     ← → /home/node/.openclaw/workspace
    ├── AGENTS.md                  ← Agent-Prompt
    ├── SOUL.md                    ← Persönlichkeit
    └── skills/                    ← Installierte Skills
```

---

## 9. Backup & Restore

### Backup (automatisch über Hyper Backup)

- **Was wird gesichert**: Alles unter `/volume1/docker/openclaw/`
- **Frequenz**: Täglich (Hyper Backup bestehende Aufgabe)
- **Enthält**: Config, Credentials, Sessions, Workspace, .env, docker-compose.yml

### Restore

1. Hyper Backup → Ordner `/volume1/docker/openclaw/` wiederherstellen
2. Container neu starten:
   ```bash
   cd /volume1/docker/openclaw
   sudo docker compose up -d
   ```
3. Verifikation:
   ```bash
   sudo docker compose logs -f openclaw-gateway
   curl -s http://127.0.0.1:18789/health
   ```

### Disaster Recovery

| Szenario                    | Aktion                                                |
|-----------------------------|-------------------------------------------------------|
| Container defekt            | `sudo docker compose down && sudo docker compose up -d` |
| Image defekt                | `sudo docker compose pull && sudo docker compose up -d`  |
| Config korrupt              | Hyper Backup → `/volume1/docker/openclaw/config/` restore |
| NAS defekt                  | Neue NAS + Hyper Backup vollständig restoren           |
| Tailscale-Zugriff verloren  | DSM-UI → Tailscale-Paket prüfen, `tailscale up`       |

---

## 10. Offizielle Referenzen

- Docker-Setup: https://docs.openclaw.ai/install/docker
- Security-Guide: https://docs.openclaw.ai/gateway/security
- Tailscale-Guide: https://docs.openclaw.ai/gateway/tailscale
- Konfiguration: https://docs.openclaw.ai/gateway/configuration
- Telegram-Channel: https://docs.openclaw.ai/channels/telegram
- Health Checks: https://docs.openclaw.ai/gateway/health
