## Endpoints, die derzeit nicht detailliert behandelt werden

Die folgenden Endpoints sind im API‑Router vorhanden, werden hier aber aktuell nur als Verweis gelistet; ihre detaillierten Response‑Schemas und Beispiele folgen später.

- GET /api/export
  - Zweck: Streamt passende Telegramme als ETS6-kompatible CommunicationLog XML (StreamingResponse).
  - Wichtige Query‑Parameter: `source_address`, `target_address`, `telegram_type`, `start_time`, `end_time`, `limit`
  - Typische Nutzung: Download großer Exportdateien; Response ist ein XML‑Attachment (Content‑Disposition).
  - Kurzes Beispiel:
    ```bash
    curl -L "http://localhost:8765/api/export?start_time=2025-01-01T00:00:00Z&end_time=2025-01-02T00:00:00Z" -o export.xml
    ```

- POST /api/project/upload
  - Zweck: Upload einer ETS `.knxproj` Datei (multipart/form-data) plus optionales Passwort; speichert Projekt und triggert ein Reload, damit Filter / Namen verfügbar sind.
  - Request: multipart/form-data mit `file` (die .knxproj) und `password` (Form‑Field, optional)
  - Mögliche Antworten:
    - 200 OK: `{ "status": "ok", "message": "Project loaded successfully" }`
    - 400 / 403: Fehler bei Validierung oder Schreibrechten
  - Kurzes Beispiel:
    ```bash
    curl -F "file=@myproject.knxproj" -F "password=secret" http://localhost:8765/api/project/upload
    ```

- POST /api/database/purge
  - Zweck: Löschen älterer Telegramme oder aller Telegramme; unterstützt `dry_run` zur Voransicht.
  - Request (JSON):
    - `{ "older_than": "ISO datetime" | null, "purge_all": bool, "dry_run": bool }`
  - Response: `{ "deleted": <number>, "dry_run": <bool> }`
  - Hinweise: Endpoint prüft, ob der Store nicht read-only ist; `purge_all` löscht komplett.
  - Kurzes Beispiel:
    ```bash
    curl -X POST -H "Content-Type: application/json" \
      -d '{"older_than":"2025-01-01T00:00:00Z","dry_run":true}' \
      http://localhost:8765/api/database/purge
    ```

Hinweis: Diese Liste ist bewusst kurz gehalten — wenn du möchtest, schreibe ich für jeden Endpoint vollständige Response‑Schemas (inkl. Fehlerbeispiele) und erweitere die Dokumentation um konkrete Beispiel‑Antworten und JSON‑Schemas.
