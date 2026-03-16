# Kommunikation mit Schlaubi (OC Agent)

> **Zuletzt aktualisiert:** 16.03.2026

## Übersicht

Schlaubi ist der OpenClaw-Agent auf dem NAS, erreichbar über Telegram (@BenkisSchlaubiBot).  
Copilot kann über die OpenClaw CLI direkt mit Schlaubi kommunizieren – per SSH auf dem NAS.

---

## Copilot → Schlaubi (Nachrichten senden)

### Variante 1: Agent-Kommando (empfohlen)

Synchrones Request/Response – Copilot sendet eine Nachricht und bekommt Schlaubis Antwort direkt zurück:

```bash
ssh Benki-NAS3 "sudo /var/packages/ContainerManager/target/usr/bin/docker exec openclaw-gateway \
  openclaw agent -m 'Deine Nachricht hier' --session-id SESSION_ID --json"
```

**Mit Telegram-Zustellung** (Schlaubi antwortet UND sendet die Antwort an Telegram):
```bash
ssh Benki-NAS3 "sudo /var/packages/ContainerManager/target/usr/bin/docker exec openclaw-gateway \
  openclaw agent -m 'Deine Nachricht hier' --session-id SESSION_ID --deliver --json"
```

### Variante 2: Einfache Nachricht senden (fire & forget)

```bash
ssh Benki-NAS3 "sudo /var/packages/ContainerManager/target/usr/bin/docker exec openclaw-gateway \
  openclaw message send --channel telegram --target 1960260229 --message 'Nachricht hier'"
```

### Bekannte Session-Daten

| Parameter | Wert |
|---|---|
| Telegram Chat-ID | `1960260229` |
| Session-Key | `agent:main:telegram:slash:1960260229` |
| Bot | `@BenkisSchlaubiBot` |

### Sessions auflisten

```bash
ssh Benki-NAS3 "sudo /var/packages/ContainerManager/target/usr/bin/docker exec openclaw-gateway \
  openclaw sessions --active 60 --json"
```

---

## Schlaubi → Copilot (Rückkanal)

### Direkt: Nicht möglich

Schlaubi hat **keinen direkten Zugriff** auf VS Code oder GitHub Copilot. Er kann nicht proaktiv eine Nachricht an Copilot schreiben.

### Indirekt: Über Dateien im Workspace

Schlaubi kann über Gateway-Tools (`fs_write`) Dateien in den Workspace schreiben. Copilot kann diese per SSH lesen:

```bash
# Schlaubi schreibt z.B. in: /volume1/docker/openclaw/workspace/nachricht_an_copilot.md
# Copilot liest:
ssh Benki-NAS3 "cat /volume1/docker/openclaw/workspace/nachricht_an_copilot.md"
```

**Workflow für asynchrone Kommunikation:**
1. Copilot bittet Schlaubi per `openclaw agent`, eine Antwort in eine Datei zu schreiben
2. Schlaubi schreibt die Datei über `fs_write`
3. Copilot liest die Datei per SSH

### Beste Methode: Synchrones Agent-Kommando

Der einfachste Weg ist `openclaw agent -m "..." --json`. Copilot sendet eine Frage, Schlaubi verarbeitet sie, und die Antwort wird direkt als JSON zurückgegeben. Das entspricht einem normalen Request/Response-Pattern.

---

## Einschränkungen

- **Timeout**: Default 600s (10 Min). Für lange Tasks `--timeout` erhöhen.
- **Kein Streaming**: Die Antwort kommt erst nach vollständiger Verarbeitung.
- **Session-Kontext**: Mit `--session-id` bleibt der Gesprächskontext erhalten. Ohne wird eine neue Session erstellt.
- **Sandbox-Modus**: Schlaubis Code-Ausführung läuft in einer isolierten Docker-Sandbox (kein Netzwerk, kein Root). Details: [sandbox_handover.md](sandbox_handover.md)
