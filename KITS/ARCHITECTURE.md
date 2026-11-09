# KITS – KI Tax Search

## Architektur und Entwicklungsplan (Version 1)

**Status:** Nur Planung – keine Implementierung  
**Datum:** 9. November 2026  
**Zielplattform:** native macOS-App (Swift / SwiftUI)  
**Sprache UI:** Deutsch  

---

## 1. Prüfung der Entwicklungsumgebung

### Ergebnis dieser Cloud-Umgebung

| Werkzeug | Status | Hinweis |
|---|---|---|
| Betriebssystem | Ubuntu 24.04.5 LTS (Linux x86_64) | **Kein macOS** |
| Xcode | **nicht installiert** | `xcodebuild` / `xcode-select` nicht vorhanden |
| Swift-Toolchain | **nicht installiert** | `swift` / `swiftc` nicht vorhanden |
| Apple Command Line Tools | **nicht verfügbar** | macOS-only |
| Clang / GCC / Make / CMake | vorhanden | für Linux-Build, nicht für macOS-App |
| Git | vorhanden | Versionskontrolle möglich |

### Konsequenz

KITS kann in dieser Cloud-Umgebung **architektonisch und dokumentarisch** vorbereitet werden.  
Kompilieren, Signieren, Sandbox-Entitlements und UI-Tests erfordern einen **Mac mit Xcode**.

### Empfohlene lokale Mac-Voraussetzungen (Zielmaschine)

| Komponente | Mindestempfehlung |
|---|---|
| macOS | 14 Sonoma oder neuer (idealerweise 15 Sequoia) |
| Xcode | 16.x |
| Swift | 5.9+ (besser Swift 6 / Xcode 16) |
| Deployment Target | macOS 14.0+ |
| Account | Apple Developer (für Notarisierung später; lokal signieren mit Personal Team möglich) |
| Festplatte | ausreichend Platz für Anhänge-Cache und SQLite |

**Checkliste auf dem Mac (vor Implementierungsstart):**

```bash
xcode-select -p
xcodebuild -version
swift --version
xcrun --find swift
# Optional: SPM-Auflösung prüfen
```

---

## 2. Projektarchitektur (Vorschlag)

### Leitidee

Lokaler, datenschutzfreundlicher Dokumenten-Assistent:

- Quellen: WEB.DE (IMAP), ausgewählte iCloud-Drive-Ordner, lokale Mac-Ordner  
- Verarbeitung: Indexierung, Steuerjahr-Erkennung, PDF-/Link-Erkennung, OCR  
- Ausgabe: Vorschau, Auswahl, Kopieren in Zielordner  
- **Keine** externe KI, **keine** Cloud-Datenbank, **keine** Bankanbindung in v1  
- Bankportale später über ein austauschbares `BankConnector`-Protokoll vorbereitet

### Architekturstil

**Modulare Clean Architecture** mit klaren Grenzen:

```
┌─────────────────────────────────────────────────────────────┐
│  Presentation (SwiftUI)                                     │
│  Views · ViewModels · Navigation · Preview Pane             │
└────────────────────────────┬────────────────────────────────┘
                             │ Commands / State
┌────────────────────────────▼────────────────────────────────┐
│  Application (Use Cases / Services)                         │
│  SearchCoordinator · ImportService · ExportService          │
│  TaxClassificationService · InvoiceLinkExtractor            │
└───────┬─────────────────┬──────────────────┬────────────────┘
        │                 │                  │
┌───────▼───────┐ ┌───────▼───────┐ ┌────────▼────────┐
│ Domain        │ │ Persistence   │ │ Connectors      │
│ Models        │ │ GRDB/SQLite   │ │ IMAP / File /   │
│ TaxYear       │ │ Migrations    │ │ (Bank v2 stub)  │
│ TaxCategory   │ │ FTS5 Index    │ │ Keychain        │
│ Document      │ │ File Cache    │ │ Vision OCR      │
└───────────────┘ └───────────────┘ └─────────────────┘
```

### Kernmodule

| Modul | Verantwortung |
|---|---|
| `KITSApp` | App-Entry, Dependency Injection, Fenster/Scenes |
| `UI` | Deutsche SwiftUI-Oberfläche, Suche, Filter, Vorschau |
| `Domain` | Reine Modelle und Regeln (Steuerbereiche, Jahre, Relevanz) |
| `Persistence` | SQLite-Schema, Migrationen, Repositories |
| `MailIMAP` | WEB.DE-IMAP-Client, Anhang-/Link-Extraktion |
| `FileSources` | Lokale Ordner + iCloud-Drive-Ordner (Security-Scoped Bookmarks) |
| `DocumentPipeline` | PDF-Erkennung, Metadaten, OCR (Vision), Klassifikation |
| `Preview` | PDFKit + Quick Look |
| `Security` | Keychain, Berechtigungen, Secrets |
| `Banking` (Stub) | Protokoll `BankConnector` – in v1 ohne Implementierung |

### Steuerbereiche (Domain)

Feste Kategorien in v1:

1. **Arbeitnehmer**  
2. **Selbst genutztes Eigenheim**  
3. **Vermietete Ferienimmobilie**  
4. **Beteiligung Grundstücksgemeinschaft (Mehrfamilienhaus)**  

Zusätzliche Dokumenttypen (quer zu den Bereichen):

- Kontoauszüge  
- Saldenbestätigungen  
- Darlehensunterlagen  
- Zinsbescheinigungen  

### Datenfluss (vereinfacht)

1. Nutzer konfiguriert Quellen (IMAP, Ordner, Zielordner).  
2. `SearchCoordinator` startet parallele Scans (IMAP + Dateisystem).  
3. Treffer werden normalisiert (`DocumentCandidate`) und persistiert.  
4. Klassifikation anhand Betreff/Dateiname/Pfad/OCR/Text → Steuerjahr + Bereich.  
5. UI filtert nach Jahr/Bereich/Typ; Vorschau über PDFKit/QL.  
6. Nutzer wählt Dateien → `ExportService` kopiert in Zielordner (strukturiert nach Jahr/Bereich).

### Erweiterbarkeit Onlinebanking (v2+)

```swift
protocol BankConnector: Sendable {
    var id: String { get }
    var displayName: String { get }
    func authenticate() async throws
    func fetchStatementCandidates(taxYear: Int) async throws -> [DocumentCandidate]
}
```

In v1 existiert nur das Protokoll + leerer Registry-Eintrag – **keine** konkrete Bankanbindung.

### Nicht-Ziele von Version 1

- Keine automatische Steuererklärung  
- Kein Upload in Cloud-Dienste  
- Keine externe LLM-/KI-API  
- Kein SMTP-Versand  
- Keine direkte Bank-API / Scraping  

---

## 3. Verzeichnisstruktur

Geplante Xcode-/Repo-Struktur unter `KITS/` (noch nicht angelegt außer dieser Datei):

```
KITS/
├── ARCHITECTURE.md                 ← diese Datei
├── README.md                       ← später: Kurzüberblick + Setup
├── Package.swift                   ← optional: Shared Packages / CLI-Tools
├── KITS.xcodeproj/                 ← Xcode-Projekt (macOS App)
├── KITS/
│   ├── App/
│   │   ├── KITSApp.swift
│   │   ├── AppEnvironment.swift
│   │   └── RootScene.swift
│   ├── Resources/
│   │   ├── Assets.xcassets
│   │   ├── Localizable.xcstrings   ← Deutsch als Basissprache
│   │   └── Info.plist
│   ├── UI/
│   │   ├── Navigation/
│   │   ├── Search/
│   │   ├── Results/
│   │   ├── Preview/
│   │   ├── Settings/
│   │   └── Components/
│   ├── Domain/
│   │   ├── Models/
│   │   ├── TaxCategory.swift
│   │   ├── TaxYear.swift
│   │   ├── DocumentKind.swift
│   │   └── Rules/
│   ├── Application/
│   │   ├── SearchCoordinator.swift
│   │   ├── ImportService.swift
│   │   ├── ExportService.swift
│   │   ├── ClassificationService.swift
│   │   └── InvoiceLinkExtractor.swift
│   ├── Persistence/
│   │   ├── Database.swift
│   │   ├── Migrations/
│   │   ├── Repositories/
│   │   └── Records/
│   ├── Connectors/
│   │   ├── Mail/
│   │   │   ├── IMAPClient.swift
│   │   │   ├── WebDEConfiguration.swift
│   │   │   └── MIMEParser.swift
│   │   ├── Files/
│   │   │   ├── LocalFolderScanner.swift
│   │   │   └── ICloudFolderScanner.swift
│   │   └── Banking/
│   │       └── BankConnector.swift   ← Stub für v2
│   ├── Pipeline/
│   │   ├── PDFInspector.swift
│   │   ├── OCRService.swift          ← Vision
│   │   └── MetadataExtractor.swift
│   ├── Preview/
│   │   ├── PDFPreviewView.swift
│   │   └── QuickLookPreview.swift
│   ├── Security/
│   │   ├── KeychainStore.swift
│   │   └── BookmarkStore.swift
│   └── Support/
│       ├── Logging.swift
│       └── Date+TaxYear.swift
├── KITSTests/
│   ├── DomainTests/
│   ├── PersistenceTests/
│   ├── ClassificationTests/
│   ├── InvoiceLinkTests/
│   └── Fixtures/
├── KITSUITests/
└── Config/
    ├── KITS.entitlements
    └── ExportOptions.plist           ← später Notarisierung
```

**Hinweis:** Bestehende Anwendungen außerhalb von `KITS/` bleiben unberührt.

---

## 4. Datenbankschema (SQLite)

### Technologie

- **SQLite** lokal unter Application Support  
- Zugriff über **GRDB.swift** (Migrationen, Beobachtung, FTS5)  
- Pfad-Beispiel:  
  `~/Library/Application Support/de.kits.taxsearch/kits.sqlite`

### Tabellen (v1)

#### `sources`

Konfigurierte Datenquellen.

| Spalte | Typ | Beschreibung |
|---|---|---|
| `id` | TEXT PK | UUID |
| `kind` | TEXT | `imap_webde` \| `local_folder` \| `icloud_folder` |
| `display_name` | TEXT | Anzeigename |
| `config_json` | TEXT | Art-spezifische Konfiguration (ohne Passwort) |
| `bookmark_id` | TEXT NULL | Verweis auf Security-Scoped Bookmark |
| `is_enabled` | INTEGER | 0/1 |
| `last_scan_at` | TEXT NULL | ISO-8601 |
| `created_at` | TEXT | ISO-8601 |
| `updated_at` | TEXT | ISO-8601 |

#### `documents`

Indexierte Dokumentkandidaten (Metadaten, keine vollständigen Dateiinhalte außer Cache-Pfad).

| Spalte | Typ | Beschreibung |
|---|---|---|
| `id` | TEXT PK | UUID |
| `source_id` | TEXT FK → sources.id | Herkunft |
| `external_id` | TEXT | z. B. IMAP UID / Dateipfad-Hash |
| `origin_kind` | TEXT | `email_attachment` \| `email_link` \| `filesystem` |
| `title` | TEXT | Betreff / Dateiname |
| `original_filename` | TEXT NULL | |
| `mime_type` | TEXT NULL | |
| `byte_size` | INTEGER NULL | |
| `content_hash` | TEXT NULL | SHA-256 (Deduplizierung) |
| `tax_year` | INTEGER NULL | erkanntes Steuerjahr |
| `tax_category` | TEXT NULL | Enum-String der 4 Bereiche oder `unbekannt` |
| `document_kind` | TEXT NULL | `rechnung`, `kontoauszug`, `saldenbestaetigung`, `darlehen`, `zinsbescheinigung`, `sonstiges` |
| `relevance_score` | REAL | 0.0–1.0 |
| `source_path` | TEXT NULL | lokaler Pfad, falls vorhanden |
| `cache_path` | TEXT NULL | lokal gecachter Anhang |
| `mail_message_id` | TEXT NULL | |
| `mail_folder` | TEXT NULL | IMAP-Mailbox |
| `mail_received_at` | TEXT NULL | |
| `invoice_url` | TEXT NULL | erkannter Rechnungslink |
| `ocr_status` | TEXT | `none` \| `pending` \| `done` \| `failed` |
| `notes` | TEXT NULL | |
| `created_at` | TEXT | |
| `updated_at` | TEXT | |

**Unique:** `(source_id, external_id)`

#### `document_tags`

Optionale Mehrfach-Tags (z. B. mehrere Bereiche).

| Spalte | Typ |
|---|---|
| `document_id` | TEXT FK |
| `tag` | TEXT |
| PK | `(document_id, tag)` |

#### `scan_runs`

Protokoll jedes Scan-Laufs.

| Spalte | Typ | Beschreibung |
|---|---|---|
| `id` | TEXT PK | |
| `source_id` | TEXT NULL FK | |
| `started_at` | TEXT | |
| `finished_at` | TEXT NULL | |
| `status` | TEXT | `running` \| `success` \| `failed` \| `cancelled` |
| `items_found` | INTEGER | |
| `items_new` | INTEGER | |
| `error_summary` | TEXT NULL | |

#### `export_jobs` / `export_items`

Nachvollziehbarkeit von Kopiervorgängen.

| `export_jobs` | Typ |
|---|---|
| `id` | TEXT PK |
| `destination_root` | TEXT |
| `created_at` | TEXT |
| `status` | TEXT |

| `export_items` | Typ |
|---|---|
| `id` | TEXT PK |
| `job_id` | TEXT FK |
| `document_id` | TEXT FK |
| `destination_path` | TEXT |
| `status` | TEXT |
| `error_message` | TEXT NULL |

#### `settings`

Key/Value für App-Einstellungen (kein Secret).

| Spalte | Typ |
|---|---|
| `key` | TEXT PK |
| `value` | TEXT |

#### Full-Text-Suche

FTS5 virtuelle Tabelle `documents_fts` über `title`, `original_filename`, `ocr_text` (ocr_text ggf. in eigener Spalte oder Side-Table `document_text`).

Empfohlene Side-Table:

#### `document_text`

| Spalte | Typ |
|---|---|
| `document_id` | TEXT PK FK |
| `plain_text` | TEXT | extrahierter PDF-/OCR-Text (lokal) |
| `updated_at` | TEXT |

### Enum-Werte (verbindlich in Code + DB)

```text
tax_category:
  arbeitnehmer
  eigenheim_selbstgenutzt
  ferienimmobilie_vermietet
  grundstuecksgemeinschaft_mfh
  unbekannt

document_kind:
  rechnung
  kontoauszug
  saldenbestaetigung
  darlehen
  zinsbescheinigung
  lohnsteuer
  nebenkosten
  sonstiges
```

### Was **nicht** in SQLite landet

- WEB.DE-Passwort → **Keychain**  
- Security-Scoped Bookmark-Daten → Keychain oder verschlüsselte Bookmark-Datei in Application Support  
- Rohe E-Mail-Bodies dauerhaft (nur abgeleitete Metadaten + ggf. gecachte Anhänge)

---

## 5. IMAP-Bibliotheken (verfügbare Swift-Optionen)

Stand der Recherche: November 2026.

### Empfehlung für KITS v1: **SwiftMail** (Cocoanetics)

| Kriterium | Bewertung |
|---|---|
| Paket | [Cocoanetics/SwiftMail](https://github.com/Cocoanetics/SwiftMail) |
| SPM | ja |
| Plattform | macOS 11+ |
| Swift | 5.9+ |
| API | async/await, Actor-basiert (`IMAPServer`) |
| Basis | SwiftNIO + apple/swift-nio-imap |
| Lizenz | BSD-2-Clause |
| Aktivität | aktiv (Releases bis 2026, z. B. 1.5.x) |
| Eignung WEB.DE | TLS auf Port 993 geeignet |

**Warum empfohlen:** praxistaugliche High-Level-API, moderne Concurrency, aktive Pflege – besser geeignet als Roh-NIO oder veraltetes MailCore2.

### Alternativen

| Bibliothek | Pros | Contras | Empfehlung |
|---|---|---|---|
| **MailFoundation** (migueldeicaza) | async IMAP, viele Auth-Methoden, TLS | Swift 6.2+ Pflicht, jüngeres Ökosystem | starke Alternative, falls Swift 6.2 ohnehin Ziel |
| **apple/swift-nio-imap** | offiziell, typstark | Low-Level; README warnt vor Production-Readiness; Swift 6.2 / macOS 15 | eher Baustein (SwiftMail nutzt ihn) |
| **MailCore2** | historisch bekannt | veraltet, SPM-Probleme, wenig Pflege | **nicht empfohlen** |

### WEB.DE IMAP-Parameter (offiziell)

Voraussetzung: POP3/IMAP in den WEB.DE-Einstellungen **aktivieren**.

| Parameter | Wert |
|---|---|
| Server | `imap.web.de` |
| Port | `993` |
| Verschlüsselung | SSL/TLS |
| Auth | Benutzername + Passwort (Keychain) |
| Fallback | Port 143 + STARTTLS (nur falls nötig) |

SMTP wird in v1 **nicht** benötigt.

### Architektur-Hinweis IMAP

```text
MailIMAP Connector
  → SwiftMail IMAPServer
  → list/select mailboxes
  → search by date / tax-year window
  → fetch headers + structure
  → download PDF attachments
  → parse HTML/text for invoice URLs
  → map to DocumentCandidate
```

Rechnungslinks: Regex + URL-Heuristik (z. B. `rechnung`, `invoice`, `beleg`, typische Provider-Hosts) – deterministic, keine Cloud-KI.

---

## 6. Notwendige macOS-Berechtigungen

### App Sandbox (empfohlen für Distribution)

Datei: `Config/KITS.entitlements`

| Entitlement | Zweck |
|---|---|
| `com.apple.security.app-sandbox` | Sandbox aktiv |
| `com.apple.security.network.client` | IMAP zu `imap.web.de` |
| `com.apple.security.files.user-selected.read-write` | Nutzerwählt Quell-/Zielordner |
| `com.apple.security.files.bookmarks.app-scope` | dauerhafter Ordnerzugriff via Bookmarks |

Optional je nach Packaging:

- `com.apple.security.files.downloads.read-only` – falls Downloads-Standardordner gescannt werden soll  

### Info.plist / Privacy-Strings (Deutsch)

Auch bei Sandbox sinnvoll, sobald geschützte Bereiche berührt werden:

| Key | Beispieltext |
|---|---|
| `NSDocumentsFolderUsageDescription` | „KITS durchsucht ausgewählte Ordner nach steuerrelevanten Dokumenten.“ |
| `NSDownloadsFolderUsageDescription` | „KITS kann den Downloads-Ordner nach Belegen durchsuchen, wenn Sie ihn freigeben.“ |
| `NSRemovableVolumesUsageDescription` | nur falls externe Volumes unterstützt werden |

### iCloud Drive

- Zugriff über vom Nutzer freigegebene Ordner (NSOpenPanel) + Security-Scoped Bookmarks  
- **Kein** CloudKit, **kein** iCloud-Container für App-Daten nötig in v1  
- Hardlink/`~/Library/Mobile Documents/` nicht stillschweigend scannen – immer explizite Freigabe

### Keychain

- Speicherung WEB.DE-Zugangsdaten über Security-Framework (`kSecClassGenericPassword`)  
- Service-Name z. B. `de.kits.taxsearch.webde`  
- Kein Entitlement speziell nötig für eigene Keychain-Items der App  

### Vision / OCR

- Apple Vision läuft on-device  
- Keine extra Privacy-Prompt-Keys typischerweise für lokale Bild-/PDF-OCR in einer Mac-App  
- PDF-Seiten ggf. als Image rendern → `VNRecognizeTextRequest` (Deutsch: `.de-DE`)

### Hardened Runtime / Notarisierung (später)

Für Distribution außerhalb des Dev-Builds:

- Hardened Runtime  
- Notarisierung bei Apple  
- ggf. Ausnahme nur falls Drittanbieter-Libs Unsigned Memory brauchen (SwiftMail/NIO i. d. R. unproblematisch)

### Netzwerk

- Nur ausgehende Verbindungen zu konfiguriertem IMAP-Host  
- Keine Analytics-/KI-Endpoints in v1  

---

## 7. Entwicklungsplan (kleine, testbare Schritte)

Jeder Schritt endet mit einem **prüfbaren Ergebnis** (Unit-Test und/oder manuelle Checkliste). Implementierung startet erst nach Freigabe.

### Phase 0 – Werkbank (Mac)

1. Xcode 16 + Swift prüfen  
2. Leeres macOS SwiftUI-Projekt `KITS` anlegen (Bundle-ID z. B. `de.kits.taxsearch`)  
3. Entitlements + deutsche Basis-UI-Strings setzen  
4. **Test:** App startet, leeres Hauptfenster „KITS – KI Tax Search“

### Phase 1 – Domain & Persistenz

5. Domain-Enums: Steuerbereiche, Dokumentarten, Steuerjahr-Helfer  
6. GRDB einbinden, Migration v1 anlegen  
7. Repositories für `sources` / `documents`  
8. **Test:** In-Memory-DB speichert und liest Dokument mit Kategorie `arbeitnehmer`

### Phase 2 – Lokale Dateiquellen

9. Ordnerauswahl via NSOpenPanel + BookmarkStore  
10. Rekursive PDF-Erkennung (Filter `.pdf`)  
11. Metadaten-Index (Name, Größe, Hash, mtime)  
12. **Test:** Fixture-Ordner mit 3 PDFs erscheint in Ergebnisliste

### Phase 3 – Klassifikation (regelbasiert)

13. Heuristiken Dateiname/Pfad/Betreff → Steuerjahr  
14. Keyword-Maps für die 4 Steuerbereiche + Bankenbelege  
15. Relevance-Score  
16. **Test:** „Zinsbescheinigung_2024.pdf“ → Jahr 2024, `zinsbescheinigung`

### Phase 4 – Vorschau & Export

17. PDFKit-Vorschau in der UI  
18. Quick Look Fallback  
19. Mehrfachauswahl → Kopieren nach `Ziel/Jahr/Kategorie/`  
20. **Test:** Export erzeugt erwartete Ordnerstruktur, `export_jobs` protokolliert

### Phase 5 – IMAP WEB.DE

21. SwiftMail SPM-Dependency  
22. Keychain-Login-Flow (ohne Passwort in Logs/DB)  
23. Mailbox-Liste + Datums-/Jahressuche  
24. PDF-Anhänge laden und cachen  
25. Rechnungslink-Extraktor  
26. **Test:** Integrationstest gegen Testpostfach / Fixtures (MIME-Samples)

### Phase 6 – OCR

27. Vision-OCR für gescannte PDFs ohne Text-Layer  
28. Speicherung in `document_text` + FTS  
29. **Test:** Fixture-Scan-PDF liefert erwartete Keywords

### Phase 7 – Such-UX

30. Filter: Steuerjahr, Bereich, Dokumentart, Quelle  
31. Volltextsuche (FTS5)  
32. Scan-Fortschritt / Abbrechen  
33. **Test:** UI-Test Filter + leere Zustände auf Deutsch

### Phase 8 – Härtung

34. Deduplizierung per `content_hash`  
35. Fehlerbehandlung Offline/IMAP  
36. Logging ohne Secrets  
37. Banking-Stub (`BankConnector`) dokumentiert einbinden  
38. **Test:** Regression der Phasen 1–7

### Reihenfolge der Abhängigkeiten

```text
Phase 0 → 1 → 2 → 3 → 4
                ↘ 5 → 6 → 7 → 8
```

Phasen 2 und 5 können nach Phase 1 teilweise parallelisiert werden; OCR (6) hängt von indexierten Dokumenten ab.

---

## 8. Technologie-Stack (Ziel)

| Bereich | Wahl |
|---|---|
| UI | SwiftUI (Deutsch) |
| Persistenz | SQLite + GRDB.swift |
| Mail | SwiftMail (Fallback-Option: MailFoundation) |
| PDF | PDFKit |
| Vorschau | PDFKit + Quick Look |
| OCR | Vision (`VNRecognizeTextRequest`) |
| Secrets | Keychain (Security.framework) |
| Ordnerzugriff | Security-Scoped Bookmarks |
| Concurrency | Swift async/await, Actors |
| Tests | Swift Testing / XCTest |
| KI/Cloud | **keine** in v1 |

---

## 9. Offene Entscheidungen (Freigabe nötig)

Vor Implementierungsstart bitte bestätigen oder anpassen:

1. **Deployment Target:** macOS 14 vs. 15?  
2. **IMAP-Library:** SwiftMail als Standard – ok?  
3. **Bundle-ID:** `de.kits.taxsearch` – ok?  
4. **Sandbox von Anfang an** – ja/nein?  
5. **Mindest-Steuerjahre in UI** (z. B. laufendes Jahr ± 10)?  
6. Soll der Cloud-Agent nur Architektur liefern (dieser Stand), und die Xcode-Implementierung lokal auf dem Mac erfolgen?

---

## 10. Nächster Schritt

Nach Ihrer Freigabe dieser Architektur:

- Xcode-Projektgerüst unter `KITS/` anlegen  
- Phase 0–1 umsetzen  
- **Keine** Änderungen an Anwendungen außerhalb von `KITS/`

---

*Ende der Architekturfassung v1 – wartend auf Freigabe.*
