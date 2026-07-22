# Spectrum KNX — API Kurz‑Dokumentation

Diese Datei fasst die im Backend registrierten REST‑ und WebSocket‑Endpunkte zusammen. Sie wurde automatisch aus dem Code (backend/api.py) erstellt — prüfe bitte lokal auf Vollständigkeit und passe Beispiele nach Bedarf an.

Quelle: backend/api.py
(Generiert aus dem Backend-Code; prüfe bitte auf Vollständigkeit)

Hinweis
- Diese Dokumentation basiert auf dem API‑Router in backend/api.py. Sie beschreibt die registrierten REST‑ und WebSocket‑Routen, ihre Parameter und typische Antworten. Fehlerantworten enthalten in der Regel das Feld `detail` (FastAPI).
- Zeitstempel sind ISO8601 (UTC). Die meisten POST/GET-Antworten sind JSON; Uploads nutzen multipart/form-data.
- Endpoints, die Änderungen am KNX‑Bus auslösen (Senden/Read), prüfen Laufzeit‑Berechtigungen (READ_ONLY, ALLOW_WRITE) und den Verbindungsstatus.

Basis
- Base path: /api
- WebSocket path: /ws/telegrams

WebSocket — Live‑Feed
- Endpoint: ws://<host>/ws/telegrams (wss:// bei https)
- Verhalten:
  - Beim Verbinden sendet der Server eine initiale Nachricht vom Typ `connection_state` (connected/disconnected + timestamp).
  - Clients können Filter als JSON senden; der Server akzeptiert JSON‑Objekte und aktualisiert die Filter.
  - Beispiel (JS):

    const ws = new WebSocket("ws://localhost:8765/ws/telegrams");
    ws.onmessage = (e) => console.log(JSON.parse(e.data));
    // Filtersenden (z. B. nur GAs A und B)
    ws.send(JSON.stringify({ target_address: "1/2/3,1/2/4" }));

REST‑Endpoints (Kurzliste)
- GET /api/version
  - Liefert die Backend‑Version.
  - Response: { "version": "..." }

- GET /api/update
  - Liefert Update‑Check / Release‑Infos.
  - Response: { enabled, current, latest, update_available, ... }

- GET /api/telegrams
  - History / Suche (absteigend nach Zeit).
  - Query-Parameter:
    - limit, offset
    - source_address (komma-separiert)
    - target_address (komma-separiert)
    - telegram_type (komma; UI: Write,Read,Response)
    - dpt_main (z. B. "5" oder "5.001", komma-separiert)
    - start_time, end_time (ISO datetime)
    - delta_before_ms, delta_after_ms
  - Response: { "telegrams": [...], "metadata": { total_count, limit, offset, limit_reached } }

- GET /api/telegrams/last
  - Letztes Telegramm pro GA (aggregation). Optional: target_address filter.
  - Response: { "telegrams": [...] }

- GET /api/filter-options
  - Liefert Listen für die UI: sources, targets, types, dpts, ga_group_names, pa_line_names.

- GET /api/statistics
  - Aggregierte Zählstatistiken: { total, by_ga, by_pa }

- GET /api/database/info
  - DB‑Statistiken & Fähigkeiten (size_bytes, telegram_count, retention_days, supports_optimize, read_only, ...)

- POST /api/database/purge
  - Löschen alter oder aller Telegramme; unterstützt dry_run.
  - Body: { "older_than": "ISO datetime" | null, "purge_all": bool, "dry_run": bool }
  - Response: { "deleted": N, "dry_run": bool }

- POST /api/database/optimize
  - DB‑Optimierung / Reclaim space.
  - Response: { "size_bytes_before": N, "size_bytes_after": M }

- GET /api/project
  - Geladenes KNX‑Projekt (group_addresses, devices) oder Status no_project_loaded.

- GET /api/project/status
  - Status der Upload/Projekt‑Funktion (upload_writable, project_loaded, upload_required).

- POST /api/project/upload
  - Upload einer .knxproj + password (multipart/form-data). Triggert Reload.
  - Response: { "status": "ok", "message": "Project loaded successfully" } oder HTTP Fehler.

- GET /api/server/config
  - Effektive Serverkonfiguration (Passwörter maskiert).

- GET /api/knxkeys/status
  - Status des knxkeys Upload‑Features.

- POST /api/knxkeys/upload
  - Upload einer .knxkeys Datei + password; schreibt Datei und reconnect.

- POST /api/knx/send
  - Senden eines GroupValueWrite/Response an die GA (nur wenn Writes erlaubt).
  - Body: { "address": "1/2/3", "payload": Any, "dpt": "5.001" | null, "response": bool }
  - Response: { "status": "sent" }
  - Fehler: 403 (disabled), 409 (not connected), 400 (invalid payload/address)

- POST /api/knx/read
  - Sendet GroupValueRead an Adresse.
  - Body: { "address": "1/2/3" }
  - Response: { "status": "sent" }

- POST /api/knx/send/scheduled
  - Startet verzögertes / zyklisches Senden (ein Job gleichzeitig).
  - Body: { address, payload, dpt, response, delay_seconds, interval_seconds }
  - Response: Job‑Dict (state, id, next_send_at, ...)

- GET /api/knx/send/scheduled/status
- POST /api/knx/send/scheduled/cancel

- GET /api/import/status
  - Status des Telegram‑Importjobs (gibt read_only zurück).

- POST /api/import
  - Upload eines Telegram-Logs (.xml oder .zip) und Start eines Hintergrundimports.

- POST /api/import/cancel

- GET /api/export
  - Streams matching telegrams als ETS6 XML (StreamingResponse).
  - Query: source_address, target_address, telegram_type, start_time, end_time, limit

Fehlercodes (typisch)
- 400: Validation / Bad request
- 403: Forbidden (z. B. read-only, upload disabled)
- 404: Not found (z. B. cancel ohne Job)
- 409: Conflict (z. B. not connected, job exists)
- Fehlerantworten enthalten meist { "detail": "..." }

Nächste Schritte (nach Commit)
- Auf Wunsch ergänze ich die README mit vollständigen Response‑Schemas (z. B. Struktur eines Telegram-Objekts) und Beispiel‑Payloads (cURL).  
- Alternativ kann ich eine OpenAPI‑YAML erzeugen (für Swagger / ReDoc).
