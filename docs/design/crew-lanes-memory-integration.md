# Architektur- und Implementierungsplan: Integrierte Lanes, Agent-Gateway, Crew-Memory und Context-Orchestrierung

**Ziel-Repository:** `eyshoit-commits/bkg-crew-cli`  
**Basis:** aktueller `main`-Stand; bei jeder Implementierung HEAD und Worktree erneut prüfen.  
**Status:** Designvorschlag. Die Gateway-/Lane-Basis ist laut aktuellem Repository-Dokument bereits in `bkgcrewcli` integriert; Memory, projektübergreifende Code-Suche und recoverable Context-Kompression sind die wesentlichen Erweiterungen.  
**Architekturvorgabe:** Laufzeitfunktionen bleiben in `bkgcrewcli`. Kein verpflichtender externer Proxy, Memory-Dienst, Python-Sidecar oder anderer separater Prozess. Optionale Protokolladapter dürfen unabhängig entwickelt werden, sind aber nicht Voraussetzung für den lokalen Kern.

## 1. Ziel und Leitentscheidung

Die Referenzprojekte werden nicht als unbesehenes Paketbündel oder als verpflichtende Runtime-Abhängigkeiten eingebaut. Stattdessen übernehmen wir ihre jeweils stärksten, nicht überlappenden Ideen hinter eigenen Rust-Schnittstellen:

- **Shodh-Memory** ist die Hauptvorlage für lokale, LLM-freie Memory-Lebenszyklen: gestufte Erinnerung, deduplizierte Erfassung, Zugriffssignale, Konsolidierung und nachvollziehbare Pflege.
- **Uteke** ergänzt explizite Rooms mit Autor-/Quellen-Provenienz, Hybrid-Retrieval, zeitbezogene Abfragen und Beziehungen zwischen Erinnerungen.
- **srag** inspiriert den projektübergreifenden Code-Index: AST-bewusste Chunks, inkrementelles Indexieren, Hybrid-Suche, Pattern-Erkennung und Caller/Callee-Kontext. Anders als srag soll der erste Zielpfad ohne Python-ML-Sidecar funktionieren.
- **TinyJuice** liefert das Konzept für inhaltsbewusste, policy-gesteuerte Tool-Output-Reduktion mit expliziter Wiederherstellung des Originals statt stiller Trunkierung.
- **agent-cli-to-api** dient als Vorlage für einen klaren Adaptervertrag zwischen heterogenen Agent-CLIs und OpenAI-/Anthropic-kompatiblen Clientoberflächen. Das vorhandene native Gateway bleibt die Implementierungsbasis.
- **prmt** inspiriert allenfalls die schnelle, kontextabhängige Terminal-Statusanzeige. Es ist kein Memory-, Routing- oder Agenten-Orchestrierungsbaustein.

Zusammen ergibt sich eine Pipeline: **Agenten-Tools und Projektdateien → sichere, abrufbare Tool-Outputs → Code-/Memory-Retrieval → begrenzter Context Builder → Agenten-Task → überprüfbare Ergebnisse und Memory-Vorschläge**. Provider-Routing, Modell-Governor, Telemetrie und Wiederherstellung hängen an derselben Task-/Session-Identität, bleiben aber getrennte Module.

## 2. Bestehende Basis und Grenzen

Das Repository besitzt bereits Rust-Agentenadapter, Prozesssteuerung/Cancellation, Konfiguration, Keystore, normalisierte Provider-Requests und Events sowie einen integrierten Gateway. Die aktuelle Architekturdokumentation beschreibt unter anderem OpenAI-kompatible Chat-/Responses-Routen, Anthropic Messages, scoped Routen, Tokens, Streaming, Usage und Admin-Funktionen. Diese existierenden Verträge sind zuerst am tatsächlichen HEAD zu prüfen und weiterzuverwenden; der Plan setzt nicht voraus, dass alle Adapter dieselben Fähigkeiten haben.

Der Gateway-/Lane-Pfad bleibt der einzige gepoolte Provider-Sendepfad. Ein Agentenadapter gilt nur dann als gepoolt, wenn Tests nachweisen, dass Base-URL, Authentifizierung, Modellzuordnung und Stream tatsächlich über den lokalen Gateway laufen. Bei nicht kompatiblen Agenten muss Diagnose den ungepoolten Modus klar ausweisen; kein stiller Direct-Fallback.

## 3. Zielarchitektur

```text
CLI / Public Rust API / vorhandene WebUI
        │
        ├── Task- und Crew-Orchestrierung (IDs, DAG, Deadline, Cancellation)
        ├── Context Builder
        │     ├── Memory: Working / Session / Long-term, Rooms, Provenienz
        │     ├── Code Index: AST-Chunks, FTS, optional lokaler Vektorindex
        │     └── Tool-output policy + abrufbarer Original-Cache
        ├── Agent adapters / capability matrix
        ├── Provider admission: bounded FIFO + Modell-Governor + Key-Lanes
        └── integrierter Loopback-Gateway / direkter ApiClient
                └── erlaubte Provider- oder Agent-CLI-Backends
```

Jede Komponente erhält explizite Grenzen und Versionen. Memory-Store, Code-Index, CCR-ähnlicher Output-Cache und Session-Logs sind unterschiedliche Datenklassen mit getrennten Aufbewahrungs-, Lösch- und Datenschutzregeln. Ein Retrieval- oder Recovery-Token ist ein opaker, autorisierungsgebundener Verweis, kein öffentliches Geheimnis und kein Ersatz für Scope-Prüfung.

## 4. Integrierter Gateway und Provider-Lanes

Bestehende Gateway-Funktionalität wird nicht neu erfunden. Die Erweiterungen sollen sicherstellen, dass direkte API-Aufrufe und kompatible Agenten-CLI-Aufrufe dieselbe Admission-API und denselben Pool verwenden.

### 4.1 Provider-Pool und Lane

Poolidentität mindestens: `provider_id + normalized_base_url + model_id + rate_limit_scope`. Eine Lane enthält Keystore-Referenz, Rate-Limit, Rolling-Window-Zeitstempel, Cooldown und nicht geheime Health-/Fehlerklassifikation. Secrets erscheinen nie in Config-Ausgaben, Logs, Debug oder Metriklabels. Legacy-Provider mit einem einzelnen Key werden rückwärtskompatibel als Ein-Lane-Pool behandelt.

### 4.2 Admission, Fairness und Governor

- Sliding Window exakt nach konfiguriertem Providerlimit; atomare Reservation vor dem Send.
- Abgebrochene Requests vor dem Senden geben die Reservation frei; gesendete Requests, Retries und Hedges zählen.
- 429/`Retry-After` wirkt auf die sendende Lane; klar klassifizierte modellweite Worker-Exhaustion auf den Modell-Governor.
- Begrenzte globale FIFO-Queue, Queue-Full-Fehler, Deadline, Cancellation und modellweite Concurrency-Permits.
- Lane-Auswahl: sofort verfügbare Conversation-Affinity, andernfalls normalisierte Auslastung und fairer Round-Robin-Tie-Break, dann Spillover. Kein unbegrenztes Warten auf Sticky-/Cache-Affinity.
- Adaptive Limits und Same-Model-Hedging bleiben spätere, standardmäßig deaktivierte Optimierungen und müssen über das Rate-/Concurrency-Budget laufen.

### 4.3 Gateway-Adaptervertrag

Capability je Agent/Backend explizit erfassen und testen: API-Base-URL, Auth-Format, unterstützte API-Oberfläche (Chat Completions, Responses, Messages), Modellabbildung, Tool-/Multimodal-Verhalten, Streaming/SSE und Credentials-Fallback. `agent-cli-to-api` zeigt den Nutzen eines einheitlichen Clientvertrags über mehrere CLIs und API-Formate. Für bkgcrewcli heißt das: vorhandenes Gateway gezielt um benötigte Adapter erweitern, nicht pauschal jede CLI als HTTP-Backend starten.

Gateway bleibt taskgebunden, Loopback-only, token-authentifiziert, modell-/requestbeschränkt und fail-closed. Tokens kurzlebig, zufällig, redigiert und beim Taskende widerrufen. Kein beliebiger URL-Proxy. Offizielle, autorisierte CLI-Login- oder Provider-API-Wege nutzen; keine Umgehung von Anbieterzugriffskontrollen oder stilles Wiederverwenden nicht dafür vorgesehener Zugangsdaten.

## 5. Memory: Shodh + Uteke als Hauptkombination

### 5.1 Eigenes Datenmodell und Scope

```text
project_id, room_id, workflow_id, task_id, session_id
agent_id, author_kind (human|agent|system), source_ref
memory_id, memory_type, tier, content_hash, version, status
created_at, updated_at, valid_from, valid_until, confidence, importance
privacy_class, tags/entities, accessed_at, access_count
relations (supports|contradicts|supersedes|references|co_retrieved)
```

Scopes sind Autorisierungsgrenzen, nicht bloß Suchfilter. Ein Projekt- oder Room-Recall muss vor der Suche auf erlaubte IDs eingeschränkt werden. Es gibt keine automatische Hochstufung von Projektwissen in globale User-Memory. Room-Zugriff sowie Import/Export und Löschung werden explizit auditiert.

### 5.2 Tiers und Lern-/Konsolidierungsregeln

- **Working:** flüchtiger Taskkontext und kompakte offene Ziele.
- **Session/Episodic:** Taskverlauf, Entscheidungen, Tests, Fehler und Übergaben mit Quellenreferenzen.
- **Long-term/Semantic:** geprüfte Fakten, Constraints, Entscheidungen und wiederverwendbare Muster/Prozeduren.

Shodh-inspirierte Zugriffssignale, Verfall und Konsolidierung können Relevanz verbessern; sie dürfen aber keine Wahrheits- oder Autorisierungsentscheidung treffen. Zunächst deterministische, konfigurierbare Regeln, Content-Hash-Dedupe und optionale lokale Embeddings. Keine zwingenden LLM-Aufrufe zum Speichern oder Abrufen. Automatische Promotion nur für eng definierte, belegte Fälle; Agenten-Ausgaben sind standardmäßig Vorschläge. Alte Episoden bleiben über Provenienz/Archiv oder Tombstones nachvollziehbar und werden nicht unbemerkt überschrieben.

Uteke-inspirierte Rooms, Autorzuordnung, Quellenzitate, point-in-time-Abfragen und explizite `contradicts`/`supersedes`-Kanten verbessern Teamarbeit und Debugging. Optimistisches `expected_version`-Update verhindert verlorene parallele Änderungen. Falsche oder abgelehnte Werte können als Tombstone gespeichert werden, damit derselbe Inhalt nicht durch Import oder Auto-Capture wiederkehrt.

### 5.3 Retrieval und Sicherheit

Erste Ausbaustufe: SQLite-FTS/strukturierte Filter, Tags, Zeit-/Roomfilter und Beziehungsgraph; Ergebnisse deterministisch fusionieren. Optionaler lokaler Embedding-/Vektorindex kommt hinter `MemoryEmbedder`/`MemoryIndex`-Traits hinzu und kann deaktiviert bleiben. Reranking nur nach Messung von Qualität, Latenz und Ressourcen. Suchergebnisse enthalten knappe Vorschau plus ID/Quelle; vollständiger Text wird gezielt nachgeladen.

Secrets vor Persistenz und Embedding redigieren oder im Strict-Modus den Write ablehnen. Memory und Code-Snippets sind untrusted data: im Prompt klar als Daten einfassen, niemals als System-/Tool-Anweisung behandeln. Privacy-/Retention-Einstellungen und „forget“ müssen auch abgeleitete Indizes und Cache-Einträge abdecken.

### 5.4 Capture statt blindem Auto-Memory

Task-Ende kann eine strukturierte Episoden-Zusammenfassung vorschlagen: Ziel, geänderte Artefakte, tatsächlich ausgeführte Tests, beobachtete Fehler, Ergebnis und offene Punkte; jeweils mit Events/Dateien als Beleg. Entscheidungen/Fakten erhalten Status `proposed` bis Review oder passende Policy greift. Kein automatisches Speichern vollständiger Prompts, Antworten oder NDJSON-Logs.

## 6. Projektübergreifender Code-Index nach srag-Ideen

Ein optionaler, lokal verwalteter Index hilft Agenten, bereits vorhandene Implementierungen und Konventionen wiederzuverwenden. Der erste native Pfad kann Rust-seitig Dateien entdecken, stabile Hashes speichern, in AST-bewusste Chunks teilen (Tree-sitter wo zuverlässig, ansonsten markierter Fallback), und inkrementell nur geänderte Dateien indexieren. Projekt-Allowlist und Ignore-Regeln schließen `.git`, Build-Artefakte, generierte Dateien und Secrets standardmäßig aus.

Suchoberfläche in Stufen:

1. exakte Symbol-/Textsuche und SQLite FTS;
2. Projekt- und Sprachfilter sowie Pattern-/Konventionszusammenfassung;
3. optional lokales semantisches Retrieval mit Vektorindex und Reciprocal Rank Fusion;
4. optionales Reranking und Call-Graph-Information nur, wenn Sprachunterstützung belastbar ist.

Ergebnis enthält Projekt, Pfad, Symbol, Zeilenbereich, Commit-/Indexversion und Relevanzgrund. Standardmäßig werden kleine Ausschnitte statt ganzer Dateien in den Prompt gegeben; Agenten können bei Bedarf die Datei oder einen Bereich gezielt anfordern. Indexierung ist inkrementell und interruptible, mit `status`, `sync`, `watch` als mögliche spätere CLI-UX. Cross-project retrieval setzt ausdrückliche Projektberechtigung voraus. Kein Code verlässt die Maschine für Embeddings, außer Betreiber konfiguriert dies ausdrücklich.

## 7. Recoverable Context-Kompression nach TinyJuice-Ideen

Tool-Ausgaben (Tests, Logs, JSON, Diff, Suche, HTML) sind ein wesentlicher Context-Kostentreiber. Statt einer pauschalen Kürzung führt bkgcrewcli eine optionale, content-aware Policy-Schicht ein:

- `off`, `light`, `full` je Agent/Tool/Task; sichere Defaults und sichtbare Gründe.
- Deterministische Reduktion für redundante Logs, strukturierte Suchtreffer, große JSON-Listen und Diffs. Fehler, Warnungen, Stacktraces, relevante Treffer, Signaturen und geänderte Zeilen bleiben priorisiert.
- Exakte Datei-Lesevorgänge standardmäßig unverändert. Kompression darf nicht stillschweigend Ergebnisse als vollständig ausgeben.
- Jede verlustbehaftete Ausgabe nennt klar, was ausgelassen wurde und stellt – sofern Original sicher gespeichert werden kann – einen kurzlebigen Recovery-Verweis bereit. Der Agent kann Details gezielt nachladen.
- Ist Recovery nicht verfügbar oder scheitert Speicherung, wird verlustbehaftete Kompression abgelehnt und Original bzw. ausdrücklich deklarierter Größenfehler weitergegeben. Keine falsche Verlustfreiheitsgarantie.
- Cache autorisiert pro Projekt/Task/Agent, hat Größen-/TTL-Limits, Verschlüsselungs-/Dateirechte-Policy und Löschpfad. IDs sind nicht erratbar; Token enthält keine Inhalte.
- Keine LLM-Zusammenfassung im ingest-Pfad der ersten Version. Spätere on-demand-Zusammenfassung braucht Modellbudget, Taskkontext, Quelle und gesicherte Original-Recovery.

`TinyJuice` selbst ist laut Referenz-README GPL-3.0. Daher keine direkte Abhängigkeit oder Codeübernahme in die MIT-lizenzierte Zielcrate ohne explizite Lizenz-/Rechtsprüfung. Zunächst wird die gewünschte Verhaltensspezifikation unabhängig umgesetzt und getestet.

## 8. Task Context Builder: Integration der vier Wissensquellen

Der Context Builder wird zentraler, deterministischer Komponist, nicht ein neuer Speicher. Eingaben: Task-Anweisung, erlaubte Projekt-/Room-Scopes, Agentenrolle, aktueller Git-Stand, relevante Workflow-Abhängigkeiten und token budget. Er ruft Code-Index und Memory getrennt ab und integriert komprimierte Tool-Artefakte nur anhand stabiler Referenzen.

Prioritätsreihenfolge:

1. feste System-/Sicherheitsregeln und aktuelle Task-Anweisung;
2. Agentenrolle/Tool-Definitionen, soweit dieser Adapter sie nicht ohnehin selbst verwaltet;
3. gepinnte, gültige Projekt-Constraints und geprüfte Entscheidungen;
4. wenige passende Code-Symbole/-Ausschnitte mit Pfad und Zeilenangabe;
5. passende Episoden/Fehler/Prozeduren samt Autor, Zeit und Quelle;
6. Dependency-Zusammenfassung und Recovery-/Artefaktverweise.

Pro Eintrag Tokenkosten schätzen, hartes Gesamtbudget einhalten, Duplikate entfernen und Widersprüche sichtbar nebeneinander darstellen. Memory- und Code-Index-Version sowie verwendete Retrieval-IDs in Diagnose speichern, Inhalte aber nicht routinemäßig loggen. Kontextbudget hat getrennte Buckets für Systemprompt, Tools, Memory, Code und Conversation. Tatsächliche Provider-Usage ersetzt Näherungswerte, sobald verfügbar. Kontextkompression ist keine Garantie für Provider-KV-Cache oder geringere Abrechnung; diese Effekte getrennt messen.

## 9. CLI und Observability: pragmatische prmt-Inspiration

`prmt` ist ein schneller, kontextsensitiver Shell-Prompt, kein Kernbaustein der Agentenplattform. Optional kann bkgcrewcli eine minimale Statusansicht für Projekt, Git-Branch, aktive Crew/Task, Queue-Wartezahl, Gateway-/Memory-Zustand und letzten Exit-Status bieten. Sie darf keine Versionserkennung oder Netzwerkzugriffe im interaktiven Prompt erzwingen und muss mit `--no-color`/Plain-Text funktionieren. Erst messen, bevor eine zusätzliche Prompt-Abhängigkeit eingeführt wird.

Die vorhandene WebUI/Diagnose sollte stattdessen zuerst verständliche Statuskarten für Lanes, Queue, Adapter-Capabilities, Memory-/Index-Health, letzte Retrieval-IDs und Kompressionsstatistiken ergänzen. Metriken ausschließlich aggregiert/redigiert: Tokens/Bytes, Strategie, Dauer, skip-Grund, Cache-Recovery-Erfolg, Recall-/Index-Latenz. Keine Rohprompts, Memory-Texte, Snippets oder Keys als Labels.

## 10. Datenhaltung, Restart und Betrieb

SQLite/SQLx bleibt Metadaten-, Memory- und FTS-Basis, wenn Tests die Anforderungen bestätigen. Speicher für Vektoren/komprimierte Originale sind getrennt und injizierbar; kein unkontrolliertes Wachstum in der SQLite-DB. Schema-Versionen, Transaktionen, Backups, Integritätsprüfung und Wiederherstellung gehören zum MVP. Indizes werden nach Crash prüfbar und aus kanonischen Records neu aufbaubar.

Pool-Sendefenster überleben Prozessneustarts konservativ: Persistenz oder Drain-/Window-Wait, wenn derselbe Key-Pool wieder aktiviert wird. Kein Multi-Process-Shared-Key-Betrieb ohne atomaren Limiter und Fencing; erste Version dokumentiert/exerzisiert genau einen aktiven Pool-Eigentümer je Key-Set. Memory, Code-Index und Recovery-Cache haben unterschiedliche Retention und Löschkommandos.

## 11. Implementierungsreihenfolge

### Phase 0 — Baseline und Verträge

- Aktuellen Fork-HEAD, Gateway-Routen, Keystore, Agentenadapter, Build/CI und Lizenzinformationen prüfen.
- Capability-Matrix pro Agent: Base-URL, Auth, API-Format, Streaming, Tools, Fallbacks.
- Schemas für Task-/Room-/Memory-/Artifact-IDs und Datenaufbewahrung festlegen.
- Test-Fixtures für Codeindex, Memory-Recall und komprimierbare Tool-Ausgaben einrichten.

**Gate:** klarer Ist-Stand und keine Annahme aus einem anderen Branch.

### Phase 1 — Memory-Grundlage (Shodh/Uteke-Kern)

- Versionierter SQLite-Store, Scopes/Rooms, Provenienz, Hash-Dedupe, Typen, Tags, Relations, CAS und Tombstones.
- FTS/strukturierter Recall, deterministische Ranking-Baseline, manuelle/proposed Writes.
- Backup/Restore, Redaction, Scope-Isolation und Löschung testen.

**Gate:** isolierte Projekte/Räume, reproduzierbarer Recall, keine verlorenen parallelen Updates.

### Phase 2 — Context Builder und Capture

- Tokenbudget, feste Prompt-Reihenfolge, Quellenangaben, Widerspruchsdarstellung und degradierter Modus.
- Task-Abschluss erzeugt Episoden-Vorschlag aus Ergebnis/Events/Artefakten, nicht vollständigen Transkripten.
- CLI `memory status/recall/inspect` nach Prüfung vorhandener Befehlsstruktur.

**Gate:** Memory optional ausfallbar, erforderliche Policy explizit fail-closed.

### Phase 3 — Code Index (srag-Ideen)

- Allowlist/Ignore-/Secret-Regeln, Hash-basiertes inkrementelles Indexieren, Symbol-/AST-Chunks, SQLite FTS.
- Projektübergreifende Suche mit Pfad-/Zeilenbelegen; später optionaler lokaler Embedder und Vektor-RRF.
- Indexstatus, Rebuild und Löschpfad.

**Gate:** inkrementelle Aktualität und Scope-Isolation; nachvollziehbare Trefferquellen.

### Phase 4 — Recoverable Tool-Output-Reduktion

- Content-Detection, `off/light/full`, zunächst deterministische Reducer und explizite Loss-Marker.
- Taskgebundener Original-Cache mit Recovery-Tool/API, TTL, Limits und Auth.
- Test-/Diff-/JSON-/Log-Fixtures, exact-read bypass und Fail-safe-Passthrough.

**Gate:** kein stiller Datenverlust; jeder verlustbehaftete Pfad ist wiederherstellbar oder wird abgelehnt.

### Phase 5 — Semantische Qualität, Pflege und UX

- Optional lokale Embeddings/ANN, Hybrid-Fusion, Candidate reranking nur anhand reproduzierbarer Evaluation.
- Memory-Decay/Konsolidierung als Vorschläge/konfigurierbare Wartung; Nutzer kann anpinnen, prüfen, zurücksetzen.
- WebUI-/Diagnoseansichten und gemessene CLI-Statusanzeige.

**Gate:** kein Retrieval-Qualitätsverlust, nachvollziehbare Pflege, Privacy- und Performanceziele erfüllt.

### Phase 6 — Provider-/Adapter-Härtung und Rollout

- Einheitliche Zähler für direkten und gatewayvermittelten Sendepfad, Adapter-End-to-End-Tests, Deadline/Cancel/429.
- Fünf-Key enforcing Mock, Modell-Concurrency und begrenzte Queue-Lasttests.
- Begrenzter Rollout mit Opt-in-Flags und Rückfall auf klar markierten Basismodus.

## 12. Evaluierung und Release-Gates

- **Memory Recall:** LongMemEval-/LoCoMo-inspirierte, eigene reproduzierbare Projektfragen; Recall@k, Quellenabdeckung, Widerspruchstreffer, falsche Scope-Treffer. Keine Benchmarks oder Zahlen aus Referenzprojekten als eigene Leistung ausgeben.
- **Code Search:** bekannte Symbole/Patterns über mehrere Testrepos, top-k Trefferqualität, Zeilenkorrektheit, inkrementelle Indexzeit und Speicherverbrauch.
- **Compression:** Tool-spezifische Fixtures mit Fakten-/Fehler-/Diff-Signalprüfungen, Ausgabegröße, Laufzeit, vollständiger Original-Roundtrip und sicherem Skip-Verhalten.
- **Context:** identische Tasks mit/ohne Retrieval; Input-Tokens, Latenz, Task-Erfolg und Regressionen getrennt berichten.
- **Gateway:** per-Lane RPM, 429/Retry-After, Queue-Wait, In-Flight, Cancellation, TTFT und Gesamtzeit.
- CI: `cargo fmt --all -- --check`, Workspace-Tests, Clippy mit `-D warnings`, Migration-/Recovery-/Security-Tests. Keine Live-Keys oder unbeschränkten Provider-Requests.

## 13. Referenzen, Lizenz und Übernahmegrenzen

Dies sind Designreferenzen, keine behauptete Kooperation oder gemeinsame Runtime:

- [Shodh-Memory](https://github.com/varun29ankuS/shodh-memory) — Apache-2.0 laut README; lokale Memory-Tiers, LLM-freies Store/Recall, Dedup, Zugriffssignale, Konsolidierung und lokale semantische Suche. **Hauptinspiration für Lifecycle/Pflege**, nicht ungeprüftes Übernehmen neurobiologischer Heuristiken.
- [Uteke](https://github.com/codecoradev/uteke) — Apache-2.0 laut README; Rooms, Autor-/Quellenangaben, FTS+Vektor-Fusion, Beziehungen, Zeitreisen, Privacy und nachvollziehbare Benchmarks.
- [srag](https://github.com/wrxck/srag) — GPL-3.0 laut README; AST-Codechunks, inkrementelles Multi-Projekt-Indexing, Hybrid Search/Reranking, Pattern- und Call-Graph-Tools, Secret-Redaction. GPL-3.0-Kompatibilität mit der MIT-Zielcrate vor jeder Code-/Link-Übernahme rechtlich prüfen; bevorzugt unabhängig spezifizieren.
- [TinyJuice](https://github.com/tinyhumansai/tinyjuice) — GPLv3 laut README; profilgesteuerte, inhaltsbewusste Tool-Output-Reduktion und explizite Original-Recovery. Keine direkte Abhängigkeit/Codekopie ohne Lizenzprüfung.
- [agent-cli-to-api](https://github.com/leeguooooo/agent-cli-to-api) — heterogene CLI-Backends hinter kompatiblen API-Endpunkten, Modellrouting und Streaming als Adapter-Ideen. Lizenz und konkrete Auth-/Backend-Verträge vor Übernahme prüfen.
- [prmt](https://github.com/3axap4eHko/prmt) — MIT laut README; schneller, kontextsensitiver Shell-Prompt, nur optionale Terminal-UX-Inspiration.

Zunächst werden nur Konzepte und beobachtbare Verträge übernommen. Für jede tatsächliche Code- oder Dependency-Übernahme sind exakte Revision, Lizenzdatei, Copyright/NOTICE, Transitivabhängigkeiten und Kompatibilität mit der Zielcrate zu prüfen.

## 14. Hauptrisiken und Gegenmaßnahmen

| Risiko | Gegenmaßnahme |
|---|---|
| Agent umgeht Pool oder nutzt unerwartete Credentials | Capability-Tests, fail-closed bzw. sichtbarer ungepoolter Modus, Secret-Redaction |
| Projektwissen überschreitet Scope | Autorisierung vor Retrieval, explizite Rooms/Allowlist, Isolationstests |
| falsches Memory oder Prompt Injection | Provenienz, Vorschlagsstatus, Tombstones, Widerspruchsbeziehungen, untrusted-data framing |
| Code-Index leakt Secrets oder Fremdprojektcode | lokale Verarbeitung, Ignore-/Secret-Filter, Scopeprüfung, minimale Snippets |
| Kompression lässt wichtige Fakten verschwinden | markierte Auslassungen, Recovery-Cache, exact-read bypass, golden fixtures |
| Embedding/Reranker erhöht Komplexität oder Latenz | optionale Trait-Grenze; FTS/structured baseline zuerst |
| dynamischer Decay löscht wertvolles Wissen | keine Hard-Delete-Promotion automatisch; Pins, Audit, Backup und Review |
| Referenzlizenzen kollidieren | keine ungeprüfte GPL-Codeübernahme; Lizenzgate vor Dependency/Copy |
| Kontext-/KV-Cache-Effekt wird überschätzt | Tokenreduktion, Provider-Cache-Usage und Qualität separat messen |

## 15. Definition of Done

- Provider-Gateway und kompatible Agenten nutzen nachweislich einen gemeinsamen Lane-/Admission-Pfad; inkompatible Agenten werden sichtbar gekennzeichnet.
- Memory ist lokal-first, versioniert, scoped, mit Herkunft versehen, redigiert, export-/backupfähig und sicher löschbar.
- Memory-Retrieval ist im Kern ohne LLM-Aufruf nutzbar; semantische Modelle sind optional und offlinefähig, sofern aktiviert.
- Code-Retrieval liefert überprüfbare Projekt-/Pfad-/Zeilenquellen und darf keine nicht freigegebenen Projekte durchsuchen.
- Tool-Output-Kompression ist transparent; verlustbehaftete Inhalte sind wiederherstellbar oder Kompression wird abgelehnt.
- Context Builder hält ein hartes Budget ein und protokolliert nur redigierte Retrieval-/Nutzungsmetadaten.
- Tests belegen Isolation, Cancellation, Restart, Recovery, Scope, Migration und Lizenz-/Secret-Redaction.
- Bestehende CLI-, Config-, Event-, Gateway- und Einzelkey-Verträge bleiben rückwärtskompatibel oder erhalten eine dokumentierte Migration.
