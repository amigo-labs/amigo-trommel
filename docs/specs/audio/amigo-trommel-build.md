---
title: amigo-trommel-build
status: draft
phase: 1
parent: amigo-trommel.md
last_updated: 2026-05-12
---

# amigo-trommel-build

CLI-Tool für die Build-Pipeline von Trommel-Projekten zu Engine-konsumierbaren `.trommel`-Bundles.

## Zweck

`amigo-trommel-build` verarbeitet ein Trommel-Projekt-Verzeichnis und erzeugt:

- Kompilierte JS aus TypeScript-Quellen
- Serialisierte Voice-Definitionen
- Pre-rendered Audio für statische Patterns
- Dynamische JS-Module für Live-Eval-Patterns
- Asset-Manifest mit Metadata
- Validierungs-Reports

Das Output-`.trommel`-Bundle wird von der `amigo-engine` zur Laufzeit konsumiert.

## CLI-Interface

```bash
amigo-trommel-build [options] <project-path>

Commands:
  build              Komplettes Bundle bauen (default)
  watch              Watch-Mode, rebuild bei Änderungen
  validate           Nur validieren, kein Output
  render <pattern>   Einzelnes Pattern zu Audio rendern
  
Options:
  -o, --output <path>          Output-Pfad (default: ./dist)
  --target <web|engine|both>   Target-Plattform (default: both)
  --pre-render-all             Alle Patterns pre-rendern (kein dynamic)
  --pre-render-none            Keine Patterns pre-rendern (alles dynamic)
  --audio-format <ogg|opus|wav> Audio-Format für pre-rendered (default: opus)
  --verbose                    Verbose Logging
  --strict                     Bei jeglicher Warnung abbrechen
```

## Projekt-Struktur (Input)

```
my-game-audio.trommel/
├── trommel.config.ts        # Konfiguration
├── package.json
├── tsconfig.json
├── voices/
│   ├── drums/
│   │   ├── kick.ts
│   │   ├── snare.ts
│   │   └── index.ts         # Re-Exports
│   └── sfx/
│       ├── laser.ts
│       └── coin.ts
├── samples/                 # Audio-Source-Files
│   ├── drums/
│   └── foley/
└── patterns/
    ├── combat.ts
    ├── intro.ts
    └── outro.ts
```

## Bundle-Struktur (Output)

```
dist/my-game-audio.trommel-bundle/
├── manifest.json            # Bundle-Metadata
├── voices/
│   ├── kick.json            # Serialisierte VoiceDefinition
│   ├── snare.json
│   └── laser.json
├── samples/                 # Kopierte/konvertierte Sample-Files
│   ├── kick-808.opus
│   └── snare.opus
├── patterns/
│   ├── static/
│   │   ├── intro.opus       # Pre-rendered Audio
│   │   └── outro.opus
│   └── dynamic/
│       ├── combat.js        # Live-Eval JS
│       └── combat.json      # Metadata (referenzierte Voices, etc.)
└── debug/                   # Optional, mit --verbose
    ├── compile-report.json
    └── analysis-report.json
```

## Konfiguration

### trommel.config.ts

```typescript
import { defineConfig } from '@amigo/trommel/build';

export default defineConfig({
  name: 'my-game-audio',
  version: '1.0.0',
  
  // Audio-Format-Settings
  audio: {
    outputFormat: 'opus',
    sampleRate: 48000,
    bitrate: 96,              // kbit/s für lossy formats
    channels: 2
  },
  
  // Pattern-Build-Strategie
  patterns: {
    // Per-Pattern Override
    strategy: {
      'intro': 'static',       // pre-render
      'combat': 'dynamic',     // live-eval
      'outro': 'static'
    },
    defaultStrategy: 'auto'    // auto-detect basierend auf statischer Analyse
  },
  
  // Engine-Targets
  targets: ['web', 'engine'],
  
  // Asset-Optimierungen
  optimize: {
    deduplicateSamples: true,
    compressSamples: true,
    stripUnusedVoices: true
  }
});
```

## Build-Pipeline

### Phase 1: Validation

1. **TypeScript-Compile** mit `tsc --noEmit` als Type-Check
2. **Schema-Validation** für Voice/Pattern-Definitionen
3. **Reference-Check:** Alle Sample-Pfade existieren, alle Voice-Imports in Patterns auflösbar
4. **Dead-Code-Detection:** Voices, die nirgends verwendet werden (Warning, mit `--strict` Error)

### Phase 2: Voice-Compilation

Für jede Voice-Definition:

1. TypeScript via SWC zu JS kompilieren
2. JS in einem Sandbox-Context ausführen (`defineVoice(...)` aufrufen)
3. Resultat-Objekt zu JSON serialisieren
4. JSON in `dist/voices/<voice-id>.json` speichern

Wichtig: `defineVoice()` ist eine pure Funktion. Aufruf produziert immer dieselbe Struktur. Deshalb deterministisch buildbar.

### Phase 3: Sample-Processing

Für jedes referenzierte Sample:

1. Sample dekodieren (via `symphonia`)
2. Optional: Sample-Rate-Konversion auf Ziel-Rate
3. Optional: Format-Konversion (zu Opus/OGG/WAV)
4. In `dist/samples/<sample-name>.<format>` speichern
5. Pfad-Mapping im Manifest aktualisieren

Sample-Deduplication: Wenn mehrere Voices dasselbe Sample referenzieren, wird es nur einmal kopiert.

### Phase 4: Pattern-Compilation

Für jedes Pattern entscheidet der Builder die Strategie:

**Static-Strategie (Pre-Rendering):**

Pattern wird im Builder evaluiert und zu Audio gerendert.

Voraussetzung: Pattern hat keine externalen Parameter, keine Variation, kein Random.

```typescript
// Builder evaluiert das Pattern in einer Sandbox
const pattern = require('./patterns/intro.ts').default;
const events = strudelToEventStream(pattern, { tempo: 120, duration: 8 });

// Events durch amigo-audio-core rendern
const audioBuffer = renderEvents(events, voices, samples);

// Als Opus/OGG speichern
encodeOpus(audioBuffer, 'dist/patterns/static/intro.opus');
```

**Dynamic-Strategie (Live-Eval):**

Pattern bleibt als JS-Code, wird zur Laufzeit evaluiert.

1. TypeScript via SWC zu JS kompilieren
2. JS-Datei in `dist/patterns/dynamic/<pattern-name>.js` speichern
3. Metadata-JSON mit referenzierten Voices und Parametern erzeugen

**Auto-Strategie:**

Builder analysiert das Pattern statisch:

- Wenn nur statische Strudel-Operationen → Static
- Wenn externale Parameter, Random, oder Custom-JS-Logik → Dynamic
- Im Zweifel: Dynamic (sicherer)

### Phase 5: Manifest-Generation

Asset-Manifest beschreibt alle Bundle-Inhalte:

```json
{
  "name": "my-game-audio",
  "version": "1.0.0",
  "format_version": "1.0",
  "voices": {
    "kick": {
      "path": "voices/kick.json",
      "samples": ["samples/kick-808.opus"],
      "external_params": ["pitch", "punch"]
    }
  },
  "patterns": {
    "intro": {
      "type": "static",
      "path": "patterns/static/intro.opus",
      "duration": 8.0
    },
    "combat": {
      "type": "dynamic",
      "path": "patterns/dynamic/combat.js",
      "referenced_voices": ["kick", "snare", "hihat"],
      "external_params": ["intensity"]
    }
  },
  "samples": {
    "kick-808.opus": {
      "format": "opus",
      "sample_rate": 48000,
      "channels": 2,
      "duration": 1.2
    }
  }
}
```

### Phase 6: Validation des Bundles

Final-Check vor Output:

- Alle referenzierten Files existieren im Bundle
- Manifest ist konsistent
- Bundle-Größe wird gemeldet
- Pre-rendered Audio wird hash-validated

## Pre-Rendering im Detail

### Wie rendert der Builder ein Pattern?

Pattern-Pre-Rendering nutzt `amigo-audio-core` im `render_deterministic`-Modus:

```rust
// Pseudo-Code
fn render_pattern(
    pattern_events: Vec<Event>,
    voices: HashMap<String, VoiceDefinition>,
    samples: HashMap<String, Sample>,
    duration: f32,
    sample_rate: u32,
) -> Vec<f32> {
    let mut engine = AudioEngine::new_deterministic(sample_rate);
    
    for (id, def) in voices {
        engine.load_voice(id, def);
    }
    
    let mut output = vec![0.0; (duration * sample_rate as f32) as usize * 2];
    let mut event_iter = pattern_events.iter();
    let mut next_event = event_iter.next();
    
    for (i, sample) in output.chunks_mut(2).enumerate() {
        let time = i as f32 / sample_rate as f32;
        
        // Trigger Events at this time
        while let Some(event) = next_event {
            if event.time <= time {
                engine.trigger(&event.voice, event.params);
                next_event = event_iter.next();
            } else {
                break;
            }
        }
        
        engine.process(sample);
    }
    
    output
}
```

### Determinismus

Pre-Rendering muss plattform-übergreifend bit-identisch sein:

- CI auf Linux produziert identische Opus/WAV wie lokales Build auf Windows
- Hash-Validation in CI: erwarteter Hash pro Pattern wird mit tatsächlichem verglichen

Test-Setup:

```
tests/
├── render_determinism/
│   ├── kick_pattern.ts          # Test-Pattern
│   ├── expected_hash.txt        # SHA-256 des erwarteten Outputs
│   └── test_render.rs           # Test, der über alle Plattformen läuft
```

## Watch-Mode

```bash
amigo-trommel-build watch ./
```

- File-Watcher über `voices/`, `patterns/`, `samples/`, `trommel.config.ts`
- Inkrementelles Rebuild: nur geänderte Voices/Patterns werden neu kompiliert
- Sample-Reuse: Samples nur neu kopieren bei Änderung
- HMR-Support für Trommel-Editor-App (siehe `amigo-trommel.md`)

## Render-Modus

Einzelnes Pattern zu Audio rendern für Vorschau:

```bash
amigo-trommel-build render combat-beat --output preview.opus
```

Nützlich für:
- Schnelle Vorschau ohne UI
- Asset-Validierung in CI
- Export für DAW-Integration

## Validation-Modus

```bash
amigo-trommel-build validate ./
```

Prüft alles ohne Output:

- TypeScript-Typen
- Voice/Pattern-Schema
- Asset-Referenzen
- Sample-Files existieren und sind dekodierbar
- Voice-Pool-Konsistenz (Patterns referenzieren existierende Voices)

Exit-Code 0 bei Erfolg, !=0 bei Fehlern. Nutzbar in CI als Pre-Commit-Check.

## Performance

### Ziele

- **Inkrementelles Rebuild** für eine geänderte Voice: < 500ms
- **Komplettes Rebuild** für mittleres Projekt (50 Voices, 20 Patterns, 100MB Samples): < 30s
- **Pre-Render** für 60s Pattern auf modernem Hardware: < 5s

### Strategien

- **Parallel Sample-Processing** (Sample-Encoding ist embarrassingly parallel)
- **Cached Compilation:** Voice-JSON-Output wird gecached, nur bei Source-Änderung neu erzeugt
- **Parallel Pattern-Rendering:** mehrere Patterns gleichzeitig rendern

## Implementation

### Rust + Embedded JS Runtime

Build-Tool ist in Rust geschrieben für Performance und einfache Distribution (Single-Binary). TypeScript-Kompilation via `swc_core` (SWC als Rust-Bibliothek).

JS-Evaluation für `defineVoice()` und `definePattern()`: QuickJS-NG via `rquickjs`.

Pattern-Pre-Rendering: Direkter Aufruf von `amigo-audio-core` mit `render_deterministic`.

### Crate-Struktur

```
amigo-trommel-build/
├── src/
│   ├── main.rs               # CLI-Entry
│   ├── config.rs             # trommel.config.ts Parser
│   ├── compile/
│   │   ├── typescript.rs     # SWC-Wrapper
│   │   ├── voice.rs          # Voice-Compilation
│   │   └── pattern.rs        # Pattern-Compilation
│   ├── render/
│   │   ├── strudel.rs        # Strudel-Pattern-Evaluation
│   │   └── audio.rs          # Audio-Rendering via amigo-audio-core
│   ├── bundle/
│   │   ├── manifest.rs       # Manifest-Generation
│   │   └── package.rs        # Bundle-Packaging
│   ├── watch.rs              # Watch-Mode
│   └── validate.rs           # Validation
├── Cargo.toml
└── README.md
```

### Dependencies

- `swc_core` — TypeScript-Kompilation
- `rquickjs` — JS-Evaluation
- `amigo-audio-core` — Audio-Engine
- `serde`/`serde_json` — Serialization
- `notify` — File-Watching
- `clap` — CLI-Argumente
- `rayon` — Parallelisierung

## Engine-Konsum

Die `amigo-engine` lädt das Bundle:

```rust
use amigo_audio::Bundle;

let bundle = Bundle::load("game-audio.trommel-bundle")?;

// Statische Voices und Samples werden direkt verfügbar
let kick = bundle.voice("kick")?;

// Dynamic-Patterns brauchen QuickJS-Runtime
let mut pattern_engine = PatternEngine::new();
pattern_engine.load_bundle(&bundle)?;
pattern_engine.start_pattern("combat", &[("intensity", 0.5.into())])?;
```

Details in der amigo-engine-Audio-Spec.

## Implementierungs-Reihenfolge

Für Phase-1-Implementation:

1. CLI-Setup, Argument-Parsing
2. Config-Loading aus `trommel.config.ts`
3. TypeScript-Compilation via swc_core
4. Voice-Evaluation via rquickjs
5. Voice-JSON-Output
6. Sample-Discovery und -Copying
7. Sample-Format-Conversion (Opus-Encoding)
8. Sample-Deduplication
9. Pattern-Compilation: Dynamic-Strategie (JS-Output)
10. Pattern-Compilation: Static-Strategie (Pre-Rendering via amigo-audio-core)
11. Auto-Strategy-Detection
12. Manifest-Generation
13. Bundle-Validation
14. Watch-Mode mit inkrementellem Rebuild
15. Render-Modus für Einzel-Pattern
16. Validate-Modus
17. Determinismus-Tests in CI
