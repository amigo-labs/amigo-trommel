# amigo-trommel — CLAUDE.md

Anweisungen für Claude Code zum Bauen von amigo-trommel basierend auf den Specs in `docs/specs/audio/`.

## Projekt-Übersicht

amigo-trommel ist ein TypeScript-natives Game-Audio-Authoring-Tool. Sample-basierte Voices, Strudel-Pattern-System, Engine-Integration für die Amigo Engine.

**Wichtiges Naming:** Das Tool heißt **amigo-trommel**, nicht Trommel allein oder Chispa.

## Spec-Lesepfad

Bevor du mit der Implementation startest, lies in dieser Reihenfolge:

1. `docs/specs/audio/_index.md` — Übersicht
2. `docs/specs/audio/amigo-trommel.md` — Hauptspec (Architektur, Workflow)
3. Die für deine aktuelle Aufgabe relevanten Detail-Specs

Spec-Status-Flow: `draft → ready → implementing → stable`. Initial sind alle Specs auf `draft`.

## Stack-Constraints

- **Frontend:** Svelte 5 (kein React, kein Vue)
- **Audio-DSP:** Rust, kompiliert nach WASM (Web) und nativ (Desktop)
- **TS-Kompilierung:** SWC (nicht tsc für Produktiv-Builds)
- **JS-Runtime Web:** Browser-natives JS
- **JS-Runtime Native:** QuickJS-NG via rquickjs (nicht V8, nicht Boa)
- **Desktop-Wrapper:** Tauri v2 (nicht v1, nicht Electron)
- **Pattern-Sprache:** Strudel direkt verwendet, nicht reimplementiert

## Architektur-Regeln

### Wo Audio-DSP läuft

Audio-DSP läuft **immer** in Rust. JavaScript triggert Voices (sagt *was* passieren soll), Rust führt aus (macht *wie*). Das ist nicht verhandelbar — kein DSP-Code in JS.

### Wo Strudel läuft

Strudel-Pattern-Evaluation läuft im JS-Layer (Control-Thread), nicht im Audio-Thread. Strudel produziert Events, Events triggern Voices in Rust. Die Pattern-Eval-Frequenz ist niedrig (Events pro Sekunde, nicht Samples pro Sekunde).

### Source-of-Truth

TypeScript-Files im Projekt-Verzeichnis sind Source-of-Truth. Die UI ist ein Editor *über* TypeScript-Code. Es gibt kein proprietäres Binär-Format für Voices oder Patterns.

### Determinismus

Zwei Render-Pfade in `amigo-audio-core`:

- `render_deterministic` für CI/Pre-Render: plattform-übergreifend bit-identisch
- `render_realtime` für Runtime: plattform-spezifische Optimierungen erlaubt

Pre-Render-Output wird in CI via Hash-Test validiert.

## Implementierungs-Reihenfolge

Folge der Reihenfolge aus `_index.md`:

1. `amigo-audio-core` Basis (Sample-Player, Mixer)
2. `amigo-trommel-runtime` Voice-API
3. Voice-Modus Minimal-Version in Editor
4. DSP-Erweiterungen (Modulation, Filter, FX)
5. Voice-Modus Vollausbau
6. Strudel-Integration
7. Pattern-Modus (Grid + Code)
8. Audio-Analyse
9. `amigo-trommel-build` MVP
10. MIDI
11. Engine-Integration
12. Polishing

Jede dieser Phasen sollte zu einem lauffähigen Zustand führen. Keine wochenlange Refactoring-Phasen ohne sichtbares Ergebnis.

## Code-Konventionen

### Rust

- Edition 2024
- `cargo fmt` vor jedem Commit
- `cargo clippy` darf keine Warnings haben
- Tests in `tests/`, Benchmarks in `benches/`
- Determinismus-Tests laufen auf allen Plattformen in CI

### TypeScript

- Strict Mode aktiviert
- Keine `any`-Types ohne expliziten Kommentar warum
- ESLint mit Standard-Regeln
- Voice-Definitionen müssen ohne `unknown`-Casts durch das Type-System gehen

### Svelte

- Svelte 5 mit Runes (`$state`, `$derived`, `$effect`)
- Kein Svelte-3/4-Pattern (no `let`-reactivity, no `$:`)

## Testing-Strategie

### Unit-Tests

- Rust: pro DSP-Modul Unit-Tests mit bekannten Input/Output-Paaren
- TypeScript: Vitest für Voice/Pattern-Serialization, Type-Tests

### Integration-Tests

- End-to-End Voice → Pattern → Audio-Render in CI
- Cross-Plattform Determinismus-Tests (Linux + Windows + macOS in CI)

### Performance-Benchmarks

- Voice-Trigger-Latenz < 1ms
- Realtime-Render: 256 Voices parallel ohne Dropouts (44.1kHz, Buffer 128)
- Audio-Analyse: 30s Sample in < 1s

## Verbotene Ansätze

Folgende Patterns sind absichtlich nicht erlaubt:

- **Synthese-from-Scratch Voices** (Phase 1): Voices basieren auf Samples. Oscillator-Sources sind Phase 2.
- **Eigene Pattern-Sprache:** Strudel wird direkt verwendet. Keine eigene Sprach-Implementation.
- **Tracker-UI:** Wurde bewusst weggelassen. Pattern-Modus hat Grid-View und Code-View, kein Tracker-Grid.
- **Adaptive-Music-Editor:** Adaptive Music lebt in der amigo-engine, nicht in Trommel.
- **VST/AU-Plugin-Hosting:** Bricht Source-of-Truth-Modell.
- **Browser-Storage für persistenten State:** `.trommel`-Projekte sind File-System-basiert, nicht IndexedDB.
- **Bidirektionale Tracker-Pattern-Code-Sync:** Wurde diskutiert und verworfen. Grid und Code sind zwei Sichten auf *Strudel-Code*, nicht zwei separate Daten-Modelle.

## Phasen-Disziplin

Phase 1 ist explizit definiert. Wenn du beim Implementieren auf ein Feature stößt, das in „Phase 2 — Erweiterungen" steht, **baue es nicht ein**. Erstelle stattdessen ein Issue oder TODO und arbeite an Phase-1-Scope weiter.

Versuchung wird groß sein, „nur kurz" Oscillator-Sources oder ML-Drum-Transcription einzubauen. Tu's nicht. Phase 1 muss zuerst rund werden.

## Verzeichnis-Struktur

Monorepo unter `amigo-labs/amigo-trommel`:

```
amigo-trommel/
├── crates/
│   └── amigo-audio-core/
├── packages/
│   ├── amigo-trommel-runtime/
│   └── amigo-trommel-build/
├── apps/
│   └── amigo-trommel/             # Svelte + Tauri Editor
├── docs/
│   └── specs/audio/
├── CLAUDE.md                       # diese Datei
├── README.md
└── Cargo.toml                      # Workspace
```

## Voraussetzungen für Implementations-Start

**Pincel muss zu v1.0 sein**, bevor Trommel-Implementation startet. Trommel ist nicht parallel zu Pincel-Implementation zu bauen — Fokus-Risiko.

## Spec-Änderungen während Implementation

Wenn du beim Implementieren merkst, dass eine Spec lückenhaft, widersprüchlich oder unklar ist:

1. **Implementiere nicht ad-hoc.** Stoppe, dokumentiere die Frage.
2. **Schlage Spec-Änderung vor:** Konkrete Diff-artige Beschreibung, was geändert werden soll und warum.
3. **Erst nach Bestätigung implementieren.**

Specs sind die Single-Source-of-Truth des Designs. Code soll Specs entsprechen, nicht umgekehrt.
