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

Inhalt
- Übersicht der Endpoints
- WebSocket — Live‑Feed
- REST‑Endpoints (Kurzliste)
- Response‑Schemas
  - Telegram Objekt (aus /api/telegrams)
  - KnxSend Request / Response (aus /api/knx/send)

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

---

Response‑Schemas

1) Telegram Objekt (wie von /api/telegrams und /api/telegrams/last zurückgegeben)

Bezeichnung: Telegram
Typ: JSON Objekt

Felder (aus Backend‑Serializer _build_telegram_response):
- timestamp (string, ISO8601) — Zeitstempel des Telegramms, z. B. "2026-07-22T09:12:34.123456Z"
- source_address (string) — Physikalische Absenderadresse (Individual Address), z. B. "1.1.10"
- target_address (string) — Zieladresse (Group Address), z. B. "1/2/3"
- direction (string) — Richtung (wenn relevant)
- telegram_type (string) — Technischer Typ, z. B. "GroupValueWrite"
- dpt_main (int|null) — Haupt‑DPT (z. B. 5)
- dpt_sub (int|null) — DPT‑Subtyp (z. B. 1)
- value_numeric (number|null) — Numerischer Wert (falls decodiert)
- value_json (any|null) — Strukturierter Payload (JSON) für komplexe DPTs
- raw_data (string|null) — Hex‑String der Rohdaten (ohne 0x), z. B. "0F3A..."
- source_name (string|null) — Name aus Projekt (oder null)
- target_name (string|null) — Name aus Projekt (oder null)
- simplified_type (string) — Kurztyp (UI‑friendly), z. B. "Write"/"Read"/"Response"
- dpt_name (string|null) — Menschlich lesbarer DPT‑Name (z. B. "5.xxx - Percent")
- unit (string|null) — Einheit für value_formatted (z. B. "%")
- value_formatted (string|null) — Formatiertes Anzeige‑Feld, bevorzugte Textdarstellung
- raw_hex (string|null) — raw_data mit optionaler 0x‑Präfix, z. B. "0x0F3A"

Beispiel (ein Telegram‑Objekt):

{
  "timestamp": "2026-07-22T09:12:34.123456Z",
  "source_address": "1.1.10",
  "target_address": "1/2/3",
  "direction": "inbound",
  "telegram_type": "GroupValueWrite",
  "dpt_main": 5,
  "dpt_sub": 1,
  "value_numeric": 42,
  "value_json": null,
  "raw_data": "0F3A",
  "source_name": "Living Room Sensor",
  "target_name": "Lighting Group",
  "simplified_type": "Write",
  "dpt_name": "5.001 - Percent",
  "unit": "%",
  "value_formatted": "42 %",
  "raw_hex": "0x0F3A"
}

Hinweis: /api/telegrams liefert ein Array dieser Objekte unter dem Key "telegrams"; zusätzlich wird ein metadata‑Objekt mit total_count und limit_reached zurückgegeben.

---

2) KnxSend Request / Response (POST /api/knx/send)

Request Body (JSON) — KnxSendRequest
- address (string) — Ziel‑GroupAddress, z. B. "1/2/3"
- payload (boolean|number|string|object) — Decodierter Wert für den angegebenen DPT (z. B. true, 21.5, "Hello"); für undekodierte Werte kann ein Raw‑Byte‑Array erwartet werden.
- dpt (string|null) — DPT als String im Format "main.sub", z. B. "5.001" oder null
- response (boolean) — Wenn true wird ein Response‑Telegramm statt Write gesendet (falls unterstützt)

Antwort (bei Erfolg)
- Status: 200 OK
- Body: { "status": "sent" }

Beispiel cURL (Boolean):

curl -X POST -H "Content-Type: application/json" \
  -d '{"address":"1/2/3","payload":true,"dpt":"1.001","response":false}' \
  http://localhost:8765/api/knx/send

Mögliche Fehlerantworten (FastAPI HTTPException)
- 400 Bad Request — bei ConversionError / invalid address / payload
  Beispiel: { "detail": "Invalid DPT value: ..." }
- 403 Forbidden — wenn Bus‑Writes deaktiviert (READ_ONLY oder ALLOW_WRITE=false)
  Beispiel: { "detail": "Sending to the KNX bus is disabled" }
- 409 Conflict — wenn der Dienst nicht mit dem KNX‑Bus verbunden ist
  Beispiel: { "detail": "Not connected to the KNX bus" }

---

Weiteres

Wenn du möchtest, erweitere ich diese Datei um:
- Vollständige JSON‑Schemas (JSON Schema Draft) für alle wichtigen Objekte
- Beispiele / cURL‑Snippets für weitere Endpoints (/api/export, /api/project/upload, /api/database/purge)
- Ein Inhaltsverzeichnis mit Links zu den Endpoint‑Abschnitten

Sag mir bitte, ob ich JSON‑Schemas erzeugen und die Beispielabschnitte erweitern soll, und welche Endpoints Priorität haben (z. B. /api/export, /api/project/upload).