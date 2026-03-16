# Sandbox-Handover für Schlaubi (OC Agent)

> **Datum:** 16.03.2026  
> **Änderung:** Docker-Sandbox für autonome Tool-/Skript-Ausführung aktiviert

---

## Was wurde eingerichtet?

Du (Schlaubi) hast jetzt eine **Docker-Sandbox** zur Verfügung. Das bedeutet: Wenn du Python-Skripte, Shell-Befehle oder andere Tools ausführen musst, passiert das **in einem isolierten Container** – nicht im Gateway.

### Verfügbare Tools in der Sandbox

| Tool/Paket | Version | Verfügbar |
|---|---|---|
| **Python 3.11** | System | ✅ |
| **pandas** | 3.0.1 | ✅ |
| **matplotlib** | 3.10.8 | ✅ |
| **numpy** | 2.4.3 | ✅ |
| **openpyxl** | 3.1.5 | ✅ |
| **requests** | 2.32.5 | ✅ |
| **tabulate** | 0.10.0 | ✅ |
| **python-dateutil** | 2.9.0 | ✅ |
| **git** | 2.39.x | ✅ |
| **curl, jq** | System | ✅ |

### Sandbox-Modus

- **Modus:** `non-main` – Dein Hauptchat (Telegram DM) läuft normal. Sub-Tasks und Skriptausführungen werden automatisch in die Sandbox geleitet.
- **Scope:** `agent` – Jeder Agent bekommt seinen eigenen isolierten Container.
- **Workspace:** `none` – Kein direkter Workspace-Zugriff (DinD-Kompatibilität). Dateien werden über Gateway-Tools gelesen/geschrieben.

---

## Sicherheitsregeln (WICHTIG!)

### Was du DARFST:
- ✅ Python-Skripte schreiben und ausführen
- ✅ Daten analysieren (CSV, Excel, JSON)
- ✅ Plots und Visualisierungen erstellen
- ✅ Shell-Befehle im Workspace ausführen (grep, awk, sort etc.)
- ✅ Dateien im Workspace über Gateway-Tools (fs_read, fs_write) lesen und schreiben

### Was du NICHT KANNST (by design):
- ❌ **Kein Netzwerk** – Der Sandbox-Container hat `network: none`. Keine HTTP-Requests, keine API-Calls, kein Internet aus der Sandbox heraus.
- ❌ **Kein direkter Workspace-Mount** – Wegen Docker-in-Docker (DinD) hat die Sandbox keinen direkten Zugriff auf `/workspace`. Dateien müssen über Gateway-Tools gelesen/geschrieben werden.
- ❌ **Kein Zugriff auf NAS-Dateien** außerhalb des Workspace – `/volume1/docker/openclaw/workspace` ist dein Spielfeld, sonst nichts.
- ❌ **Kein Docker-Socket** in der Sandbox – Du kannst keine Container starten, stoppen oder inspizieren.
- ❌ **Keine Gateway/Config-Änderungen** – Die Tools `gateway`, `cron` und `sessions_spawn` sind gesperrt.
- ❌ **Kein Root** – Du läufst als User 1000 (non-root).

### Für Internet-Zugriff (Web-Suche, Fetch):
Die Tools `web_search` und `web_fetch` funktionieren weiterhin **direkt über den Gateway** (nicht über die Sandbox). Wenn du eine Webseite abrufen oder suchen willst, nutze diese Tools normal – die laufen nicht in der Sandbox.

---

## Datenpfade

| Pfad (im Container) | Pfad (auf dem NAS) | Zweck |
|---|---|---|
| `~/.openclaw/sandboxes/` | Container-intern | Temporäre Sandbox-Arbeitsdaten |
| Sandbox-Daten | `/volume1/docker/openclaw/config/sandboxes/` | Temporäre Sandbox-Workspaces |

---

## Typische Workflows

### Datenanalyse
```
1. Daten im Workspace ablegen (CSV, JSON etc.)
2. Python-Skript schreiben
3. Ausführen → Ergebnis landet ebenfalls im Workspace
4. Plot als PNG speichern → Im Workspace verfügbar
```

### InfluxDB-Daten analysieren
```
1. Web-Fetch/API zum Abrufen der Daten (läuft über Gateway, nicht Sandbox)
2. Daten als CSV im Workspace speichern
3. pandas-Analyse in der Sandbox ausführen
```

---

## Was bei Problemen zu tun ist

- **"Package nicht verfügbar"**: Die Sandbox hat nur die vorinstallierten Pakete. Neue Pakete können nicht zur Laufzeit installiert werden (kein Netzwerk). Melde dem Besitzer, welches Paket fehlt → es wird ins Sandbox-Image eingebaut.
- **"Permission denied"**: Du läufst als User 1000. Dateien im Workspace sollten dir gehören.
- **"Network unreachable"**: Korrekt – die Sandbox hat kein Netzwerk. Nutze `web_fetch`/`web_search` über den Gateway für Internet-Zugriff.

---

## Technische Details (für Referenz)

```json
{
  "sandbox": {
    "mode": "non-main",
    "scope": "agent",
    "workspaceAccess": "none",
    "docker": {
      "image": "openclaw-sandbox:bookworm-slim",
      "network": "none",
      "readOnlyRoot": false,
      "user": "1000:1000"
    }
  }
}
```

**Image:** Eigenes Image basierend auf `debian:bookworm-slim` mit Python 3.11 + Data-Science-Stack. Kein `setupCommand` nötig – alles vorinstalliert.
