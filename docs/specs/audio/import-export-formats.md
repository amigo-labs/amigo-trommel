---
title: amigo-trommel — Import/Export-Formate
status: draft
phase: 1
parent: amigo-trommel.md
last_updated: 2026-05-12
---

# Import/Export-Formate

Spezifikation aller unterstützten Import- und Export-Formate in amigo-trommel.

## Übersicht

| Kategorie | Import | Export |
|---|---|---|
| **Audio lossless** | WAV, FLAC, AIFF | WAV, FLAC |
| **Audio lossy** | OGG Vorbis, Opus, MP3 | OGG Vorbis, Opus, MP3 |
| **Eigenes Format** | `.trommel`, `.ts` Voices, npm | `.trommel`, `.ts` Voices, npm-Package |
| **Pattern-Daten** | MIDI, Strudel-Code | MIDI, Strudel-Code, JSON-Event-Stream |
| **Engine** | — | `.trommel`-Bundle, Stem-Bundle |
| **Legacy (Phase 2)** | `.sfxr`, `.bfxrsound`, `.sf2`, DecentSampler | — |
| **Visualisierungen** | — | PNG/SVG (Waveform, Spektrogramm, Pattern) |

## Audio-Import

### Unterstützte Formate

Implementation über `symphonia`-Crate in `amigo-audio-core`.

| Format | Extension | Typ | Notes |
|---|---|---|---|
| WAV | `.wav` | PCM | Alle Bit-Tiefen (8/16/24/32), alle Sample-Rates |
| FLAC | `.flac` | Lossless | Komprimiert, identisch zu WAV qualitativ |
| AIFF | `.aif`, `.aiff` | PCM | Mac-Welt, äquivalent zu WAV |
| OGG Vorbis | `.ogg` | Lossy | Open-Source, gut für Web |
| Opus | `.opus` | Lossy | Modern, exzellente Quality-pro-Byte |
| MP3 | `.mp3` | Lossy | Sehr verbreitet |

### Import-Workflow

Im Voice-Modus:

1. **Datei-Auswahl** (Drag-and-Drop oder File-Picker)
2. **Format-Detection** automatisch über Datei-Header
3. **Decoding** in interne Float32-Repräsentation
4. **Sample-Rate-Konversion** auf Project-Rate (falls nötig, via `rubato`)
5. **Auto-Analyse** läuft im Background (siehe `audio-analysis.md`)
6. **Speicherung** in `samples/`-Verzeichnis des Projekts (optional konvertiert)

### Import-Optionen

```typescript
interface ImportOptions {
  targetFormat?: 'original' | 'wav' | 'flac' | 'opus';
  targetSampleRate?: number;
  normalize?: boolean;
  trimSilence?: boolean;
  monoMixdown?: boolean;
}
```

## Audio-Export

### Render-Modi

Audio kann aus mehreren Quellen exportiert werden:

- **Voice rendern:** Eine Voice mit definierten Parametern als Audio-File
- **Pattern rendern:** Ein Pattern für definierte Dauer als Audio-File
- **Selection rendern:** Bereich auf der Pattern-Waveform-View als Audio
- **Stems rendern:** Pattern als separate Track-Files (für DAW-Import)

### Export-Optionen

```typescript
interface AudioExportOptions {
  format: 'wav' | 'flac' | 'ogg' | 'opus' | 'mp3';
  sampleRate?: number;          // Default: Project-Rate
  bitDepth?: 16 | 24 | 32;       // für PCM-Formate
  bitrate?: number;              // kbit/s für Lossy
  channels?: 1 | 2;              // mono | stereo
  normalize?: boolean;
  fadeIn?: number;               // ms
  fadeOut?: number;
}
```

### Encoder-Implementation

| Format | Encoder | Notes |
|---|---|---|
| WAV | `hound` | Pure-Rust, deterministisch |
| FLAC | `flac-bound` oder eigene | Pure-Rust verfügbar |
| OGG Vorbis | `vorbis-rs` (FFI) oder pure-Rust | FFI für WASM kniffliger |
| Opus | `opus`-Crate | Bessere WASM-Story als Vorbis |
| MP3 | `mp3lame-encoder` | FFI, LGPL-Hinweis nötig |

**Empfehlung:** Opus als Default für Game-Audio (modernes Format, klein, gut). WAV als verlustloses Master. OGG/MP3 nur für Sharing-Use-Cases.

## Trommel-Projekt-Format

### `.trommel`-Verzeichnis

Ein Trommel-Projekt ist ein Verzeichnis mit definierter Struktur (siehe `amigo-trommel.md` für Details). Es ist kein „File-Format" im klassischen Sinn, sondern eine Verzeichnis-Konvention.

```
my-game-audio.trommel/
├── trommel.config.ts
├── package.json
├── voices/
├── samples/
└── patterns/
```

### Projekt-Import

- **Verzeichnis öffnen:** Trommel öffnet das Verzeichnis als Workspace
- **`.trommel`-Archive:** Optional als ZIP-Archive für einfaches Sharing (Phase 2)

### Projekt-Export

- **Aktuelle Workspace** ist immer das Source-of-Truth (kein expliziter „Export" nötig)
- **Optional: ZIP-Archive** für einfaches Sharing

## Voice-Import/Export

### Einzelne Voice-Files

Voice = `.ts`-File. Import = TypeScript-Modul-Import.

```typescript
// In einem Pattern:
import { kick } from '../voices/drums';
import { laser } from './my-imported-voice'; // aus anderem Projekt kopiert
import { snare } from '@amigo-labs/drum-kit-808'; // aus npm-Package
```

### Voice-Bundle

Für Sharing einer Voice plus referenzierter Samples:

```
my-voice-bundle/
├── voice.ts
└── samples/
    └── source-sample.wav
```

Trommel kann ein Bundle als ZIP exportieren und importieren.

### npm-Package-Export

Trommel kann eine Auswahl von Voices als npm-Package exportieren:

```bash
amigo-trommel-build export-npm \
  --voices "drums/*" \
  --package-name "@amigo-labs/drum-kit-808" \
  --output ./packages/drum-kit-808
```

Erzeugt:

```
packages/drum-kit-808/
├── package.json
├── tsconfig.json
├── src/
│   ├── index.ts
│   └── voices/
│       ├── kick.ts
│       └── snare.ts
└── samples/
    └── ...
```

## Pattern-Import/Export

### MIDI

#### MIDI-Import

`.mid`-File wird interpretiert als:

- **Tempo-Map** → Tempo-Setting des Patterns
- **Notes pro Channel/Track** → Voice-Triggers
- **Velocity** → Velocity in Voice-Trigger
- **CC-Daten** → optional Voice-Parameter-Modulation (manuell zuzuordnen)

Workflow:

1. MIDI-File auswählen
2. Tempo + Channels zeigen
3. Pro Channel eine Voice aus dem Voice-Pool zuordnen
4. Bei Drum-Channels (Channel 10): Standard-GM-Drum-Mapping vorschlagen
5. Pattern wird generiert (Grid-View, sofern grid-kompatibel)

#### MIDI-Export

Pattern wird zu MIDI-File:

- Tempo wird in Tempo-Map geschrieben
- Pro Track eine MIDI-Channel-Zuweisung
- Voice-Trigger werden zu Note-On/Note-Off-Events
- Velocity wird übernommen
- Externalized-Parameter werden als CC-Daten exportiert (optional)

### Strudel-Code

#### Strudel-Code-Import

Strudel-Code-Snippets können direkt eingefügt werden:

- Im Code-View einfach pasten
- Voice-References werden gegen Voice-Pool geprüft
- Fehlende Voices werden als Warning markiert

#### Strudel-Code-Export

Pattern wird als Standalone-`.ts`-File exportiert mit allen Imports:

```typescript
// Exported pattern, ready to use in another Trommel project
import { stack, s } from '@strudel/core';
// Voices benötigt: kick, snare, hihat

export default stack(
  s("bd*4"),
  s("~ sd ~ sd"),
  s("hh*8")
);
```

### JSON-Event-Stream

Patterns können als JSON-Event-Stream exportiert werden — alle Events mit absoluten Zeiten:

```json
{
  "tempo": 120,
  "duration": 4.0,
  "events": [
    { "time": 0.0, "voice": "kick", "params": { "velocity": 1.0 } },
    { "time": 0.5, "voice": "snare", "params": { "velocity": 0.9 } },
    { "time": 1.0, "voice": "kick", "params": { "velocity": 1.0 } },
    { "time": 1.5, "voice": "snare", "params": { "velocity": 0.9 } }
  ]
}
```

Nützlich für:

- Engine-Integration (statische Pattern-Vorberechnung)
- Custom-Tools, die Trommel-Patterns konsumieren
- Datenanalyse und Visualisierung

## Engine-Bundle-Export

### `.trommel`-Bundle

Output von `amigo-trommel-build`, konsumiert von `amigo-engine`. Details siehe `amigo-trommel-build.md`.

### Stem-Bundle

Alternative: Mehrere pre-rendered Audio-Files plus Manifest, für Engines/Spiele, die kein dynamic Audio brauchen:

```
my-game-audio-stems/
├── manifest.json
├── voices/
│   ├── kick.opus
│   ├── snare.opus
│   └── ...
└── patterns/
    ├── combat.opus
    └── intro.opus
```

Manifest:

```json
{
  "voices": {
    "kick": { "path": "voices/kick.opus", "polyphony": "mono" }
  },
  "patterns": {
    "combat": { "path": "patterns/combat.opus", "duration": 8.0, "loop": true }
  }
}
```

Engine spielt diese als einfache Audio-Files ab, ohne JS-Runtime.

## Visualisierungs-Export

### Waveform-Export

Eine Voice oder ein Pattern als Waveform-Visualisierung:

```typescript
exportWaveform({
  source: 'voice' | 'pattern' | 'selection',
  id: string,
  format: 'png' | 'svg',
  width: number,
  height: number,
  color?: string,
  backgroundColor?: string
});
```

### Spektrogramm-Export

```typescript
exportSpectrogram({
  source: 'voice' | 'pattern' | 'selection',
  id: string,
  format: 'png' | 'svg',
  width: number,
  height: number,
  fmin?: number,           // Hz, niedrigste Frequenz
  fmax?: number,           // Hz, höchste Frequenz
  colorMap?: string        // viridis, magma, plasma, ...
});
```

### Pattern-Plot-Export

Strudel-Pattern als Cycle-Visualisierung (Punkte/Linien):

```typescript
exportPatternPlot({
  patternId: string,
  format: 'png' | 'svg',
  width: number,
  height: number,
  style: 'rings' | 'linear' | 'grid'
});
```

## Phase 2 — Legacy-Importer

Phase 2 ergänzt um Importer für etablierte Tool-Formate:

### Sfxr-Familie

`.sfxr`-, `.bfxrsound`-, `.jsfxr`-Files werden zu Trommel-Voices konvertiert. Diese Formate sind klein und gut dokumentiert.

```bash
amigo-trommel-build import-sfxr ./my-old-sounds.sfxr --output voices/imported/
```

Resultat: Eine Trommel-Voice pro Sfxr-Sound, mit Synthese-Parametern als Synthesizer-Voice (sobald Phase 2 Oscillator-Sources unterstützt).

### SoundFont

`.sf2`-Files werden zu mehreren Voices (eine pro Instrument):

```bash
amigo-trommel-build import-sf2 ./my-soundfont.sf2 --output voices/imported/
```

Jede SoundFont-Instrument-Definition wird zu einer Trommel-Voice mit dem Sample-Set.

### DecentSampler

Modernerer Sampler-Standard. Konvertierung ähnlich zu SoundFont.

## Was bewusst weggelassen wird

### DAW-Project-Files

- Ableton `.als`, FL Studio `.flp`, Reaper `.rpp` — proprietär bzw. komplex
- MIDI-Export deckt DAW-Bridging ab

### Tracker-Formate

- `.mod`, `.xm`, `.it`, `.s3m` — Tracker-Paradigma wurde bewusst weggelassen
- Falls Sample-Material aus Trackern gewünscht: separate Konverter-Tools nutzen

### VST/AU/LADSPA-Plugins

- Bricht das Source-of-Truth-Modell
- Phase 2 evaluiert eventuell WebAssembly-Plugin-System

## Implementierungs-Reihenfolge

Für Phase-1-Implementation:

1. WAV-Import + Export (Basis-Format)
2. OGG Vorbis Import
3. FLAC Import
4. Opus Encoding für Export (primärer Game-Audio-Output)
5. MP3 Import (read-only initial)
6. OGG Vorbis Encoding
7. AIFF Import
8. MP3 Export
9. MIDI Import (mit Voice-Zuordnungs-Workflow)
10. MIDI Export
11. JSON-Event-Stream Export
12. Strudel-Code Export als Standalone-File
13. Voice-Bundle Export/Import (ZIP)
14. npm-Package Export
15. Waveform-PNG/SVG-Export
16. Spektrogramm-PNG/SVG-Export
17. Pattern-Plot-Export

Phase 2:

18. Sfxr-Familie Import
19. SoundFont Import
20. DecentSampler Import
