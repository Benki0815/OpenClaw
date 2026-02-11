# Copilot Project Instructions (must follow)

## Language & style
- Always communicate with the user in German.
- Code: comments in English.
- Identifiers: variables, functions, classes, files in English.

## Docs
- Documentation/plans are written in German as Markdown under /docs.
- If you create/modify a plan, update the related /docs/*.md.
- If you get new behavior instructions, add them to this file. Always keep this file up to date with the latest instructions and rules.

## Config
- GitHub Repository: https://github.com/Benki0815/OpenClaw.git
- Upstream-Projekt: https://github.com/openclaw/openclaw
- Offizielle Doku: https://docs.openclaw.ai
- Docker-Image: ghcr.io/openclaw/openclaw:latest
- Docker-Container: openclaw-gateway, openclaw-watchtower
- NAS: Benki-NAS3 HostName: benki-nas3.tail73ca5d.ts.net bzw. lokal: 192.168.178.142; User: *new*, Port: 22, IdentityFile: C:\Users\s.benik\.ssh\id_rsa
- OC-Config-Verzeichnis: /volume1/docker/openclaw/config
- OC-Workspace-Verzeichnis: /volume1/docker/openclaw/workspace
- OC-Compose-Pfad: /volume1/docker/openclaw/docker-compose.yml
- OC-Env-Pfad: /volume1/docker/openclaw/.env
- OC-Gateway-URL (Tailscale): https://benki-nas3.tail73ca5d.ts.net/
- OC-Gateway-Port: 18789 (nur localhost, Tailscale Serve davor)
- LLM-Provider: OpenRouter (verschiedene Modelle, wechselbar)
- Messaging: Telegram Bot (Phase 1)
- Home Assistant: Vorbereitet (spätere Phase)
- Monitoring: Uptime Kuma (Port 3001)
- Updates: Watchtower (automatisch, Label-basiert)

## NAS/SSH
- Host-Shortcut: Benki-NAS3
- HostName: benki-nas3.tail73ca5d.ts.net
- User: *new*
- Port: 22
- IdentityFile: C:\Users\s.benik\.ssh\id_rsa
- NAS-Modell: Synology DS220+ / DSM 7.3.2
- Docker-Pfad: /var/packages/ContainerManager/target/usr/bin/docker (NICHT im PATH!)
- Tailscale-Pfad: /var/packages/Tailscale/target/bin/tailscale
- Docker-Befehle immer mit vollem Pfad und sudo ausfuehren:
  sudo /var/packages/ContainerManager/target/usr/bin/docker compose ...
- Details/Referenz: ssh_dev_setup_guide.md im Workspace
- NAS-Datenpfade stehen im Workspace unter /docs oder in der ssh_dev_setup_guide.md.
- Wenn unklar: im Workspace nach Pfaden/Volumes/Container-Infos suchen und dann per SSH verifizieren.
- Der User *new* ist in sudoers.
- Fuer administrative Schritte sudo verwenden.
- Keine destruktiven Aktionen ohne explizite Freigabe.
- Alle SSH-/NAS-Aktionen im VS Code Terminal ausfuehren.
- Selbststaendig per SSH arbeiten (kein lokaler Docker).
- Container nur direkt auf dem NAS per SSH verwalten (kein Docker lokal).
- Terminal-Kommandos sauber begruenden, kurze Beschreibungen geben.
- Bei Unsicherheit zuerst pruefen (z.B. ls, docker ps, docker compose ps).
- Aenderungen kurz dokumentieren.

## OpenClaw-spezifische Regeln
- Secrets (API-Keys, Tokens) NIEMALS im Repo speichern. Nur in .env auf dem NAS.
- .env.example als Template im Repo pflegen, aber ohne echte Werte.
- Vor Aenderungen an der OpenClaw-Config: Backup des aktuellen Stands.
- Gateway nur neu starten, wenn noetig (laeuft als Service).
- Tailscale Serve Status immer pruefen nach NAS-/Tailscale-Neustart.
- OpenClaw-Doku unter /docs/ pflegen. Aenderungen in den richtigen Dateien dokumentieren:
  - Architektur/Konzept: docs/openClaw_project.md
  - Setup-Befehle: docs/setup.md
  - Copilot-Aufgaben: docs/todo_copilot.md
  - Benki-Aufgaben: docs/todo_benki.md
  - Interview-Status: docs/interview_progress.md
- Bei Sicherheitsfragen zuerst die offizielle Security-Doku konsultieren:
  https://docs.openclaw.ai/gateway/security


## Workflow automation rules. MUST FOLLOW
- Nach persistenter Aenderung: klaren Github Commit erstellen und online pushen.
- Container nur per SSH direkt auf dem NAS neu starten (lokal kein Docker verfuegbar).
- Wenn ein Docker-Conteiner neu zu starten/builden ist: Fuehre das selbststaendig per SSH auf dem NAS aus.
- Wenn eine Funktion oder andere Dinge unklar sind, schaue online in der offiziellen Doku nach: https://docs.openclaw.ai
- Bei Unsicherheit oder Unklarheiten: Immer zuerst in der offiziellen Doku nachschauen, bevor du Annahmen triffst oder Aktionen durchfuehrst.
- Wenn eine Funktion oder andere Dinge unklar sind, schaue online in der offiziellen Doku nach: https://docs.openclaw.ai
- Bei Unsicherheit oder Unklarheiten: Immer zuerst in der offiziellen Doku nachschauen, bevor du Annahmen triffst oder Aktionen durchfuehrst.


## Grundgesetz (Harte Regeln)
- §1: Niemals eine Datenbank loeschen. Niemals Daten loeschen, ausser explizit vom User angeordnet oder bei eindeutig fehlerhaften/redundanten Daten und nur nach ausdruecklicher Genehmigung.
- §2: Vor dem Loeschen von Dateien, Funktionen oder API-Endpunkten den User explizit hinweisen und eine Erlaubnis einholen; niemals eigenmaechtig loeschen.
- §3: Hilfsprogramme/Scriptdateien nicht direkt loeschen. Zuerst in einen passenden Unterordner verschieben (z.B. /deprecated, /trash o.a.).
- §4: Diese Hinweise im Chat stets deutlich kennzeichnen, damit sie nicht untergehen.