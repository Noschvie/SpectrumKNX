# Spectrum KNX — API Kurz‑Dokumentation

Diese Datei fasst die im Backend registrierten REST‑ und WebSocket‑Endpunkte zusammen. Sie wurde automatisch aus dem Code (backend/api.py) erstellt — prüfe bitte lokal auf Vollständigkeit und Korrektheit.

Quelle: backend/api.py
(Generiert aus dem Backend-Code; prüfe bitte auf Vollständigkeit)

Hinweis
- Diese Dokumentation basiert auf dem API‑Router in backend/api.py. Sie beschreibt die registrierten REST‑ und WebSocket‑Routen, ihre Parameter und typische Antworten. Fehlerantworten enthalten ein `detail` Feld mit Fehlermeldung.
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
  - Projekt Objekt (aus /api/project)
  - Import/Export Responses

WebSocket — Live‑Feed
- Endpoint: ws://<host>/ws/telegrams (wss:// bei https)
- Verhalten:
  - Beim Verbinden sendet der Server eine initiale Nachricht vom Typ `connection_state` (connected/disconnected + timestamp).
  - Clients können Filter als JSON senden; der Server akzeptiert JSON‑Objekte und aktualisiert die Filter.
  - Beispiel (JS):

    ```javascript
    const ws = new WebSocket("ws://localhost:8765/ws/telegrams");
    ws.onmessage = (e) => console.log(JSON.parse(e.data));
    // Filtersenden (z. B. nur GAs A und B)
    ws.send(JSON.stringify({ target_address: "1/2/3,1/2/4" }));
    ```

REST‑Endpoints (Kurzliste)
- GET /api/version
  - Liefert die Backend‑Version.
  - Response: { "version": "..." }

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

- POST /api/database/optimize
  - DB‑Optimierung / Reclaim space.
  - Response: { "size_bytes_before": N, "size_bytes_after": M }

- POST /api/database/purge
  - Löscht ältere oder alle Telegramme; unterstützt `dry_run` zur Voransicht.
  - Body: { "older_than": "ISO datetime" | null, "purge_all": bool, "dry_run": bool }
  - Response: { "deleted": <number>, "dry_run": <bool> }
  - Fehler: 403 Forbidden (wenn read-only)

- GET /api/project/status
  - Status der Upload/Projekt‑Funktion (upload_writable, project_loaded, upload_required).

- GET /api/project
  - Liefert das aktuell geladene KNX‑Projekt (group_addresses, devices) oder Status `no_project_loaded`.
  - Response: JSON‑Objekt mit `project_loaded` (bool) und optional `group_addresses` / `devices` Arrays

- POST /api/project/upload
  - Upload einer .knxproj + password (multipart/form-data). Triggert Reload.
  - Request: multipart/form-data mit `file` (die .knxproj) und `password` (Form‑Field, optional)
  - Response: { "status": "ok", "message": "Project loaded successfully" } oder HTTP Fehler (400/403)

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
  - Status des aktuellen Sendejobs

- POST /api/knx/send/scheduled/cancel
  - Aktuellen Sendjob abbrechen

- POST /api/import
  - Upload eines Telegram‑Logs (.xml oder .zip mit .xml) und Start eines Hintergrund‑Imports.
  - Request: multipart/form-data mit `file=@telegrams.xml`
  - Response: Job‑Info (z. B. { "job_id": "...", "state": "running" })
  - Fehler: 403 (in read-only mode)

- GET /api/import/status
  - Status des Telegram‑Importjobs (gibt read_only zurück).

- POST /api/import/cancel
  - Bricht den laufenden Importjob ab.

- GET /api/export
  - Streamt passende Telegramme als ETS6-kompatible CommunicationLog XML (StreamingResponse).
  - Query-Parameter: `source_address`, `target_address`, `telegram_type`, `start_time`, `end_time`, `limit`
  - Response: XML-Datei als Attachment (Content-Disposition: attachment)

- GET /api/update
  - Liefert Informationen zu verfügbaren Releases und Metadaten (Update‑Popup).
  - Response: Objekt mit `enabled`, `current`, `latest`, `update_available`, `html_url`, `releases`

Fehlercodes (typisch)
- 400: Validation / Bad request (z. B. ungültige Adresse, Payload-Fehler)
- 403: Forbidden (z. B. read-only, upload disabled, Writes deaktiviert)
- 404: Not found (z. B. cancel ohne aktiven Job)
- 409: Conflict (z. B. not connected to KNX bus, Job existiert bereits)
- Fehlerantworten enthalten: { "detail": "Fehlermeldung" }

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

```json
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
```

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

```bash
curl -X POST -H "Content-Type: application/json" \
  -d '{"address":"1/2/3","payload":true,"dpt":"1.001","response":false}' \
  http://localhost:8765/api/knx/send
```

Mögliche Fehlerantworten (FastAPI HTTPException)
- 400 Bad Request — bei ConversionError / invalid address / payload
  Beispiel: { "detail": "Invalid DPT value: ..." }
- 403 Forbidden — wenn Bus‑Writes deaktiviert (READ_ONLY oder ALLOW_WRITE=false)
  Beispiel: { "detail": "Sending to the KNX bus is disabled" }
- 409 Conflict — wenn der Dienst nicht mit dem KNX‑Bus verbunden ist
  Beispiel: { "detail": "Not connected to the KNX bus" }

---

3) Projekt Objekt (GET /api/project)

Response bei geladenem Projekt:
```json
{
  "project_loaded": true,
  "group_addresses": [
    {
      "address": "1/2/3",
      "name": "Lighting Group",
      "dpt_main": 5,
      "dpt_sub": 1
    }
  ],
  "devices": [
    {
      "physical_address": "1.1.10",
      "name": "Living Room Sensor",
      "product_name": "KNX Device"
    }
  ]
}
```

Response bei keinem Projekt:
```json
{
  "project_loaded": false,
  "message": "no_project_loaded"
}
```

---

4) Datenbankoptimierung Response (POST /api/database/optimize)

Response nach erfolgreicher Optimierung:
```json
{
  "size_bytes_before": 10485760,
  "size_bytes_after": 8388608,
  "space_freed_bytes": 2097152
}
```

Fehler: 403 Forbidden (wenn read-only mode)

---

5) Datenbank Purge Response (POST /api/database/purge)

Request Body:
```json
{
  "older_than": "2025-01-01T00:00:00Z",
  "purge_all": false,
  "dry_run": false
}
```

Response:
```json
{
  "deleted": 1250,
  "dry_run": false,
  "deleted_from_timestamp": "2025-01-01T00:00:00Z"
}
```

Dry-run Beispiel (prüft ohne zu löschen):
```json
{
  "deleted": 1250,
  "dry_run": true,
  "message": "This is a dry run. 1250 telegrams would be deleted."
}
```

Fehler: 403 Forbidden (wenn read-only mode)

---

6) Import Response (POST /api/import)

Response bei erfolgreicher Upload:
```json
{
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "state": "running",
  "filename": "telegrams.xml",
  "file_size_bytes": 5242880,
  "progress": 0,
  "message": "Import started"
}
```

Import Status Response (GET /api/import/status):
```json
{
  "state": "running",
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "progress": 35,
  "telegrams_imported": 3500,
  "total_telegrams": 10000,
  "read_only": false,
  "start_time": "2026-07-22T09:00:00Z",
  "elapsed_seconds": 125
}
```

Fehler: 403 Forbidden (wenn read-only mode), 404 Not Found (kein aktiver Import)

---

7) Export Response (GET /api/export)

Query-Parameter Beispiel:
```bash
GET /api/export?start_time=2025-01-01T00:00:00Z&end_time=2025-01-02T00:00:00Z&target_address=1/2/3&limit=10000
```

Response: XML-Stream (ETS6 CommunicationLog Format)
- HTTP Header: Content-Disposition: attachment; filename=telegrams.xml
- Content-Type: application/xml
- Body: ETS6-kompatibles XML‑Format mit allen gefilterten Telegrammen

Beispiel Export-Datei (vereinfacht):
```xml
<?xml version="1.0" encoding="utf-8"?>
<CommunicationLog>
  <Telegrams>
    <Telegram>
      <Timestamp>2026-07-22T09:12:34.123456Z</Timestamp>
      <SourceAddress>1.1.10</SourceAddress>
      <TargetAddress>1/2/3</TargetAddress>
      <GroupValueWrite />
      <Data>0F3A</Data>
    </Telegram>
  </Telegrams>
</CommunicationLog>
```

Fehler: 400 Bad Request (ungültige Parameter)

---

8) Update Response (GET /api/update)

Response Struktur:
```json
{
  "enabled": true,
  "current_version": "1.2.3",
  "latest_version": "1.3.0",
  "update_available": true,
  "html_url": "https://github.com/Noschvie/SpectrumKNX/releases/tag/v1.3.0",
  "releases": [
    {
      "tag_name": "v1.3.0",
      "name": "Version 1.3.0",
      "body": "Release notes...",
      "created_at": "2026-07-20T12:00:00Z",
      "assets": [
        {
          "name": "spectrumknx-1.3.0.zip",
          "download_url": "https://github.com/.../releases/download/v1.3.0/spectrumknx-1.3.0.zip"
        }
      ]
    }
  ]
}
```

---

9) Filter Options Response (GET /api/filter-options)

Response:
```json
{
  "sources": [
    { "address": "1.1.10", "name": "Living Room Sensor" },
    { "address": "1.1.20", "name": "Bedroom Sensor" }
  ],
  "targets": [
    { "address": "1/2/3", "name": "Lighting Group" },
    { "address": "1/2/4", "name": "Temperature Group" }
  ],
  "types": ["Write", "Read", "Response"],
  "dpts": [
    { "main": 1, "sub": 1, "name": "1.001 - Boolean" },
    { "main": 5, "sub": 1, "name": "5.001 - Percent" }
  ],
  "ga_group_names": ["Lighting", "Temperature", "Security"],
  "pa_line_names": ["Area 1", "Area 2", "Area 3"]
}
```

---

10) Statistics Response (GET /api/statistics)

Response:
```json
{
  "total": 250000,
  "by_ga": {
    "1/2/3": 5420,
    "1/2/4": 3820,
    "1/2/5": 2150
  },
  "by_pa": {
    "1.1.10": 8500,
    "1.1.20": 6200,
    "1.1.30": 4500
  },
  "by_type": {
    "Write": 150000,
    "Read": 75000,
    "Response": 25000
  }
}
```

---

11) Database Info Response (GET /api/database/info)

Response:
```json
{
  "size_bytes": 104857600,
  "telegram_count": 250000,
  "retention_days": 30,
  "supports_optimize": true,
  "read_only": false,
  "last_optimized": "2026-07-20T15:30:00Z",
  "database_type": "SQLite",
  "version": "3.x.x"
}
```

---

CURL Beispiele für häufige Operationen

**1. Letzte Telegramme abrufen:**
```bash
curl "http://localhost:8765/api/telegrams?limit=50&offset=0"
```

**2. Telegramme in Zeitbereich filtern:**
```bash
curl "http://localhost:8765/api/telegrams?start_time=2026-07-22T00:00:00Z&end_time=2026-07-22T23:59:59Z"
```

**3. Wert senden:**
```bash
curl -X POST -H "Content-Type: application/json" \
  -d '{"address":"1/2/3","payload":42,"dpt":"5.001","response":false}' \
  http://localhost:8765/api/knx/send
```

**4. Leseanfrage:**
```bash
curl -X POST -H "Content-Type: application/json" \
  -d '{"address":"1/2/3"}' \
  http://localhost:8765/api/knx/read
```

**5. Geplantes Senden (mit Verzögerung):**
```bash
curl -X POST -H "Content-Type: application/json" \
  -d '{"address":"1/2/3","payload":true,"dpt":"1.001","delay_seconds":5}' \
  http://localhost:8765/api/knx/send/scheduled
```

**6. Projekt hochladen:**
```bash
curl -F "file=@myproject.knxproj" -F "password=secret" \
  http://localhost:8765/api/project/upload
```

**7. Telegramme exportieren:**
```bash
curl -L "http://localhost:8765/api/export?start_time=2025-01-01T00:00:00Z&end_time=2025-01-02T00:00:00Z" \
  -o telegrams.xml
```

**8. Telegramme importieren:**
```bash
curl -F "file=@telegrams.xml" \
  http://localhost:8765/api/import
```

**9. Datenbank-Informationen:**
```bash
curl http://localhost:8765/api/database/info
```

**10. Datenbank optimieren:**
```bash
curl -X POST http://localhost:8765/api/database/optimize
```

**11. Alte Telegramme löschen (dry-run):**
```bash
curl -X POST -H "Content-Type: application/json" \
  -d '{"older_than":"2025-01-01T00:00:00Z","dry_run":true}' \
  http://localhost:8765/api/database/purge
```

**12. WebSocket verbinden (JavaScript):**
```javascript
const ws = new WebSocket("ws://localhost:8765/ws/telegrams");
ws.onopen = () => console.log("Connected to WebSocket");
ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  console.log("Received telegram:", data);
};
ws.onerror = (error) => console.error("WebSocket error:", error);
ws.onclose = () => console.log("WebSocket closed");
```

---

**Hinweise zur Dokumentation:**
- Diese Dokumentation wird regelmäßig aktualisiert basierend auf dem Code in `backend/api.py`.
- Für die neueste API-Version und detailliertere Implementation Details siehe die Backend-Quelle.
- Bei Fragen oder Fehlern in der Dokumentation bitte ein Issue im Repository erstellen.
