# Codex Review: ChatGPT Conversation Extractor

Stand: 2026-06-22. Rolle: kritischer Software Engineer, Zielgruppe: Junior Developer.

## 1. Kurzfazit

Dieses Repository ist ein Python-CLI-Tool, das einen offiziellen ChatGPT-Datenexport liest und daraus pro Unterhaltung lesbare Markdown-Dateien, strukturierte JSON-Dateien oder beides erzeugt. Der Kern ist ein defensiver Parser für die `mapping`-Graphstruktur von ChatGPT-Konversationen. Er rekonstruiert den aktiven Pfad über `current_node`, filtert interne System-/Tool-Nachrichten, extrahiert Inhalte, Dateien, Web-URLs, Zitate, Custom-GPT-/Modell-Metadaten und schreibt anschließend Dateien mit Frontmatter und optional synchronisierten Dateizeitstempeln.

Der wichtigste aktuelle Befund: Das Repo ist bereits teilweise auf neuere Export-Änderungen vorbereitet, aber der tatsächliche CLI-Einstieg verarbeitet weiterhin nur **eine** Eingabedatei. Nach aktuellem offiziellen OpenAI-Hilfetext können größere Exporte jedoch `conversations.json` **oder nummerierte Conversation-JSON-Dateien** enthalten. Für den heutigen Standard sollte das Tool deshalb zusätzlich eine ZIP-Datei, ein Verzeichnis oder mehrere `conversations-*.json`/nummerierte JSON-Dateien als Input akzeptieren.

## 2. Setup und Betrieb

### 2.1 Laufzeit und Installation

- Package-Layout: `src/chatgpt_extractor`.
- Python-Version laut `setup.py`: `>=3.9`.
- Runtime-Dependency: `PyYAML>=6.0.1`.
- Dev-Dependencies: `pytest`, `pytest-cov`, `black`, `flake8`, `mypy`.
- CLI-Entry-Point nach Installation: `chatgpt-extractor=chatgpt_extractor.__main__:main`.
- Alternativ kann direkt `python -m chatgpt_extractor ...` oder der Wrapper `python extract.py ...` genutzt werden.

### 2.2 Minimaler Quickstart

```bash
python -m venv .venv
source .venv/bin/activate
pip install -e .
python -m chatgpt_extractor data/raw/conversations.json data/output
```

Standardwerte im CLI:

```bash
python -m chatgpt_extractor
```

entspricht:

```bash
python -m chatgpt_extractor data/raw/conversations.json data/output --output-format markdown
```

### 2.3 Wichtige CLI-Optionen

- `--output-format markdown|json|both`: Ausgabeformat.
- `--json-format multiple|single`: einzelne JSON-Dateien pro Chat oder eine konsolidierte Datei.
- `--markdown-dir`, `--json-dir`, `--json-file`: Zielpfade überschreiben.
- `--preserve-timestamps true|false`: Datei-Modification-Times aus Chat-Metadaten setzen.
- `--per-turn-timestamps` / `--no-per-turn-timestamps`: Zeitstempel pro Nachricht ausgeben.
- `--gpt-metadata` / `--no-gpt-metadata`: Custom-GPT-, Modell- und Plugin-Signale ausgeben.
- `--gpt-names-xlsx PATH`: Sidecar-Datei zur Auflösung von `gizmo_id` zu lesbaren GPT-Namen.
- `--config PATH`: YAML-Konfigurationsdatei.
- `--analyze-failures`: optionaler Diagnosepfad nach Fehlern.
- `--debug`: Debug-Logging.

## 3. Repository-Struktur

```text
.
├── README.md
├── setup.py
├── extract.py
├── requirements.txt
├── requirements-dev.txt
├── src/chatgpt_extractor/
│   ├── __main__.py
│   ├── extractor.py
│   ├── processors.py
│   ├── gpt_metadata.py
│   ├── config.py
│   ├── trackers.py
│   ├── logging_config.py
│   └── __init__.py
├── tests/
│   ├── fixtures/sample_conversation.json
│   └── test_*.py
└── docs/
    ├── ARCHITECTURE.md
    ├── SCHEMA_NOTES.md
    ├── TECHNICAL_REFERENCE.md
    ├── USER_GUIDE.md
    └── weitere Betriebs-/Release-Dokumente
```

### 3.1 Top-Level-Dateien

- `README.md`: Produktbeschreibung, Features, Quickstart, Output-Struktur und bekannte Grenzen.
- `setup.py`: Packaging, Dependencies und CLI-Entry-Point. Achtung: Version in `setup.py` ist `2.0.0`, CLI meldet aber `v3.1`; das ist inkonsistent.
- `extract.py`: dünner Wrapper, fügt `src` in `sys.path` ein und ruft den CLI-Main auf.
- `requirements.txt` / `requirements-dev.txt`: Dependencies für Runtime und Entwicklung.

### 3.2 Quellcode-Module

#### `__main__.py`

Verantwortlich für:

1. CLI-Argumente definieren.
2. Argumentkombinationen validieren.
3. Logging konfigurieren.
4. Existenz der Eingabedatei prüfen.
5. `ConversationExtractorV2` instanziieren.
6. `extract_all()` starten.

Kritisch: Die CLI erwartet aktuell genau einen Pfad zu einer JSON-Datei. Ein Verzeichnis, ZIP oder mehrere nummerierte Export-Dateien werden nicht als Eingabe unterstützt.

#### `extractor.py`

Das Hauptmodul. Wichtigste Klasse: `ConversationExtractorV2`.

Hauptaufgaben:

- Output-Pfade bestimmen.
- Output-Verzeichnisse anlegen.
- JSON laden.
- Jede Konversation verarbeiten.
- Aktiven Konversationspfad rekonstruieren.
- Metadaten extrahieren.
- Nachrichten filtern, normalisieren und zusammenführen.
- Markdown und/oder JSON schreiben.
- Schema-Report und Fehlerlog schreiben.
- Dateizeitstempel synchronisieren.

Wichtige Methoden:

- `extract_all()`: lädt die Eingabedatei, iteriert über alle Gespräche, schreibt Ausgaben.
- `process_conversation()`: verarbeitet eine einzelne Conversation.
- `extract_metadata()`: sammelt Conversation-Level-Metadaten wie Titel, IDs, Zeiten, Modell, Project-ID, GPT-Metadaten.
- `backward_traverse()`: läuft vom `current_node` über `parent`-Links zurück zum Root und dreht danach die Reihenfolge wieder chronologisch.
- `process_messages()`: filtert und extrahiert Nachrichteninhalte.
- `merge_continuations()`: kombiniert direkt aufeinanderfolgende Assistant-Nachrichten.
- `generate_markdown()`: erzeugt Markdown mit YAML-Frontmatter.
- `generate_json_data()`: erzeugt das eigene normalisierte JSON-Format.
- `save_markdown_file()`, `save_json_multiple()`, `save_json_single()`: schreiben Dateien.
- `save_schema_report()`, `save_conversion_log()`: Diagnoseartefakte.

#### `processors.py`

Kapselt Content-Extraktion und Filterlogik.

Wichtige Aufgaben:

- Interne Systemnachrichten filtern.
- Tool-Nachrichten normalerweise filtern, aber sichtbare DALL-E- und Deep-Research-Artefakte erhalten.
- Content-Typen behandeln: `text`, `code`, `execution_output`, `multimodal_text`, `user_editable_context`, `tether_quote`, `sonic_webpage`, `system_error` usw.
- Multimodal-Parts wie Bilder, Audio, Video und Code-Interpreter-Output in Textmarker übersetzen.
- Zitate, Web-URLs und Dateinamen extrahieren.

#### `gpt_metadata.py`

Extrahiert moderne Zusatzsignale:

- Conversation-Level: `gizmo_id`, `gizmo_type`, `models_used`, Deep-Research-Erkennung.
- Message-Level: `metadata.model_slug`, `metadata.gizmo_id`, `metadata.invoked_plugin.namespace`.
- GPT-Namensauflösung aus optionaler `GPT_Names.xlsx`.
- Deep-Research-Report aus `metadata.chatgpt_sdk.widget_state.report_message`.

Kritischer Punkt: In den Repo-Docs ist bereits dokumentiert, dass neue Bulk-Exports seit 2026 bei Custom-GPT-Konversationen `gizmo_id` auf Conversation-Level verlieren können und dass `conversation_template_id` als Fallback genutzt werden sollte. Der aktuelle Code nutzt in `extract_conv_gpt_meta()` aber nur `conv.get("gizmo_id")` und nicht den Fallback. Das sollte geändert werden.

#### `config.py`

Lädt optionale YAML-Konfiguration in folgender Reihenfolge:

1. Built-in Defaults.
2. Expliziter `--config`/Constructor-Pfad.
3. `$CHATGPT_EXTRACTOR_CONFIG`.
4. `./chatgpt_extractor.yaml`.
5. `~/.config/chatgpt_extractor/config.yaml`.

Unterstützte Keys:

- `per_turn_timestamps`.
- `gpt_metadata`.
- `gpt_names_xlsx`.

Defensiv: Fehlerhafte YAML-Dateien blockieren den Lauf nicht.

#### `trackers.py`

Enthält:

- `SchemaEvolutionTracker`: sammelt unbekannte Content-Typen, Rollen, Recipients, Metadata Keys, Part Types und Finish Types.
- `ProgressTracker`: Fortschrittsanzeige und Abschlussstatistiken.

#### `logging_config.py`

Konfiguriert Production-Logging und Exception-Logging.

## 4. Datenfluss im Detail

### 4.1 Happy Path

1. CLI prüft Eingabedatei.
2. `ConversationExtractorV2` wird mit Output-Format und Pfaden gebaut.
3. `extract_all()` liest die Eingabe per `json.load()`.
4. Für jedes Conversation-Dict:
   1. `extract_metadata()` baut Frontmatter-/Envelope-Metadaten.
   2. `backward_traverse(mapping, current_node, conv_id)` rekonstruiert nur den aktiven Chat-Pfad.
   3. `collect_message_statistics()` sammelt Content-Typen, Code-Count und Custom Instructions.
   4. `process_messages()` filtert und extrahiert sichtbare Nachrichten.
   5. `merge_continuations()` verbindet zusammenhängende Assistant-Antwortsegmente.
   6. Markdown und/oder JSON wird gerendert.
   7. Dateien werden in `md/`, `json/` und optional Projektunterordner geschrieben.
   8. Optional werden Dateizeitstempel gesetzt.
5. Am Ende schreibt der Extractor `schema_evolution.log` und bei Fehlern `conversion_log.log` plus `conversion_failures.json`.

### 4.2 Warum Rückwärts-Traversal?

ChatGPT-Konversationen sind kein einfacher Array-Transcript, sondern ein Graph. Edits und Regenerierungen erzeugen Branches. Der aktive Stand hängt am `current_node`. Durch Rückwärtslaufen über `parent` wird der aktive Pfad rekonstruiert und verworfene Branches werden ignoriert. Das ist effizienter und weniger fehleranfällig als ein naiver Forward-Walk.

### 4.3 Output-Design

Markdown:

- YAML-Frontmatter mit IDs, Zeiten, Modell, Projekt, GPT-Metadaten, Statistikwerten.
- `# Titel`.
- Pro Nachricht `## User`, `## Assistant`, optional `## System`.
- Optional per-turn ISO-Zeitstempel.
- Dateien als `[File: name]`.
- Zitate und Web-URLs als Listen.

JSON:

- Eigenes normalisiertes Format, nicht identisch mit OpenAI-Rohdaten.
- Conversation-Envelope plus `messages`-Array.
- Optional Zusatzfelder für GPT-/Modell-/Plugin-Metadaten.

## 5. Tests und Qualität

Das Repo hat eine breite Testsuite unter `tests/`:

- CLI-Validierung.
- Core-Extractor-Verhalten.
- JSON-Output.
- Message-Processor.
- GPT-Metadaten.
- Tracker.
- Timestamp-Verhalten.
- Coverage-bezogene Tests.
- Performance-/Sample-Data-Helfer.

Als Junior solltest du beim Ändern dieses Repos mindestens ausführen:

```bash
python -m pytest
```

Optional, falls Dev-Dependencies installiert sind:

```bash
python -m pytest --cov=chatgpt_extractor
black --check src tests
flake8 src tests
mypy src
```

## 6. Aktueller Stand des offiziellen ChatGPT-Exports am 2026-06-22

### 6.1 Was OpenAI offiziell sagt

OpenAI beschreibt aktuell weiterhin den Export über ChatGPT Settings → Data controls → Export. Die Hilfe sagt, dass Exporte über ChatGPT Settings für Free, Plus, Pro und eligible ChatGPT Edu verfügbar sind, aber nicht für Business oder Enterprise Workspaces über diese Settings. Quelle: OpenAI Help Center, aktualisiert ca. 2026-06-18: https://help.openai.com/en/articles/7260999-how-do-i-export-my-chatgpt-history-and-data

OpenAI sagt außerdem, dass der Download-Link 24 Stunden nach Erhalt abläuft und dass die Erstellung bis zu 7 Tage dauern kann. Gleiche Quelle: https://help.openai.com/en/articles/7260999-how-do-i-export-my-chatgpt-history-and-data

Für ChatGPT Edu nennt OpenAI explizit, dass die ZIP-Datei `conversations.json` **oder nummerierte Conversation-JSON-Dateien für größere Exporte** enthalten kann, außerdem Dateien/Assets und Account-/Conversation-Metadaten. Quelle: https://help.openai.com/nb-no/articles/20001279-exporting-data-from-a-chatgpt-edu-workspace

OpenAI bestätigt im Account-Transfer-Artikel ebenfalls: Größere Exporte können statt `conversations.json` nummerierte Conversation-JSON-Dateien enthalten. Quelle: https://help.openai.com/en/articles/9106926-transfer-exported-conversations-between-chatgpt-accounts

### 6.2 Ist das Exportformat im Repo noch aktuell?

Teilweise.

Was noch aktuell ist:

- Das Kernmodell `conversations.json` als Top-Level-Liste von Conversation-Objekten mit `mapping` und `current_node` ist weiterhin relevant.
- Das Repo kennt bereits viele moderne Content-Typen und Metadatenmuster.
- Das Repo hat bereits Wissen über Projects (`g-p-*`), Custom GPTs, per-turn `model_slug`, Plugins, Deep Research, multimodale Parts und Custom Instructions.
- Die Docs im Repo kennen bereits die 2026-06-Änderung zu nummerierten Dateien und fehlendem Conversation-Level-`gizmo_id`.

Was nicht vollständig aktuell ist:

1. **Input-Formate**: Der CLI-Pfad erwartet eine einzelne Datei. Nummerierte Export-Dateien oder eine ZIP-Datei werden nicht automatisch erkannt.
2. **Custom-GPT-Fallback**: Die Repo-Dokumentation empfiehlt `conv.get("gizmo_id") or conversation_template_id`, wenn `gizmo_type == "gpt"`. Der aktuelle Code macht diesen Fallback im GPT-Metadata-Modul noch nicht.
3. **README-Quickstart**: Er spricht primär von `conversations.json`, nicht prominent von nummerierten Dateien.
4. **Versionierung**: `setup.py` sagt `2.0.0`, CLI und README sprechen von `v3.1`; das ist für Nutzer und Releases verwirrend.
5. **Large Export Workflow**: Es gibt keinen klaren Loader, der Shards zusammenführt, Duplikate erkennt, Reihenfolge/Sortierung stabilisiert und Fehler pro Shard meldet.
6. **Assets**: OpenAI erwähnt Dateien und andere Assets im ZIP. Das Tool extrahiert Dateinamen/Pointer, kopiert/verknüpft Assets aber nicht.
7. **Business/Enterprise**: Offiziell nicht über ChatGPT Settings verfügbar. Das Repo sollte in Docs klarer zwischen persönlichem/Edu-Export und Business/Enterprise unterscheiden.

## 7. Was müsste geändert werden, damit es dem heutigen Standard entspricht?

Nur Analyse, keine Codeänderung in diesem Review.

### 7.1 Input-Layer modernisieren

Empfohlene neue Abstraktion: `load_conversation_export(input_path) -> list[dict]`.

Sie sollte akzeptieren:

- Einzeldatei `conversations.json`.
- Einzeldatei `conversations-000.json` oder ähnliche nummerierte Datei.
- Verzeichnis mit `conversations.json` oder mehreren nummerierten JSON-Dateien.
- ZIP-Datei des offiziellen Exports.
- Optional mehrere CLI-Pfade.

Verhalten:

- Alle JSON-Shards laden und zu einer Liste zusammenführen.
- Duplikate über `id`/`conversation_id` erkennen.
- Stabile Sortierung, z. B. nach `update_time` oder Original-Datei + Index.
- Pro Shard aussagekräftige Fehlermeldungen.
- Große Dateien möglichst stream- oder batchfreundlich behandeln.

### 7.2 Custom-GPT-Fallback implementieren

In `gpt_metadata.extract_conv_gpt_meta()` sollte die dokumentierte Fallback-Regel genutzt werden:

```python
gizmo_id = conv.get("gizmo_id") or (
    conv.get("conversation_template_id")
    if conv.get("gizmo_type") == "gpt"
    else None
)
```

Wichtig: `gizmo_type == "gpt"` verhindert, dass Project-IDs (`g-p-*`) fälschlich als Custom-GPT-IDs ausgegeben werden.

### 7.3 CLI und README aktualisieren

- `input_file` in `input_path` umbenennen oder dokumentieren, dass Datei/Verzeichnis/ZIP erlaubt ist.
- Beispiele für große Exporte ergänzen:

```bash
python -m chatgpt_extractor export.zip data/output
python -m chatgpt_extractor path/to/extracted-export/ data/output
python -m chatgpt_extractor path/to/conversations-000.json data/output
```

- Hinweis: Business/Enterprise nicht über Settings-Export.
- Hinweis: Upload in anderes ChatGPT-Konto ist laut OpenAI nur Referenz, keine echte Migration/Rehydration.

### 7.4 Tests ergänzen

Neue Fixtures:

- Verzeichnis mit `conversations-000.json`, `conversations-001.json`.
- ZIP mit nummerierten Dateien.
- Custom-GPT-Conversation ohne `gizmo_id`, aber mit `gizmo_type: "gpt"` und `conversation_template_id`.
- Project-Conversation mit `gizmo_type: "snorlax"`/`conversation_template_id: "g-p-*"`, um falsches GPT-Fallback zu verhindern.
- Duplicate-ID-Fall.
- Shard mit invalidem JSON.

### 7.5 Version und Packaging bereinigen

- `setup.py`-Version, CLI-`--version`, README und Changelog synchronisieren.
- Optional `pyproject.toml` einführen und `setup.py` langfristig ablösen.
- Runtime-Dependencies prüfen: `openpyxl` ist optional, aber `gpt-names-xlsx` wird ohne Installation still übersprungen. Das ist okay, sollte aber klar in Docs stehen.

### 7.6 Zeitstempel korrigieren

In `extract_metadata()` wird `datetime.fromtimestamp(create_time).isoformat() + "Z"` verwendet. Ohne `timezone.utc` ist das lokalzeitabhängig, wird aber mit `Z` als UTC markiert. Die per-turn-Methode nutzt bereits korrekt UTC. Empfehlung:

```python
datetime.fromtimestamp(create_time, tz=timezone.utc).isoformat().replace("+00:00", "Z")
```

### 7.7 YAML-Ausgabe robuster machen

`generate_markdown()` baut YAML manuell. Das kann bei Sonderzeichen, mehrzeiligen Strings, Listen oder Doppelpunkten brüchig werden. Besser:

- `yaml.safe_dump(metadata, allow_unicode=True, sort_keys=False)` für Frontmatter nutzen.
- Tests für Titel/Custom Instructions mit Sonderzeichen ergänzen.

### 7.8 Assets optional verarbeiten

Da offizielle Exporte Assets enthalten können:

- Asset-Verzeichnis im ZIP erkennen.
- Datei-IDs/Asset-Pointer auf lokale Pfade mappen.
- Optional Assets in Output kopieren oder relative Links in Markdown ausgeben.

### 7.9 Schema-Evolution als First-Class Diagnose

Der Tracker ist gut, aber die Ausgabe könnte besser maschinenlesbar sein:

- Zusätzlich `schema_evolution.json` schreiben.
- Unknown Samples mit Conversation-ID, Message-ID, Content-Type, minimalem Kontext.
- Option `--strict-schema` für CI/Regressionstests.

## 8. Kritische Code-Review-Hinweise

### 8.1 Gute Entscheidungen

- Defensiver Umgang mit `None` in Metadaten, Parts und Content.
- Graph-Traversal vom `current_node` ist fachlich richtig.
- Schema-Evolution-Tracking ist wichtig, weil OpenAI das Exportformat nicht explizit versioniert.
- Message-Verarbeitung ist relativ gut separiert vom Extractor.
- Deep-Research-Sonderfall wurde erkannt; das ist ein modernes reales Problem.
- Output kann Markdown und JSON parallel erzeugen.

### 8.2 Risiken und technische Schulden

- `ConversationExtractorV2` ist sehr groß und macht zu viel: Laden, Transformieren, Rendern, Schreiben, Timestamp-Sync, Fehlerlog, Reporting.
- Keine klare Input-Abstraktion für mehrere Exportformen.
- Manuelles YAML-Rendering.
- Versionsinkonsistenz `2.0.0` vs. `3.1`.
- Einige Kommentare sprechen von empirischen lokalen Corpus-Scans; für externe Maintainer ist unklar, wo diese Daten/Belege liegen.
- `run_failure_analysis()` importiert ein Modul `.analyze_failures`, das im aktuellen Dateibaum nicht vorhanden ist. Es ist optional abgefangen, aber als Feature irritierend.
- README sagt Python 3.8+, `setup.py` verlangt >=3.9.
- Single JSON und multiple JSON sind eigene Exportformate des Tools, nicht OpenAI-kompatible Rehydrationsformate. Das sollte deutlicher kommuniziert werden.

## 9. Empfohlene Roadmap

### Phase 1: Kompatibilität mit aktuellem Export

1. Input-Loader für Datei/Verzeichnis/ZIP/Shards.
2. Custom-GPT-Fallback aus `conversation_template_id`.
3. Tests für nummerierte Dateien und fehlendes `gizmo_id`.
4. README/Docs aktualisieren.

### Phase 2: Korrektheit und Stabilität

1. UTC-Zeitstempel in Conversation-Level-Metadaten korrigieren.
2. YAML mit `safe_dump` rendern.
3. Versionierung konsolidieren.
4. Schema-Evolution zusätzlich als JSON ausgeben.

### Phase 3: Architektur

1. `InputLoader`, `ConversationTransformer`, `MarkdownRenderer`, `JsonRenderer`, `OutputWriter` trennen.
2. Größere Integrationstests mit Shards.
3. Optional Asset-Kopier-/Linking-Modus.

## 10. Antwort auf: „Was genau kann man machen, nur analysieren?“

Ja, in diesem Schritt wurde bewusst nur analysiert und dokumentiert. Der Produktcode wurde nicht angepasst.

Konkrete nächste umsetzbare Tickets wären:

1. **Ticket: Multi-file/ZIP Export Loader**
   - Akzeptiert `conversations.json`, `conversations-*.json`, Verzeichnisse und ZIPs.
   - Fügt alle Conversation-Arrays zusammen.
   - Tests mit zwei Shards.

2. **Ticket: Custom GPT Fallback**
   - Nutzt `conversation_template_id` als `gizmo_id`, wenn `gizmo_type == "gpt"` und `gizmo_id` fehlt.
   - Negative Testcases für Projects.

3. **Ticket: Docs aktualisieren**
   - README Quickstart für große Exporte.
   - Hinweis auf offizielle Export-Beschränkungen.

4. **Ticket: Timestamp-Fix**
   - Conversation-Level-Zeitstempel korrekt in UTC rendern.

5. **Ticket: YAML-Frontmatter robust machen**
   - Manuelles Escaping entfernen, `yaml.safe_dump` verwenden.

## 11. Quellen

- OpenAI Help Center: Exporting your ChatGPT history and data, Stand ca. 2026-06-18: https://help.openai.com/en/articles/7260999-how-do-i-export-my-chatgpt-history-and-data
- OpenAI Help Center: Exporting data from a ChatGPT Edu workspace, Stand ca. 2026-06-18: https://help.openai.com/nb-no/articles/20001279-exporting-data-from-a-chatgpt-edu-workspace
- OpenAI Help Center: Transfer exported conversations between ChatGPT accounts, Stand ca. 2026-06-18: https://help.openai.com/en/articles/9106926-transfer-exported-conversations-between-chatgpt-accounts
- Lokale Repo-Dateien: `README.md`, `setup.py`, `extract.py`, `src/chatgpt_extractor/__main__.py`, `src/chatgpt_extractor/extractor.py`, `src/chatgpt_extractor/processors.py`, `src/chatgpt_extractor/gpt_metadata.py`, `src/chatgpt_extractor/config.py`, `src/chatgpt_extractor/trackers.py`, `docs/SCHEMA_NOTES.md`.
