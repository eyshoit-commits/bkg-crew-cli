# Zielarchitektur: bkgcrewcli als lokale Agenten- und Provider-Plattform

**Ziel-Repository:** `eyshoit-commits/bkg-crew-cli`  
**Basis:** `main`; vor jeder Implementierung den echten HEAD, Module und Tests neu verifizieren.  
**Status:** Architektur- und Implementierungsplan; keine Runtime-Implementierung in diesem Dokument.  
**Eigenentwicklung:** Die genannten Repositories sind ausschließlich Ideengeber. Es wird kein Quellcode kopiert, kein Referenzprojekt als verpflichtende Runtime-Abhängigkeit eingebunden und keine Zusammenarbeit behauptet. Wir schreiben und testen die Implementierung selbst.

## 1. Vision und Prioritäten

bkgcrewcli soll Agenten, Providerzugriff, Crew-Workflows und Projektwissen in einer überprüfbaren lokalen Plattform vereinen. Es soll nicht einfach möglichst viele Features anhäufen: Jede Funktion braucht eine klare Zuständigkeit, Berechtigungsgrenze, Messbarkeit und Ausfallstrategie.

Priorität und Abhängigkeiten:

1. **Nim-proxy-inspirierte Provider-Lanes und Admission-Control** als zuverlässige Unterlage für alle verwalteten Provider-Sends.
2. **Copilot-API-inspirierte, multi-protokollfähige Gateway-Verträge**: mehrere Client-Protokolle, Provider-/Modellrouting, Capabilities, Streaming und Diagnose – innerhalb der bereits vorhandenen Rust-Gateway-Basis.
3. **Crew-/Task-Orchestrierung** auf den vorhandenen Agenten-, Prozess- und Queue-Funktionen aufbauen.
4. **Shodh-/Uteke-inspiriertes, lokales und belegbares Projekt-/Crew-Memory** plus deterministischer Context Builder.
5. **srag-inspirierte Code-Suche** und **TinyJuice-inspirierte, wiederherstellbare Tool-Output-Reduktion**, nachdem Scope, Storage und Recovery definiert sind.
6. **Prmt-inspirierte Terminal-UX** nur, falls sie messbar nützlich ist; kein Kernsystem und keine notwendige Abhängigkeit.

Alles ist selbst zu implementieren. „Feature übernehmen“ bedeutet hier, gewünschtes Verhalten zu spezifizieren, nicht Code, interne Endpunkte oder nicht dokumentierte Authentifizierungsverfahren nachzubauen.

## 2. Tatsächlicher Ziel-Repository-Kontext

Das Ziel ist `eyshoit-commits/bkg-crew-cli`, nicht `bkg-zig/bkg-crew-cli`. Das README des Ziels beschreibt ein Rust-CLI mit Agent-Subprozessen (Claude Code, Codex, OpenCode, Cursor), einheitlichem NDJSON-Eventstream, Model Registry und Projektkonfiguration. Die Architektur-Referenz im Ziel beschreibt außerdem einen integrierten `serve`-Gateway mit Chat Completions, Responses und Anthropic Messages, Provider-scoped Routen, Tokens, Streaming/Usage sowie einer vorhandenen WebUI.

Vor Implementierung muss der aktuelle Ziel-HEAD mit den tatsächlichen Dateien verglichen werden. Insbesondere nicht annehmen, dass `src/crew`, `src/cognition`, `src/vault`, `src/mender`, `src/task_queue` oder andere Module aus dem Upstream-Repository im Ziel vorhanden sind. Bestehende Dateien/Module nur nutzen, wenn sie im Ziel-HEAD tatsächlich existieren; andernfalls als neue Integrationspunkte planen.

## 3. Architekturübersicht

```text
CLI / Public Rust API / vorhandene WebUI
        │
        ├── Task- und Crew-Orchestrierung
        │     └── DAG, durable state, retry policy, deadline, cancellation
        ├── Context Orchestrator (Budget + provenance)
        │     ├── Memory store: project / room / task scopes
        │     ├── Code index: symbols / FTS / optional local vectors
        │     └── Artifact store: tool output + explicit recovery handles
        ├── Agent adapters + verified capability matrix
        ├── Provider Gateway: Chat / Responses / Messages adapters
        └── Shared Provider Admission API
              ├── bounded FIFO dispatcher
              ├── per-key sliding-window lanes and cooldowns
              ├── model/account concurrency governor
              └── provider transports (HTTP / supported WebSocket)
```

Provider routing, Agent-CLI compatibility, task scheduling, memory, code indexing und Artefaktkompression sind getrennte Komponenten, aber werden mit stabilen IDs und Events integriert. Alle Persistenzen sind lokal und haben unterschiedliche Retention-, Verschlüsselungs-, Backup- und Löschregeln.

## 4. Fundament: nim-proxy-inspirierte Lane- und Admission-Schicht

Die zentrale Lane-Logik kommt zuerst: weder Agenten noch Gateway-Routen dürfen Rate-Limits, Concurrency-Regeln oder Warteschlangen umgehen. Der Zielprozess kann vorhandene Gateway-/Provider-Module wiederverwenden, falls sie im aktuellen HEAD existieren; der Lane-Mechanismus wird als eigene Rust-Implementierung spezifiziert.

### 4.1 Provider-Pool und Lane

Pool-Schlüssel: `provider_id + normalized_endpoint + model_or_shared_scope + rate_limit_scope`. Jede Lane verweist nur über eine geheime Keystore-ID auf Credentials und hält `rpm_limit`, aktiv/inaktiv, Sliding-Window-Sendezustand, Cooldown und nicht geheime Health-/Fehlerklasse.

- Bestehende einzelne Provider-Keys bleiben als Pool mit einer Lane kompatibel.
- Keys werden nur dann über einen Pool addiert, wenn ihre realen Upstream-Limits dies erlauben. Keine Durchsatz- oder ToS-Umgehung versprechen.
- API-Key-Text darf nicht in Logs, Metrikenlabels, `Debug`, Dry-Run oder Config-Ausgaben erscheinen.

### 4.2 Limiter, Queue und Auswahl

- Sliding Window passend zum echten Providerlimit; kein burstfähiger Token-Bucket, wenn der Upstream Rolling RPM durchsetzt.
- Reservationen sind atomar. Vor-Send-Abbruch erstattet ungesendete Slots; gesendete Requests, Retries und Hedges zählen.
- Begrenzte zentrale FIFO-Warteschlange mit Queue-Full, Taskdeadline, Cancellation und konfiguriertem Max-Wait.
- Lane-Auswahl: sofort nutzbare Conversation-Affinity, sonst normalisierte Auslastung (`in_window / rpm_limit`) mit round-robin-Tie-Break, dann Spillover. Affinity darf nicht unbegrenzt blockieren.
- 429/`Retry-After` kühlt die betroffene Lane ab. Modellweite Worker-Exhaustion regelt nur den Modell-Governor, nicht pauschal alle Lanes.
- Request braucht RPM-Reservation und Modell-Concurrency-Permit; RAII-/Drop-Guards geben Permits auf jedem Ende frei.
- Konversations-affine Lane kann Provider-Prefix-Caching begünstigen, garantiert aber weder Cache-Hit noch Tokenrabatt.
- Retries sind begrenzt, klassifiziert und deadline-aware. Hedging bleibt standardmäßig deaktiviert und muss dasselbe Modell, Semantik und dieselben Budgetregeln nutzen.

### 4.3 Neustart und Mehrprozessbetrieb

In-memory Ratezustand allein ist nach Restart keine sichere Grundlage. Für aktivierte Pools ist persistierter Zustand oder ein konservativer Drain-/Window-Wait erforderlich. Mehrere aktive Prozesse dürfen dieselben Keys nicht ohne gemeinsamen atomaren Limiter und Fencing ausgeben. Der erste sichere Betrieb sollte genau einen Pool-Eigentümer je Key-Set erzwingen oder klar dokumentieren.

### 4.4 Streaming-Verhalten

SSE-Parser und -Writer müssen fragmentierte Events, Keepalive, EOF, Downstream-Disconnect, Cancellation, Usage-Endevents und Deadline behandeln. Ein Streaming-Request darf dem Client nur dann Erfolg signalisieren, wenn das gewählte Protokoll spätere Fehler zuverlässig als terminales Event ausdrücken kann. Heartbeats sind keine Verlängerung der absoluten Deadline.

## 5. Copilot API: Feature-Abdeckungsprüfung und sinnvolle Kombination

Die öffentliche `caozhiyuan/copilot-api`-README beschreibt einen lokalen Universal-Gateway für OpenAI Chat Completions, Responses und Anthropic Messages, mehrere eingebaute und Drittanbieter-Provider, Agenten-Setups, SSE/WebSocket, Tool-/Modellkompatibilität, Model Discovery/Mappings, Auth-/Konfiguration, Usage Monitoring und eine Electron-Desktop-App. Die Liste unten macht transparent, was der Zielplan übernimmt, was durch vorhandene Ziel-Funktionalität bereits adressiert ist und was bewusst nicht in den Kern wandert. Das ist eine Architekturabdeckung, kein Claim gleicher Implementierung oder vollständiger Feature-Parität.

| Copilot-API-Idee | Plan im bkgcrewcli-Ziel | Entscheidung / zusätzlicher Nachweis |
|---|---|---|
| Ein Gateway für Chat Completions, Responses und Messages | **Beibehalten und härten**; bestehende Gateway-Routen des Zielrepos prüfen, Protokolle als typisierte Adapter | Contract- und Round-trip-Tests je Endpoint, Client und Upstream; keine pauschale Übersetzung aller Felder behaupten |
| Mehrere Provider und Providerwahl | **Erweitern** über Registry, explizite Provider-Profile und Modelle; jeden Send über gemeinsame Lane-/Admission-API routen | Feature-Support matrix je Provider, Endpoint und Modell; unbekannte Kombination klar ablehnen |
| Agent-CLI-fertige Konfiguration | **Beibehalten/erweitern** über AgentCapabilities und geprüfte Config-Templates | Pro Agent testen: Base URL, Auth, Modell, tool calls, stream und kein unerwarteter Credential-Fallback |
| Chat/Responses/Messages Übersetzung | **Selektiv implementieren** als explizite Adapter, keine lossy magic translation | Schema-Kompatibilität versionieren; nicht unterstützte Tools/Modalitäten in Capability-Antworten ausweisen |
| SSE über mehrere Protokolle | **Übernehmen** mit Parser-Fixtures, Heartbeats und Cancellation | Chunk-Fragmentierung, Disconnect, Last-Event/Usage und Fehlerfälle testen |
| WebSocket-Upstream für Responses | **Optionaler späterer Transport** hinter demselben Admission-/Usage-Vertrag | Nur wenn Zielprovider es dokumentiert unterstützen und HTTP/SSE Tests/Benchmarks belegen Nutzen |
| Modellkatalog, Discovery, Aliases, Mapping | **Erweitern** aus bestehender Model Registry/scoped routes | Cache, TTL, single-flight Refresh, stabile IDs; Discovery darf keine Rate-Limit-Sends verbrauchen, wenn Provider dies ermöglicht |
| Tool-Use, tool_choice, Tool-Search/MCP bridge | **Teilweise, explizit**: standardisierte Tool-Calls und optionale deferred-tool-Schnittstelle | JSON-Schema, tool_choice, Round-trip IDs und Tool-Ergebnisgrenzen testen; nichts stillschweigend entfernen |
| Multimodalität (Bilder/Dateien) | **Capability-gesteuert** | Payload-Limits, MIME-/Format-Validierung, Redaction/Logs; keine Inhalte in Diagnose kopieren |
| OAuth-/Provider-Credentials | **Nur autorisierte, dokumentierte Wege** über bestehende Keystore/Auth-Adapter | Keine nicht dokumentierte Credential-Extraktion, privaten Endpunkte oder Zugriffskontrollumgehung; Providerzugang ist je Anbieter zu prüfen |
| Usage, Token/Kosten-/Quota-Ansichten | **Übernehmen**, an bestehende Usage/WebUI anschließen | exakte Providerusage von Schätzwerten trennen; Modellpreise/Quoten mit Quelle und Zeitstempel versionieren |
| Logs, Diagnose, Konfiguration | **Übernehmen** mit redigierten Request-Metadaten, Doctor und Konfigurationsvalidierung | Keine Requestinhalte in standardmäßigen Logs; Secrets immer redigieren |
| Desktop Electron App | **Nicht in den Rust-Kern übernehmen** | Vorhandene WebUI/CLI ist bereits die passende Oberfläche; zusätzliche Desktop-Runtime wäre Overhead und doppelte Konfigurationsquelle |
| Node/Bun-Paket-/Installationsmodell | **Nicht übernehmen** | Ziel ist native Rust-CLI/Server, keine parallele Runtime |
| Provider-spezifische Subscription-/OAuth-Tricks | **Nicht als allgemeine Gateway-Funktion** | Nur öffentlich unterstützte und autorisierte Providerintegration. Keine Reverse-Engineering- oder ToS-Umgehungsziele |

**Ergebnis der Abdeckung:** Die aktuell geprüfte Architektur-Referenz des Ziels deckt bereits Chat Completions, Responses, Messages, scoped routing, Tokens, Streaming, Usage und WebUI ab. Sie belegt aber nicht allein die volle Copilot-API-Abdeckung von Multi-Provider-Capabilities, durchgängigen Tool-/Multimodal-Roundtrips, modellbezogener Transportwahl, robustem Discovery/Mapping, getrenntem Quota-/Usage-Modell oder Desktop-App-Funktionen. Die obige Tabelle markiert diese als neue/zu verifizierende Arbeit oder bewusste Nicht-Ziele. **Wir sollten nicht behaupten, Copilot-API sei vollständig absorbiert, bis die Feature-Matrix pro Protokoll/Provider mit Tests ausgefüllt ist.**

## 6. Agent-Adapter und Kompatibilitätsmatrix

Jeder Agent und Provider bekommt explizite Capabilities, z. B.:

```text
protocols: chat | responses | messages
streaming: sse | websocket | none
features: tool_calls, tool_choice, json_mode, images, files, usage
routing: configurable_base_url, auth_scheme, model_mapping
limits: request_bytes, context_tokens, max_output_tokens
fallback: none | explicitly_unmanaged
```

Diese Angaben sind nicht bloß Konfiguration; CI muss sie durch contract tests verifizieren. Ein Adapter darf niemals als „durch Lanes gepoolt“ ausgewiesen werden, wenn er tatsächlich intern eigene direkte Provider-Credentials verwendet. Nicht unterstützte Felder werden strukturiert zurückgemeldet statt still verworfen.

## 7. Crew-Task-Orchestrierung

Auf den tatsächlich existierenden Task-/Crew-Modulen aufbauen. Workflow ist ein dauerhafter DAG aus typed tasks mit `workflow_id`, `task_id`, Abhängigkeiten, Agent/Provider-Zuweisung, Deadline, Retry-Policy, Cancellation-Token, Workspace-Sandbox und Result-Ref.

- Ergebnisse zwischen Aufgaben als begrenzte Dependency-Summaries und Artefakt-IDs übergeben, nicht gesamte Transkripte.
- Taskzustände und Event-IDs crash-konsistent persistieren; nach Restart eindeutige Fortsetzung oder explizit `interrupted` melden.
- Tasks können unterschiedliche Agents/Modelle nutzen, aber Modellwahl und Fallback sind explizite Workflow-Policy, nicht stille Substitution.
- Side effects (Dateiänderungen, Commands, Commits) bleiben über Sandbox-/Permission-Policy und menschliche Freigaben steuerbar.
- Idempotency keys für wiederholte Workflow-Submission und Tool-/Memory-Capture verwenden.

## 8. Memory-Kern: Shodh + Uteke

### 8.1 Datenmodell und Scope

```text
project_id, room_id, workflow_id, task_id, session_id
agent_id, author_kind (human|agent|system), source_ref
memory_id, type, tier, content_hash, version, status
created_at, valid_from, valid_until, confidence, importance
privacy_class, tags/entities, accessed_at, access_count
relations: supports | contradicts | supersedes | references | co_retrieved
```

Projekt- und Room-Scope sind echte Berechtigungsgrenzen; erlaubte IDs müssen vor Recall gefiltert werden. Projektwissen wird nicht automatisch in globale User-Memory hochgestuft. Writes sind versioniert/optimistisch (`expected_version`). Autoren-/Quellen-Provenienz bleibt erhalten. Dedupe ist idempotent nach Scope, normalisiertem Inhalt und Quelle.

### 8.2 Tiers und Lernregeln

- **Working:** flüchtige Taskziele, aktueller Fortschritt, offene Fragen.
- **Session/Episodic:** Taskergebnisse, Test-/Fehlerbeobachtungen, Übergaben und Quellen.
- **Long-term/Semantic/Procedural:** geprüfte Fakten, Constraints, Entscheidungen, Muster und Prozeduren.

Shodh-inspirierte Zugriffsstärke, Vergessen/Decay und Konsolidierung dienen nur dem Ranking und der Wartung – niemals Wahrheitsprüfung oder Scope-Autorisierung. Deterministische Regeln zuerst. Kein LLM-Aufruf ist für grundlegendes Speichern oder Abrufen erforderlich. Promotion aus Episoden in geprüfte Fakten bleibt Review-/Policy-gesteuert. Originalbelege bleiben archiviert/verknüpft.

Uteke-inspirierte Rooms, Zeitabfragen, Quellenzitate, typed relations, Widerspruchs- und Supersession-Kanten sowie tombstones für explizit zurückgewiesene Werte machen Memory nachvollziehbar und konfliktfest.

### 8.3 Recall und Sicherheit

Erste Version: strukturierte SQLite-Filter + FTS, Tags/Entities, Zeit-/Roomfilter und relationale Links; deterministisches Ranking/RRF. Lokale Embeddings/ANN und Cross-Encoder bleiben optionale, austauschbare Traits, bis Qualität, Modellgröße, Offlineverhalten und Latenz gemessen sind.

Secrets vor Persistenz und Embedding redigieren; optional strikter Modus lehnt sensible Writes ab. Recalled Memory/Code ist untrusted data und wird im Prompt als Dateninhalt gekennzeichnet. „Forget“ entfernt auch abgeleitete Index-/Cache-Inhalte, mit nachvollziehbarem Tombstone, wenn nötig.

### 8.4 Capture statt blindem Auto-Save

Beim Taskabschluss kann ein Review-fähiger Episodenvorschlag entstehen: Ziel, Änderungen/Artefakte, tatsächlich ausgeführte Tests und deren Ergebnis, beobachtete Fehler, Lösung und offene Fragen. Jeder Claim braucht Event-/Datei-/Tool-Quelle. Keine vollständigen Prompts oder Session-Logs ungeprüft als Memory speichern.

## 9. Projektübergreifender Code-Index: srag-Ideen

Lokaler optionaler Index mit Allowlist pro Projekt, `.gitignore`-/Build-/generated-Ausschlüssen, Secret-Scrub vor Persistenz und inkrementeller Hash-Synchronisierung.

1. Exakte Suche und SQLite FTS als robuste Basis.
2. Symbol-/AST-aware Chunks mit Tree-sitter, Zeilenbereichen und markiertem Fallback.
3. Cross-project Pattern-/Convention-Suche mit expliziter Projekt-Allowlist.
4. Optional lokaler Vector Search und Reciprocal Rank Fusion.
5. Caller/Callee- und Call-Graph-Kontext nur für Sprachen/Parser mit validierter Genauigkeit.

Treffer liefern Projekt, Pfad, Zeilen, Symbol, Index-/Commit-Version und Relevanzgrund. Standardmäßig gibt es kleine Snippets; Inhalte können gezielt mit Datei-/Zeilenrange nachgeladen werden. Keine Projektübergreifende Suche außerhalb der autorisierten Allowlist.

## 10. Wiederherstellbare Tool-Output-Kompression: TinyJuice-Ideen

Content-aware Policies für Logs, Tests, Diffs, JSON, Search und HTML; pro Agent/Tool/Task `off`, `light`, `full`. Deterministische Reducer behalten Fehler, Warnungen, Stacktraces, geänderte Zeilen, Signaturen, relevante Treffer und Anomalien. Exakte Datei-Lesevorgänge bleiben standardmäßig unberührt.

Verlustbehaftete Ausgabe muss Auslassungen offen markieren. Original wird in task-/scopegebundenem Artefakt-Store mit TTL, Size Cap, Auth und Löschpfad hinterlegt und per Recovery-Tool selektiv abrufbar. Wenn Original nicht sicher gespeichert werden kann, verlustbehaftete Kompression ablehnen bzw. klaren Größenfehler melden; niemals still abschneiden. Keine automatische LLM-Zusammenfassung im ingest-Pfad der ersten Version.

TinyJuice- und srag-README kennzeichnen GPLv3/GPL-3.0. Der Zielrepo ist laut README MIT-lizenziert. Es wird weder Code kopiert noch eine direkte Runtime-Abhängigkeit eingezogen; wir implementieren unabhängige Spezifikationen und lassen Lizenz-/Dependency-Policy vor Release prüfen.

## 11. Context Orchestrator

Ein einziger deterministischer Builder kombiniert Task, Memory, Code-Suche, Artefaktverweise und Adapterwissen unter hartem Tokenbudget. Es ist kein weiterer Speicher.

Prioritäten:

1. System-/Safety-Policy und unveränderte aktuelle Taskanweisung.
2. Agent-Rolle/Tools, soweit nicht native Agentenconfig.
3. Gepinnte, gültige Projekt-Constraints und geprüfte Entscheidungen.
4. Wenige Code-Symbole/Snippets mit zitierbarer Quelle.
5. Relevante Episoden, Fehler, Reparaturen und Prozeduren mit Provenienz/Alter.
6. Dependency-Summaries, Recovery- und Artefakt-IDs.

Je Block Tokenkosten schätzen, Duplikate reduzieren, Widersprüche markieren, Max-Einträge durchsetzen und verarbeitete Quellen referenzieren. Getrennte Context-Buckets (system, tools, memory, code, conversation) ermöglichen Diagnose. Provider-gemeldete Usage übersteuert lokale Schätzung. Kein Versprechen, dass Prefix-Stabilität Upstream-Cache oder Kostensenkung bewirkt.

## 12. WebUI, CLI und Betrieb

Vorhandene WebUI und Diagnoseflächen nutzen, statt einen zweiten Admin-Client einzuführen. Ergänzungen in sinnvoller Reihenfolge: Pool-/Lane-Status, Queue/Modell-Capacity, Provider-Protokollmatrix, Agent-Capabilities, Usage/Quota, Memory-/Index-Health, Retrieval-/Recovery-Metadaten und Doctor.

Dashboard zeigt Trends, Health, Kapazität, Queue-/TTFT-/Latenz-Aufschlüsselung, pro-Modell/per-Lane-Auslastung und Providerusage. Metriken nutzen counts/sizes/status/strategy statt Prompt-, Memory- oder Codeinhalt. „Measured“, „estimated“ und „unavailable“ werden unterschieden. Retention ist konfigurierbar.

Prmt ist nur Inspiration für schnellen Kontextstatus (Projekt, Branch, aktiver Workflow, Queue, letzte Tests), nicht als Dependency. Keine langsame Versionsabfrage/Netzwerkzugriffe pro Shell-Prompt; Plain-Text und no-color sind Pflicht.

## 13. Phasen und Abnahmekriterien

### Phase 0 — HEAD-Audit und Feature-Matrix

- Ziel `eyshoit-commits/bkg-crew-cli` HEAD, bestehende Gateway-/Provider-/Agent-/WebUI-/Task-Module, Crate-Lizenz und Tests auditieren.
- Copilot-API-Tabelle für jeden Endpoint, Provider, Model discovery/mapping, tool use, streaming/transport, usage/auth ausfüllen: exists / partial / missing / intentionally excluded plus Testreferenz.
- Pro Agent/Provider capability matrix; nur unterstützte und dokumentierte Auth- und Endpoint-Verträge.

**Gate:** kein Upstream-Modul irrtümlich als Zielrepo-Basis vorausgesetzt; jeder gewünschte Featurebereich hat Status und Beleg.

### Phase 1 — Lanes + Dispatcher + Governor

- ProviderPool/Lane/Reservation, exact sliding window, persistent/conservative restart state, FIFO Queue, deadlines, cancellation, per-model permits, cooldown taxonomy.
- Alle verwalteten HTTP-/SSE-/ggf. WebSocket-Sends über dieselbe Admission API.
- Fünf-Key enforcing mock mit 429/Retry-After, worker slots, disconnect und cancellations.

**Gate:** keinerlei Rate-Verstöße im Mock, kein Permit/Reservation-Leak, keine versteckten Sends außerhalb des Pools.

### Phase 2 — Multi-protocol Gateway-Abdeckung (Copilot-API-Inspiration)

- Bestehende Chat/Responses/Messages contracts härten und Endpoint-Matrix vervollständigen.
- Multi-provider routing, model mappings/discovery, tool round-trip, streaming und Usage konsistent anbinden.
- Provider-/Modellfähigkeiten und unsupported parameters explizit ausweisen; CLI config examples und Doctor.
- WebSocket nur nach protokollspezifischem Nutzen-/Kompatibilitätsnachweis.

**Gate:** Contract tests je Feature-Matrix-Zelle; keine falsche Full-Parity-Aussage.

### Phase 3 — Durable Crew-/Taskkoordination

- Task DAG, IDs, persisted status/events, bounded dependency summaries, retry/deadline/cancel und Sandbox-Bindung.
- Idempotente Submission und restart/recovery Tests.

**Gate:** parallele/abgebrochene/restarted Workflows behalten nachvollziehbaren Zustand.

### Phase 4 — Memory + Context Builder

- Scopes/Rooms/Provenienz/FTS, CAS, Dedupe, relations, tombstones, backup/restore und secret redaction.
- Token-budgetierter Context und task evidence capture proposals.

**Gate:** kein cross-project leak; Memory kann ausfallen ohne unerwarteten Taskverlust; required-policy ist explizit fail-closed.

### Phase 5 — Code Index + Recoverable artifacts

- Inkrementeller FTS/AST-Index und berechtigte Suche; optionale Vektoren nach Quality Gate.
- content-aware reducers, loss markers, Task Recovery API, TTL/capacity and fail-safe passthrough.

**Gate:** Treffer sind zitierbar; kein stiller Verlust; recovery round-trip und Löschung funktionieren.

### Phase 6 — UX, adaptive policy and rollout

- Aggregierte WebUI-/Metrics-/Doctor-Anzeigen, opt-in flags, Canary und klare Downgradepfade.
- Adaptive lane/concurrency policies oder hedging nur nach reproducible load tests, default off.

**Gate:** Sicherheit, Lizenz, Usage/Kostenklarheit, Performance und Rollback getestet.

## 14. Tests und Qualitätsmessung

- **Gateway:** Protocol contract tests, tool-call round-trip, model mapping/discovery, malformed/unsupported parameters, SSE fragmentation, WebSocket only if enabled, auth and redaction.
- **Lanes:** strict rolling-window enforcement, FIFO fairness, sticky spill, cooldown, 429/Retry-After, model governor, restart, multiprocess fencing.
- **Crew:** DAG edges, idempotency, process cancellation, retry policy, crash recovery, sandbox permissions.
- **Memory:** scope isolation, CAS conflict, dedupe, time-travel, relations, tombstones, privacy filters, backup/restore.
- **Code search:** expected top-k symbols, exact line ranges, generated/secret exclusions, incremental hash update and project allowlist.
- **Compression:** no-loss/marked-loss behavior, original retrieval exactness, safety failures, fixture signal preservation, TTL and hard caps.
- **Evaluation:** own reproducible datasets and receipts; report recall@k, source coverage, false scope hits, task success, tool/context tokens, recovery success, latency and storage. Do not copy reference benchmark claims as bkgcrewcli results.
- **CI:** `cargo fmt --all -- --check`, `cargo test --workspace`, `cargo clippy --workspace --all-targets -- -D warnings`, migrations, security and license checks. No live keys or unlimited real-provider load tests.

## 15. Referenzen und Übernahmegrenzen

Alle sind Verhaltens-/Designreferenzen. Keine Quellcodeübernahme:

- [bkg-zig/nim-proxy](https://github.com/bkg-zig/nim-proxy) — MIT laut README; Sliding-Window-Lanes, FIFO dispatcher, affinity/spillover, Retry-After, Modell-Concurrency, SSE heartbeats, deadlines, usage and operational dashboards. **Primäre Vorlage für Provider Admission**, unabhängig in bkgcrewcli implementieren.
- [caozhiyuan/copilot-api](https://github.com/caozhiyuan/copilot-api) — MIT laut Repository-Metadaten; Universal Gateway, Chat/Responses/Messages, provider/model routing, agent configuration, SSE/WebSocket, discovery/mappings, tools, usage and desktop UX. **Vorlage für die zu testende Multi-protocol Feature-Matrix**, kein Claim vollständiger Parität.
- [varun29ankuS/shodh-memory](https://github.com/varun29ankuS/shodh-memory) — Apache-2.0 laut README; local-first memory tiers, offline/no-LLM recall, dedup, usage signals and consolidation.
- [codecoradev/uteke](https://github.com/codecoradev/uteke) — Apache-2.0 laut README; Rooms, author/source, hybrid retrieval, relations, time-aware recall, privacy and reproducible evaluation.
- [wrxck/srag](https://github.com/wrxck/srag) — GPL-3.0 laut README; incremental AST-aware cross-project indexing, hybrid semantic/full-text search, patterns, caller/callee and secret filtering. Concept-only; independently implement.
- [tinyhumansai/tinyjuice](https://github.com/tinyhumansai/tinyjuice) — GPLv3 laut README; profile-aware content reducers and recoverable lossy outputs. Concept-only; independently implement.
- [leeguooooo/agent-cli-to-api](https://github.com/leeguooooo/agent-cli-to-api) — protocol facade and multiple CLI backend adapter patterns; confirm exact license and provider-supported auth before any future dependency/copy (not planned).
- [3axap4eHko/prmt](https://github.com/3axap4eHko/prmt) — MIT laut README; fast, context-aware terminal presentation only.

Vor jedem Release werden Ziel-Lizenz, alle direkten/transitiven Dependencies und SBOM erneut geprüft. Besonders GPL-3.0-Referenzen bleiben konzeptionell, weil der Ziel-README MIT nennt. Auch MIT/Apache-Repos werden nicht kopiert: dies hält Ownership, Architektur und Lizenzlage klar.

## 16. Definition of Done

- Provider-Lanes/Queue/Model-Governor decken sämtliche verwalteten Provider-Sends ab und bestehen enforcing Rate-/Concurrency-Tests.
- Copilot-API-Feature-Matrix ist ehrlich ausgefüllt: jede Zelle getestet oder als partial/intentionally excluded begründet; keine unbelegte Aussage „alles absorbiert“.
- Agenten-Kompatibilität und unverwaltete Fallbacks sind sichtbar und überprüfbar.
- Crew-Tasks sind durable, bounded, cancellable und sandbox-/permission-aware.
- Memory ist lokal-first, scoped, nachvollziehbar, redigiert, export-/backupfähig und sicher löschbar.
- Codeindex liefert belegte Treffer nur für autorisierte Projekte.
- Verluste durch Kompression sind sichtbar und wiederherstellbar oder Kompression findet nicht statt.
- Context Builder hat ein hartes Budget; Metriken speichern keine Inhalte.
- Keine Referenz-Quellcodes oder unfreigegebenen Runtime-Abhängigkeiten; Lizenz-, Security-, CI- und Rollback-Gates bestehen.
