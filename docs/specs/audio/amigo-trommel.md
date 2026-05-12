---
title: amigo-trommel
status: draft
phase: 1
last_updated: 2026-05-12
---

# amigo-trommel

Game-Audio-Authoring-Tool mit TypeScript als Source-of-Truth. Sample-basiertes Voice-System, Strudel-Live-Coding für Patterns, Engine-Integration über die Amigo Engine.

## Vision

amigo-trommel ist ein All-in-One-Tool für Game-Audio-Authoring. Es vereint Sample-basierte Voice-Bearbeitung, klickbare Drum-Patterns und Strudel-basiertes Live-Coding in einer einzigen App. Alle Voices, Patterns und Konfigurationen sind TypeScript-Code — diff-bar, type-safe, im JS-Ecosystem natürlich teilbar.

Das Tool läuft als PWA im Browser und als Desktop-App via Tauri v2. Es produziert `.trommel`-Bundles, die direkt von der Amigo Engine konsumiert werden — entweder als pre-rendered Audio oder als Live-Synthese mit Runtime-Modulation.

## Positionierung

**Was Trommel ist:**

Ein TypeScript-natives Game-Audio-Authoring-Tool für Sample-basierte Voices mit Strudel-Pattern-System und Engine-Integration.

**Was Trommel nicht ist:**

- Kein Synthesizer-from-Scratch — Voices basieren auf Samples, nicht auf Oszillator-Synthese (Phase 1)
- Keine DAW im Ableton-/Reaper-Sinn — kein Multi-Track-Recording, kein linearer Song-Editor
- Kein Tracker — Tracker-Paradigma wurde bewusst weggelassen
- Kein Adaptive-Music-Editor — Adaptive-Logik lebt in der Amigo Engine, Trommel liefert die Bausteine

## Stack

- **Frontend:** Svelte 5
- **Audio-Core:** Rust, kompiliert nach WASM (Web) und nativ (Tauri)
- **Audio-Output Web:** AudioWorklet
- **Audio-Output Native:** cpal über Tauri-Bridge
- **TypeScript-Kompilierung:** SWC
- **JS-Runtime Web:** Browser-JS (Strudel läuft nativ)
- **JS-Runtime Native:** QuickJS-NG via rquickjs (für Tauri und Engine-Integration)
- **Desktop-Wrapper:** Tauri v2
- **Build-Tool:** Vite

## Architektur

### Sub-Komponenten

```
amigo-trommel/
├── crates/
│   └── amigo-audio-core/          # Rust-DSP-Crate
├── packages/
│   ├── amigo-trommel-runtime/     # JS-Layer (Strudel-Integration)
│   └── amigo-trommel-build/       # Build-Tool (CLI)
├── apps/
│   └── amigo-trommel/             # Editor-App (Svelte + Tauri)
└── docs/specs/audio/              # Diese Specs
```

**`amigo-audio-core`** (Rust-Crate):
- Sample-Player mit Pitch-Shifting, Time-Stretching
- Modulation (Envelopes, LFOs)
- Filter (Low/High/Band-Pass mit Resonance)
- Internal FX (Distortion, Delay, Chorus, etc.)
- Mixer mit Bus-Routing
- Audio-Analyse (Beat-Detection, Onset-Detection, BPM)
- Kompiliert nach WASM (Web) und nativen Plattform-Targets (Desktop)

**`amigo-trommel-runtime`** (TypeScript/JavaScript):
- Strudel-Wrapper mit Trommel-Voice-Output-Target
- Voice-Trigger-API: `voice.trigger({ pitch, velocity, ...params })`
- Pattern-Eval-Schicht (browser-nativ oder QuickJS)
- Type-Definitionen für Voice/Pattern/Effect

**`amigo-trommel-build`** (CLI in Rust):
- TypeScript → JavaScript via SWC
- Statische Analyse: welche Voices/Patterns sind referenziert
- Voice-Definitionen → serialisierte DSP-Graphen (JSON/binary)
- Patterns: Pre-Render oder als JS für Live-Eval
- Output: `.trommel`-Bundle für Engine-Konsum

**`amigo-trommel`** (Editor-App):
- Svelte-5-UI mit Voice-Modus und Pattern-Modus
- Web-Build als PWA
- Desktop-Build via Tauri v2

### Datenfluss

```
TypeScript-Code (vom Sound-Designer geschrieben)
    ↓
SWC-Transpilation zu JS
    ↓
Web: Browser-JS    │    Native: QuickJS-NG via rquickjs
    ↓                     ↓
Strudel-Pattern-Eval (Control-Thread)
    ↓
Voice-Trigger-Events
    ↓
amigo-audio-core (Rust)
    ↓
Audio-Output (AudioWorklet auf Web, cpal auf Native)
```

**Wichtig:** Audio-DSP läuft *immer* in Rust, niemals in JS. JS-Layer beschreibt nur *was* passieren soll (welche Voice, mit welchen Parametern, wann), nicht *wie* (DSP-Berechnungen). Dies entspricht dem Pattern aus `amigo-downloader` (TS-Plugin sagt was, Rust-Core macht wie).

## Workflow

amigo-trommel hat zwei konzeptionelle Modi mit drei UI-Bereichen, die einer natürlichen Workflow-Reihenfolge folgen:

### A — Voice-Modus (Quelle des Materials)

Voices werden hier erstellt und editiert. Audio-Samples importieren, mit Modulation/Filter/FX-Chain anreichern, als TypeScript-Voice-Definition speichern.

Output: Eine Voice-Bibliothek (TS-Files im Projekt).

### B — Pattern-Modus / Grid-View (klickbare Patterns)

Voices aus A werden in einem Step-Grid zu Patterns zusammengeklickt. Im Hintergrund wird automatisch Strudel-Code generiert.

Output: Strudel-Pattern (gespeichert als TS-Code).

### C — Pattern-Modus / Code-View (textbasierte Patterns)

Strudel-Code wird direkt editiert. Volle Ausdruckskraft der Sprache verfügbar — Transformationen, Polyrhythmik, Generative Logik.

Output: Strudel-Pattern (selbe Datenquelle wie B).

### Wechselbeziehung zwischen Grid- und Code-View

Grid und Code sind zwei Sichten auf dieselbe Datenquelle (Strudel-Pattern). Edits in einer Sicht propagieren in die andere.

**Wichtige Einschränkung:** Wenn der Code Strudel-Features enthält, die nicht im Grid darstellbar sind (z. B. `.every(n, ...)`, `.degrade()`, Polymetrie), deaktiviert sich die Grid-View für den betroffenen Track. Die UI zeigt: „Dieser Track enthält Strudel-Features, die nicht im Grid darstellbar sind. Nur Code-Editing möglich." Andere Tracks im selben Pattern bleiben in Grid-View nutzbar, wenn sie grid-kompatibel sind.

**Grid-kompatible Strudel-Features:**

- Einfache Step-Notation: `s("bd ~ sd ~")`, `s("bd*4")`, `s("bd!2")`
- Verschachtelte Step-Gruppen: `s("[bd bd] sd")`
- Velocity per Step: `.gain(0.5)` auf einzelnen Steps
- Probability per Step: `s("bd?")`

**Nicht grid-kompatibel:**

- Transformationen über Cycles: `.every(n, ...)`, `.slow(n)`, `.fast(n)` mit n ≠ Cycle-Länge
- Generative Funktionen: `.degrade()`, `.sometimes(...)`, `.choose(...)`
- Polymetrik mit unterschiedlich langen Sub-Patterns
- Verkettungen mehrerer Operationen, die die Struktur verändern

## Source-of-Truth-Modell

### TypeScript als primäres Format

Alle Voices, Patterns und Konfigurationen sind TypeScript-Code im Projekt. Beispiele:

**Voice (`voices/kick.ts`):**

```typescript
import { defineVoice, sample, envelope, filter } from '@amigo/trommel';

export const kick = defineVoice({
  source: sample({ file: '../samples/kick-808.wav' }),
  modulation: [
    envelope({ target: 'amp', attack: 0, decay: 0.3, sustain: 0, release: 0.1 })
  ],
  filter: filter({ type: 'lowpass', cutoff: 800, resonance: 0.3 }),
  externalParams: ['pitch', 'punch']
});
```

**Pattern (`patterns/combat-beat.ts`):**

```typescript
import { stack, s, note } from '@strudel/core';
import { kick, snare, hihat } from '../voices/drums';

export default stack(
  s("bd*4").gain(0.8),
  s("~ sd ~ sd"),
  s("hh*8")
);
```

### Projekt-Struktur

```
my-game-audio.trommel/
├── trommel.config.ts        # Projekt-Config
├── voices/
│   ├── drums/
│   │   ├── kick.ts
│   │   ├── snare.ts
│   │   └── hihat.ts
│   └── sfx/
│       ├── laser.ts
│       └── coin-pickup.ts
├── samples/                 # Importierte Audio-Files
│   ├── drums/
│   │   └── kick-808.wav
│   └── foley/
├── patterns/
│   ├── combat-beat.ts
│   └── intro-loop.ts
└── package.json             # npm-Package-Struktur
```

### Warum TypeScript

- **Git-diff-bar:** Voices und Patterns als Code, klare Diffs in Git
- **Type-Safety:** TypeScript-Compiler erkennt fehlende Voices, falsche Parameter-Namen, etc.
- **IDE-Support:** Autocompletion, Refactoring, Find-References funktionieren out-of-the-box
- **npm-Ecosystem:** Voice-/Pattern-Packs als npm-Packages teilbar (`@amigo-labs/sfx-pack-retro`)
- **Refactoring-tauglich:** Voice umbenennen aktualisiert alle Patterns automatisch (via TS-Language-Server)

## Audacity-Einflüsse

Audacity wird als UX-Vorbild für mehrere Aspekte verwendet:

- **Waveform- und Spektrogramm-View** nebeneinander oder umschaltbar im Voice-Modus
- **Labels** als annotative Layer über Audio (Loop-Points, Attack/Release-Marker, Cue-Punkte)
- **Selection-basiertes Editing** — Bereich auf der Waveform markieren, Effekte anwenden
- **Effekt-Chain mit Live-Preview** — Effekte als Stack, Preview während Parameter-Tweaking
- **Multi-Track-View** im Pattern-Modus (jede Voice/Channel als eigener Track)
- **Solo/Mute pro Track** im Pattern-Modus

Audacity-Features, die *nicht* übernommen werden:

- Datei-basiertes Denken (Trommel ist projekt-basiert)
- Destruktives Editing (Voice-Edits sind immer non-destruktiv über die TS-Definition)
- Generische Plugin-Architektur (LADSPA/VST) — eigene Effect-Chain mit eingebautem Set

## Determinismus

**Pre-Render-Pfad (CI/Build):**

Plattform-übergreifend deterministisch. CI in Linux produziert bit-identisches WAV-Output wie lokales Render in Windows. Wichtig für reproducible builds und Hash-basierte Asset-Validierung.

Implementierung: Single-Threaded DSP-Pfad mit deterministischer Floating-Point-Reihenfolge. Keine plattform-spezifischen Optimierungen.

**Live-Synthese-Pfad (Runtime):**

Plattform-spezifische Optimierungen erlaubt. SIMD-Pfade, plattform-spezifisches FFT, parallel processing. Sub-Millisekunden-Unterschiede zwischen Plattformen sind akzeptabel und nicht hörbar.

**Konsequenz:** `amigo-audio-core` hat zwei Code-Pfade — `render_deterministic()` (CI, einfach, langsamer) und `render_realtime()` (Runtime, optimiert).

## MIDI-Integration

Vollständige MIDI-Unterstützung:

- **MIDI-In:** Voices im Voice-Modus mit externem Keyboard antesten, Drum-Pads im Pattern-Modus
- **MIDI-Out:** Externe Synths/DAWs ansteuern
- **MIDI-Export:** Patterns als `.mid`-Datei für DAW-Kompatibilität
- **MIDI-Import:** `.mid`-Datei als Drum-Pattern oder Strudel-Pattern importieren

Implementation:
- Native: `midir`-Crate in Rust
- Web: Web MIDI API direkt im Browser

## Audio-Analyse (Phase 1)

Klassische DSP-basierte Analyse für Sample-Workflows. Keine ML, voll deterministisch in Rust.

**Features:**

- **Beat-Detection** — Tempo-/BPM-Schätzung via Autocorrelation auf Spectral Flux
- **Onset-Detection** — Zeitstempel einzelner Schläge im Sample
- **Loop-Point-Erkennung** — automatische Vorschläge für saubere Loop-Punkte
- **Slice-Tools** — Sample an Beat- oder Onset-Grenzen in einzelne Sub-Samples teilen
- **Stille-Erkennung** — Auto-Trim von Anfang/Ende

**Details:** Siehe `audio-analysis.md`.

## Phase-Definition

### Phase 1 (Initiale Implementation)

Komplettes Tool mit Voice- und Pattern-Modus, Engine-Export, klassische Audio-Analyse, MIDI, Web+Desktop.

**Scope:**
- Voice-Modus mit Sample-Import, Modulation/Filter/FX-Chain
- Pattern-Modus mit Grid-View und Code-View
- Strudel-Integration über `amigo-trommel-runtime`
- `amigo-trommel-build` CLI mit Pre-Rendering
- Engine-Export als `.trommel`-Bundle
- Klassische DSP-Audio-Analyse
- MIDI vollständig
- Plattformen: Web (PWA) + Desktop (Tauri v2)
- Audacity-Style UX

**Voraussetzung:** Pincel zu v1.0 bringen, bevor Trommel-Implementation startet.

### Phase 2 (Erweiterungen, separate Specs)

- Browser-only ML-Drum-Transcription via ONNX Runtime Web
- Legacy-Importer (Sfxr-Familie, SoundFont)
- Freier Voice-Graph (modular statt fixes Skelett)
- Mobile-Apps (iOS/Android via Tauri)
- AI-Integration (Voice-Generation, Pattern-Suggestion)
- Community-Voice-/Pattern-Library
- Embed-Player für Web-Use-Cases
- Adaptive Music in amigo-engine (separate Engine-Audio-Spec, nicht Teil von Trommel)

## Engine-Integration

amigo-trommel produziert `.trommel`-Bundles, die von der Amigo Engine konsumiert werden.

### Bundle-Struktur

```
my-game-audio.trommel-bundle/
├── manifest.json            # Asset-Manifest
├── voices/                  # Voice-Definitionen (JSON)
│   ├── kick.json
│   └── laser.json
├── samples/                 # Referenzierte Sample-Files
│   └── kick-808.wav
└── patterns/
    ├── static/              # Pre-rendered Audio
    │   └── intro.ogg
    └── dynamic/             # JS-Code für Live-Eval
        └── combat.js
```

### Engine-API (Beispiel)

```rust
// Bundle laden
let audio = AudioBundle::load("game-audio.trommel-bundle")?;

// Voice triggern
audio.play("kick");

// Voice mit Parameter triggern
audio.play_with("footstep", &[
    ("surface", Surface::Grass.into()),
    ("velocity", 0.7),
]);

// Pattern starten
audio.start_pattern("combat-beat");
audio.stop_pattern("combat-beat");
```

**Details:** Erweiterung der amigo-engine-Audio-Spec (separate Spec-Datei in der Engine).

## Spec-Verzeichnis-Übersicht

Diese Hauptspec wird ergänzt durch:

- `amigo-trommel/voice-mode.md` — Voice-Modus im Detail
- `amigo-trommel/pattern-mode.md` — Pattern-Modus (Grid + Code) im Detail
- `amigo-trommel/audio-analysis.md` — Klassische DSP-Audio-Analyse
- `amigo-audio-core.md` — Rust-Crate-Spec
- `amigo-trommel-runtime.md` — JS-Runtime-Layer-Spec
- `amigo-trommel-build.md` — Build-Tool-Spec
- `import-export-formats.md` — vollständige Format-Spec
