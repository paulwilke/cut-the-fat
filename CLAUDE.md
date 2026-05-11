# Cut the Fat — Engineering Guidelines

## Projektübersicht

Persönliche Finanzanalyse. Privacy-first, lokal, kein Cloud-Zwang.
Kontoauszüge importieren → KI kategorisiert → Dashboard + Berichte + Sparempfehlungen.

**Team:** Paul Wilke (Fork-Owner) + Vlad Sabolotny (Original-Autor)
**Workflow:** Feature-Branches → Pull Requests → Review → Merge in `main`

---

## Drei Ausführungsebenen

```
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 1 — CLI  (./ctf)                                         │
│  Python + Click + Rich → direkter Service-Aufruf                │
├─────────────────────────────────────────────────────────────────┤
│  LAYER 2 — WEB  (./ctf-web)                                     │
│  FastAPI + WebSocket + vanilla HTML/JS → Browser oder Docker    │
├─────────────────────────────────────────────────────────────────┤
│  LAYER 3 — DESKTOP  (./ctf-desktop)                             │
│  Tauri 2 (Rust) wrappet den Web-Layer als nativer App           │
│  Python-Binary (PyInstaller) als Sidecar gebundled              │
└─────────────────────────────────────────────────────────────────┘
```

Alle drei Layer teilen dieselben Backend-Services (`backend/app/services/`) und
dieselbe SQLite-Datenbank. Die CLI ist die Single Source of Truth für Business-Logik.

---

## Schnellstart

```bash
cp .env.example .env   # ANTHROPIC_API_KEY + GITHUB_TOKEN setzen

./ctf --help           # CLI
./ctf-web              # Web-UI im Browser (http://localhost:8765)
./ctf-desktop          # Desktop-App (baut Sidecar + startet Tauri)
./ctf-desktop --skip-build  # Ohne Sidecar-Neubauen (Binary schon vorhanden)
```

---

## CLI-Befehle

```bash
./ctf upload <datei>             # CSV/Excel/PDF importieren
./ctf dashboard                  # Ausgabenübersicht (letzter Monat mit Daten)
./ctf dashboard --monat 2025-12
./ctf insights                   # KI-Sparempfehlungen (gecacht)
./ctf insights --neu             # Cache ignorieren
./ctf learn                      # Unkategorisierte Händler Q&A
./ctf learn --limit 50
./ctf report                     # Monatsbericht als Markdown
./ctf report --alle
```

---

## Entwicklungsbefehle

| Aufgabe | Befehl |
|---|---|
| CLI testen | `./ctf --help` |
| Web-UI starten | `./ctf-web` |
| Desktop-App starten | `./ctf-desktop` |
| Sidecar bauen | `node scripts/build-sidecar.mjs && node scripts/rename-sidecar.mjs` |
| Python venv | `cd backend && .venv/bin/pip install -r requirements.txt` |
| DB-Migration | `cd backend && .venv/bin/alembic upgrade head` |
| Migration erstellen | `cd backend && .venv/bin/alembic revision --autogenerate -m "beschreibung"` |
| Tests | `cd backend && .venv/bin/pytest tests/` |

---

## Repo-Struktur

```
cut-the-fat/
├── ctf                          # CLI-Einstiegspunkt
├── ctf-web                      # Web-Server-Einstiegspunkt
├── ctf-desktop                  # Desktop-App-Einstiegspunkt
│
├── cli/                         # CLI-Layer
│   ├── main.py                  # Click-Gruppe
│   ├── db.py                    # asyncio.run()-Wrapper über Services
│   ├── commands/                # upload.py, dashboard.py, insights.py, learn.py, report.py
│   └── render/                  # terminal.py, md_writer.py, tui_learn.py
│
├── web/                         # Web-Layer (FastAPI + WebSocket)
│   ├── app.py                   # Server-Einstiegspunkt (auch Sidecar-Entry)
│   ├── handlers/                # dashboard.py, insights.py, learn.py, report.py,
│   │                            #   bugreport.py, compare.py, explore.py, history.py
│   ├── logic/                   # formatter.py, processor.py
│   └── static/                  # index.html, chat.js, transactions.html/js, style.css
│
├── src-tauri/                   # Desktop-Layer (Tauri 2 / Rust)
│   ├── src/lib.rs               # Sidecar-Startup, Port-Discovery, IPC-Commands
│   ├── src/main.rs              # Entry point
│   ├── tauri.conf.json          # App-Konfiguration
│   ├── capabilities/default.json # Permissions
│   └── binaries/                # Sidecar-Binaries (gebaut, nicht committed)
│
├── scripts/
│   ├── build-sidecar.mjs        # PyInstaller-Wrapper
│   └── rename-sidecar.mjs       # Target-Triple-Rename für Tauri
│
├── backend/                     # Business-Logik (geteilt von CLI + Web + Desktop)
│   ├── cut_the_fat.db           # SQLite-Datenbank
│   ├── requirements.txt
│   ├── alembic/                 # DB-Migrationen
│   └── app/
│       ├── config.py            # Pydantic Settings
│       ├── database.py          # Async SQLAlchemy Engine
│       ├── models/              # Transaction, Upload, MerchantRule, InsightsCache, Category
│       ├── queries.py           # Geteilte DB-Queries
│       └── services/            # categorizer.py, insights.py, category_discovery.py, parser/
│
├── analytics/                   # Generierte Monatsberichte (Markdown)
├── data/statements/             # Originale Kontoauszüge
├── doc/                         # PRDs, Pläne, Patch-Dateien
└── .github/workflows/release.yml # CI/CD für Releases
```

---

## Architektur-Entscheidungen

### Python als Backend (Sidecar-Pattern)
Python bleibt das Backend. Die Web-App (`web/app.py`) wird per PyInstaller zu
einer Single-Binary kompiliert und von Tauri als Sidecar gestartet.

**Warum Python behalten:** Anthropic SDK, Pandas/tabula für Parser, bestehende
Logik läuft und ist getestet. Neuschreiben in Rust würde nichts Wesentliches
gewinnen ausser kleineren Binaries.

**Konsequenz:** Tauri-Sidecar-Binaries sind 50–80 MB (Python + alle Deps).
Das ist akzeptabel.

### Sidecar Port-Discovery
Port wird **nicht** hardcoded. Tauri wählt einen freien Port per `portpicker`,
übergibt ihn als Argument an den Sidecar, der `READY:<port>` auf stdout schreibt.
Rust fängt das Signal ab und gibt den Port per IPC (`invoke('get_backend_port')`)
ans Frontend weiter.

### Dual-Mode Frontend
`chat.js` erkennt automatisch ob es in Tauri oder im Browser läuft:
- **Tauri:** Port via `invoke('get_backend_port')`, URLs zu `localhost:PORT`
- **Browser:** gleicher Origin, relative URLs

Neuer Frontend-Code muss dieses Pattern respektieren:
```javascript
const isTauri = !!window.__TAURI_INTERNALS__;
const base = isTauri ? `http://localhost:${await getPort()}` : '';
```

### DB-Pfad
`config.py` leitet den DB-Pfad von `__file__` ab → immer `backend/cut_the_fat.db`,
unabhängig vom Arbeitsverzeichnis. Nicht ändern.

### Dedup-Mechanismen
- **Transaction-Dedup:** `dedup_hash = SHA-256(datum|merchant.lower()|betrag)` — nie ändern
- **Merchant-Dedup:** `merchant_normalized = lowercase + Sonderzeichen entfernen`
- **Insights-Cache:** Key auf `SHA-256(aggregierter Ausgaben-JSON)`

---

## Release-Pipeline

Releases werden durch Git-Tags ausgelöst (`git tag v1.2.3 && git push --tags`).

### Was GitHub Actions macht (`.github/workflows/release.yml`)

**Stage 1 — Sidecar-Build (Matrix: Linux, macOS, Windows):**
1. Python-Dependencies installieren + PyInstaller
2. `web/app.py` → Single-Binary `ctf-sidecar`
3. Mit Target-Triple umbenennen: `ctf-sidecar-aarch64-apple-darwin`
4. Als Artefakt hochladen

**Stage 2 — Tauri-Build (Matrix: Linux, macOS, Windows):**
1. Sidecar-Artefakte downloaden
2. `tauri-action` baut plattformspezifische Pakete:
   - Linux: `.deb` + `.AppImage`
   - macOS: `.dmg` + `.app`
   - Windows: `.msi` + `.exe`
3. Erstellt GitHub Release (Draft) mit allen Paketen

**Benötigte GitHub Secrets:**
- `GITHUB_TOKEN` — automatisch vorhanden
- `TAURI_SIGNING_PRIVATE_KEY` — für Auto-Updater-Signatur (siehe unten)

### Auto-Updater aktivieren

Aktuell deaktiviert (`"createUpdaterArtifacts": false` in `tauri.conf.json`).

Aktivieren:
1. Signing-Key generieren: `cargo tauri signer generate -w ~/.tauri/cutthefat.key`
2. Public Key in `tauri.conf.json` → `plugins.updater.pubkey` eintragen
3. Private Key als GitHub Secret `TAURI_SIGNING_PRIVATE_KEY` setzen
4. `"createUpdaterArtifacts": true` in `tauri.conf.json`
5. Update-Endpoint-URL konfigurieren (GitHub Releases JSON)

---

## Bug-Reporting

`web/handlers/bugreport.py` erstellt GitHub Issues direkt via API.

**Voraussetzung:** `GITHUB_TOKEN` in `.env` (Personal Access Token mit `issues: write`)

**Workflow:**
1. User klickt 🐛-Button in der Desktop-App
2. Modal: Titel + Beschreibung + letzte 50 Chat-Nachrichten als Kontext
3. POST `/api/bugreport` → GitHub Issue in `paulwilke/cut-the-fat`
4. Repo überschreibbar via `GITHUB_REPO=owner/name` in `.env`

**Sentry (geplant, noch nicht implementiert):**
- Python-Sidecar: `sentry-sdk` mit `FastAPIIntegration`
- Frontend: Sentry Browser SDK in `index.html`
- Sentry-Fehler können automatisch als GitHub Issues exportiert werden
  (Sentry → Integrations → GitHub)

---

## Datentransparenz (Privacy-Anforderung)

Wenn die App Daten an Anthropic sendet, muss das sichtbar sein:
- Kategorisierung: Händler-Liste im Prompt → UI markiert mit ⚠️
- Insights: Aggregiertes Ausgaben-JSON → UI zeigt konkreten Payload auf Anfrage

Ohne `ANTHROPIC_API_KEY` in `.env`: keine externen Calls, Fallback-Werte.

---

## Schlüsseldateien

| Datei | Zweck |
|---|---|
| `backend/app/models/transaction.py` | `CATEGORIES` — einzige Quelle der Wahrheit |
| `backend/app/services/categorizer.py` | Claude Haiku Batch-Kategorisierung |
| `backend/app/services/insights.py` | Claude Sonnet Insights + SHA-256-Cache |
| `backend/app/services/parser/` | CSV/Excel/PDF-Parser |
| `backend/app/queries.py` | Geteilte DB-Queries (CLI + Web) |
| `cli/db.py` | sync-wrappte async DB-Funktionen für CLI |
| `web/app.py` | FastAPI-Server + Sidecar-Entry |
| `web/handlers/bugreport.py` | GitHub Issues API |
| `web/static/chat.js` | Dual-Mode Frontend (Tauri + Browser) |
| `src-tauri/src/lib.rs` | Sidecar-Startup, Port-Discovery, IPC |
| `src-tauri/tauri.conf.json` | Tauri App-Konfiguration |
| `.github/workflows/release.yml` | CI/CD Release-Pipeline |

---

## Kategorien (kanonisch, Deutsch)

`Wohnen, Lebensmittel, Essen & Trinken, Mobilität, Freizeit, Gesundheit, Drogerie, Shopping, Abonnements, Urlaub, Bildung, Kommunikation, Versicherungen, Kinder, Post & Versand, Business Natalie, Kinder Natalie, Wohnen Natalie, Einnahmen, Einnahmen Natalie, Einkommensteuer, PayPal, Bargeld, Kreditkarte, Eigenüberweisung, Sonstiges`

Neue Kategorie: `CATEGORIES` in `backend/app/models/transaction.py` — wird beim nächsten Start automatisch in die DB gesetzt.

---

## Neuen Bank-Parser hinzufügen

1. `backend/app/services/parser/<bank>_parser.py` → `parse_<bank>(content: bytes) -> list[RawTransaction]`
2. Format-Erkennung in `cli/db.py` → `_ingest_file()` ergänzen
3. Mit echter Beispieldatei testen (`./ctf upload <testdatei>`)

---

## Häufige Fehler

- **PDF-Parser:** `_extract_from_tables()` zuerst, dann Regex-Fallback. Echte PDFs variieren stark.
- **Datumsformate:** `DATE_FORMATS` in `csv_parser.py` ergänzen falls nötig
- **Betragsvorzeichen:** alle `amount`-Werte positiv; `type`-Spalte (`debit`/`credit`) trägt das Vorzeichen
- **lru_cache auf get_settings():** nach `.env`-Änderungen Prozess neu starten
- **Async SQLAlchemy:** immer `await db.execute(...)`, nie synchrone ORM-Muster
- **Sidecar nicht gefunden:** `node scripts/build-sidecar.mjs && node scripts/rename-sidecar.mjs` ausführen
- **Tauri CSP:** neue externe Hosts in `tauri.conf.json` → `app.security.csp` eintragen

---

## Was NICHT tun

- `dedup_hash`-Algorithmus nicht ändern — bestehende Zeilen verlieren Duplikatschutz
- Python-Backend nicht durch Rust ersetzen — Sidecar-Pattern ist bewusste Entscheidung
- SQLite nicht durch Postgres ersetzen — Null-Server, Daten bleiben lokal
- FastAPI nicht aus `web/app.py` entfernen — ist der Sidecar-Entry-Point
- Sidecar-Binaries nie committen — `.gitignore` schützt `src-tauri/binaries/`
- Kein hardcoded Port — immer Port-Discovery via Sidecar-stdout-Signal

---

## PRD-Format

Neue Features werden als PRD in `doc/PRD_<feature>.md` dokumentiert, bevor
Code geschrieben wird. Mindest-Inhalte:

```markdown
## Kontext
Warum bauen wir das? Welches Problem löst es?

## Produktziele
Was soll das Feature leisten? (2–4 Punkte)

## Nicht-Ziele
Was ist explizit NICHT im Scope?

## Akzeptanzkriterien
Konkrete, testbare Bedingungen für "fertig"

## Technische Notizen
Layer-Zuordnung (CLI / Web / Tauri), neue Abhängigkeiten, Breaking Changes
```

---

## Roadmap (grob)

| Phase | Thema | Status |
|---|---|---|
| 1 | CLI (Core Business-Logik) | ✅ fertig |
| 2 | Web-UI (Chat + Dashboard) | ✅ fertig |
| 3 | Tauri Desktop-App | 🔄 in Arbeit (Sidecar läuft, Auto-Updater fehlt) |
| 4 | Auto-Updater aktivieren | ⬜ offen |
| 5 | Sentry-Integration | ⬜ offen |
| 6 | Alpha-Release + Bug Reports | ⬜ offen |
| 7 | Mobile (PWA oder Tauri Mobile) | ⬜ later |
