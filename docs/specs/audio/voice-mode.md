---
title: amigo-trommel — Voice-Modus
status: draft
phase: 1
parent: amigo-trommel.md
last_updated: 2026-05-12
---

# Voice-Modus

Der Voice-Modus ist die erste Stufe im Trommel-Workflow. Hier werden Voices erstellt, editiert und organisiert. Voices sind die Klangbausteine, die später im Pattern-Modus zu Patterns kombiniert werden.

## Voice-Konzept

Eine Voice ist eine TypeScript-Definition eines klangerzeugenden Bausteins:

```
[Sample] → [Modulation] → [Filter] → [FX-Chain] → [Output]
```

Eine Voice hat:

- Genau eine **Source** (Sample-File, später optional auch Oszillator)
- Beliebig viele **Modulation-Slots** (Envelopes, LFOs)
- Genau einen **Filter** (kann auf Bypass)
- Eine **FX-Chain** mit 0–N **Internal FX**
- Eine Liste von **Externalized Parameters** für Runtime-Modulation

## Voice als TypeScript

Voices werden als TypeScript-Module gespeichert. Jede Voice ist ein eigenes File unter `voices/`.

### Minimal-Beispiel

```typescript
import { defineVoice, sample } from '@amigo/trommel';

export const kick = defineVoice({
  source: sample({ file: '../samples/kick-808.wav' })
});
```

### Vollständiges Beispiel

```typescript
import {
  defineVoice,
  sample,
  envelope,
  lfo,
  filter,
  distortion,
  delay
} from '@amigo/trommel';

export const kick = defineVoice({
  source: sample({
    file: '../samples/kick-808.wav',
    rootNote: 'C2',           // für Pitch-Modulation
    loop: false
  }),
  modulation: [
    envelope({
      target: 'amp',
      attack: 0,
      decay: 0.3,
      sustain: 0,
      release: 0.1
    }),
    envelope({
      target: 'pitch',
      attack: 0,
      decay: 0.05,
      sustain: 0,
      release: 0,
      amount: 0.5            // halbe Oktave Pitch-Drop
    })
  ],
  filter: filter({
    type: 'lowpass',
    cutoff: 800,
    resonance: 0.3
  }),
  fx: [
    distortion({ drive: 0.2 }),
    delay({ time: 0.125, feedback: 0.3, mix: 0.2 })
  ],
  externalParams: ['pitch', 'punch'],
  polyphony: 'mono'          // mono | poly(n)
});
```

### Voice-Bibliothek

Voices können in npm-Packages organisiert sein. Importe funktionieren standardmäßig:

```typescript
import { kick, snare, hihat } from '@amigo-labs/drum-kit-808';
import { laser, coin } from './sfx-pack';
```

## Sample-Sources

### Unterstützte Formate

- WAV (PCM, alle Bit-Tiefen und Sample-Rates)
- FLAC (lossless, komprimiert)
- AIFF (Mac-Welt)
- OGG Vorbis (lossy)
- Opus (lossy, modern)
- MP3 (lossy)

Implementation in `amigo-audio-core` via `symphonia`-Crate.

### Sample-Parameter

```typescript
sample({
  file: string,              // Pfad relativ zur Voice-Datei
  rootNote?: NoteName,       // Default: 'C4'; Tonhöhe des Samples
  loop?: boolean,            // Default: false
  loopStart?: number,        // Sample-Frame oder Sekunden
  loopEnd?: number,
  startOffset?: number,      // Frames/Sekunden, ab denen abgespielt wird
  reverse?: boolean,         // Sample rückwärts abspielen
  detune?: number            // Cents (-100 bis +100 für Feintuning)
})
```

### Sample-Playback-Engine

Implementation in `amigo-audio-core`:

- **Pitch-Shifting** über Resampling (linear, hermite, sinc — konfigurierbare Qualität)
- **Loop-Support** mit Crossfade an Loop-Punkten (optional)
- **Reverse-Playback**
- **Sample-akkurates Timing**

## Modulation

Modulation-Slots verändern Voice-Parameter über die Zeit. Ein Slot hat ein **Target** (welcher Parameter wird moduliert) und eine **Source** (Envelope, LFO, etc.).

### Envelope (ADSR)

```typescript
envelope({
  target: 'amp' | 'pitch' | 'filter.cutoff' | 'filter.resonance' | string,
  attack: number,            // Sekunden
  decay: number,
  sustain: number,           // 0..1
  release: number,
  amount?: number,           // Default: 1; Skalierung des Envelopes
  curve?: 'linear' | 'exp' | 'log'  // Default: 'exp'
})
```

### LFO

```typescript
lfo({
  target: string,
  waveform: 'sine' | 'triangle' | 'square' | 'sawtooth' | 'random',
  rate: number,              // Hz
  amount: number,
  phase?: number,            // 0..1, Start-Phase
  sync?: 'free' | 'tempo'    // Default: 'free'; bei 'tempo' rate in Beats
})
```

### Modulation-Targets

Standard-Targets, die jede Voice unterstützt:

- `amp` — Amplitude (Lautstärke)
- `pitch` — Tonhöhe (in Halbtönen oder Cents)
- `filter.cutoff` — Filter-Cutoff-Frequenz
- `filter.resonance` — Filter-Resonance
- `pan` — Stereo-Panorama

Weitere Targets je nach FX:

- `fx.delay.time`, `fx.delay.feedback`, `fx.delay.mix`
- `fx.distortion.drive`
- etc.

## Filter

Jede Voice hat genau einen Filter-Slot (kann auf Bypass).

```typescript
filter({
  type: 'lowpass' | 'highpass' | 'bandpass' | 'notch' | 'bypass',
  cutoff: number,            // Hz
  resonance: number,         // 0..1
  drive?: number             // Default: 0; Saturation vor Filter
})
```

Implementation: State-Variable-Filter in `amigo-audio-core`, stabil bei Modulation.

## FX-Chain

Beliebig viele Internal FX werden seriell nach dem Filter angewendet.

### Verfügbare FX (Phase 1)

```typescript
distortion({
  type: 'soft' | 'hard' | 'foldback',
  drive: number,             // 0..1
  mix: number                // 0..1, Wet/Dry
})

delay({
  time: number,              // Sekunden oder Beat-Notation ('1/4', '1/8')
  feedback: number,          // 0..1
  mix: number,
  pingPong?: boolean
})

reverb({
  type: 'plate' | 'hall' | 'spring',
  size: number,              // 0..1
  decay: number,             // Sekunden
  mix: number
})

chorus({
  rate: number,              // Hz
  depth: number,
  mix: number
})

bitcrusher({
  bits: number,              // 1..16
  sampleRate: number,        // Hz, target Sample-Rate
  mix: number
})

eq3({
  low: number,               // dB, -24 bis +24
  mid: number,
  high: number,
  lowFreq?: number,          // Hz, Crossover
  highFreq?: number
})
```

### Internal FX vs. Bus FX

**Wichtige Unterscheidung:**

- **Internal FX** (Voice-Ebene): Teil des Voice-Sound-Designs. Wird mit der Voice transportiert. Egal wo die Voice gespielt wird, klingt sie so.
- **Bus FX** (Mixer-Ebene): Teil der Mischung eines bestimmten Songs/Setups. Nicht Teil der Voice.

Mentale Regel: „Soll der Effekt auch klingen, wenn ich diese Voice in einem anderen Song nutze?" → Internal FX. „Ist der Effekt Teil der Mischung dieses bestimmten Songs?" → Bus FX.

## Externalized Parameters

Parameter, die zur Laufzeit (in der Amigo Engine oder im Pattern-Modus) von außen gesetzt werden können.

```typescript
externalParams: ['pitch', 'velocity', 'surface', 'intensity']
```

Standard-Parameter, die jede Voice automatisch unterstützt:

- `pitch` — Tonhöhen-Modifikation (Halbtöne)
- `velocity` — Anschlagsstärke (0..1)
- `pan` — Stereo-Position (-1..+1)

Benutzerdefinierte Parameter müssen explizit deklariert werden:

```typescript
defineVoice({
  // ...
  externalParams: ['surface', 'intensity'],
  paramBindings: {
    'surface': {
      type: 'enum',
      values: ['grass', 'stone', 'water'],
      // Mapping zu internen Modulationen
      bindings: [
        { when: 'grass', set: { 'filter.cutoff': 2000, 'fx.reverb.mix': 0.1 } },
        { when: 'stone', set: { 'filter.cutoff': 4000, 'fx.reverb.mix': 0.3 } },
        { when: 'water', set: { 'filter.cutoff': 800, 'fx.reverb.mix': 0.6 } }
      ]
    },
    'intensity': {
      type: 'continuous',
      range: [0, 1],
      bindings: [
        { target: 'amp', map: 'linear' },
        { target: 'filter.cutoff', map: { from: [0, 1], to: [500, 5000] } }
      ]
    }
  }
});
```

## Polyphonie

```typescript
polyphony: 'mono' | { poly: number, stealing: 'oldest' | 'quietest' | 'similar' }
```

**Mono** (Default für Drums/SFX): Eine Instanz gleichzeitig. Neuer Trigger schneidet alten ab.

**Poly**: N gleichzeitige Instanzen erlaubt. Bei Überschreitung wird via Voice-Stealing-Strategie eine alte Instanz beendet.

Polyphonie-Management passiert im Player drumherum, nicht in der Voice selbst.

## Voice-Modus UI

### Layout

```
┌──────────────────────────────────────────────────────────────┐
│ [Voice-Modus]  [Pattern-Modus]                               │
├──────────────────┬───────────────────────────────────────────┤
│ Voice-Browser    │ Voice-Editor                              │
│                  │                                           │
│ ▾ Drums          │ ┌─────────────────────────────────────┐   │
│   • kick         │ │ Source: kick-808.wav                │   │
│   • snare        │ │ [Waveform-View / Spektrogramm]      │   │
│   • hihat        │ │ ▒▓▓▓▒▒▒░░░░░                        │   │
│ ▾ SFX            │ │ ─── attack ─ release ──             │   │
│   • laser        │ │ [Label: Loop Start]                 │   │
│   • coin         │ └─────────────────────────────────────┘   │
│                  │                                           │
│ [+ New Voice]    │ ┌──────────┬──────────┬──────────┐        │
│ [Import Sample]  │ │ Modul.   │ Filter   │ FX-Chain │        │
│                  │ │ • amp    │ Lowpass  │ Distort. │        │
│                  │ │ • pitch  │ 800Hz    │ Delay    │        │
│                  │ │ [+ Add]  │          │ [+ Add]  │        │
│                  │ └──────────┴──────────┴──────────┘        │
│                  │                                           │
│                  │ Ext. Params: [pitch] [punch] [+]          │
│                  │ Polyphony: ● mono  ○ poly                 │
├──────────────────┴───────────────────────────────────────────┤
│ [▶ Trigger]  [Loop]  [MIDI: ●]  [Output: -6.0 dB]            │
└──────────────────────────────────────────────────────────────┘
```

### Waveform/Spektrogramm-View (Audacity-Style)

- Umschaltbar oder nebeneinander
- Zoom (Maus-Wheel oder Tasten)
- Selection-Bereich per Maus aufziehen
- Effekte/Operationen auf Selection anwendbar
- Labels per Doppelklick setzen (Loop Start/End, Marker, Cue)

### Editor-Sektionen

**Source-Sektion:**
- Sample-Datei auswählen oder importieren
- Root-Note setzen (für Pitch-Modulation)
- Start-Offset, Loop-Punkte, Reverse

**Modulation-Sektion:**
- Liste der Modulation-Slots
- Pro Slot: Target wählen, Source (Envelope/LFO) konfigurieren
- Visualisierung der Envelopes/LFOs

**Filter-Sektion:**
- Typ, Cutoff, Resonance
- Live-Frequency-Response-Visualisierung

**FX-Chain-Sektion:**
- Stack von FX-Slots
- Drag-and-Drop zum Umordnen
- Pro Slot: Live-Preview-Toggle, Bypass-Toggle, Parameter

**Externalized-Parameters-Sektion:**
- Liste deklarierter Parameter
- Pro Parameter: Binding-Konfiguration (siehe oben)

**Polyphony-Sektion:**
- Mono / Poly Toggle
- Bei Poly: Stimmenzahl, Stealing-Strategie

### Test-Triggering

- **Trigger-Button** für manuelles Antesten
- **Loop-Button** für kontinuierliches Retriggering
- **MIDI-In-Toggle** für Test mit externem Keyboard/Pads
- **Output-Level-Meter** und Master-Gain-Slider

## Sample-Import-Workflow

1. **Datei wählen** (Drag-and-Drop oder Browse)
2. **Auto-Analyse** läuft automatisch:
   - Sample-Rate, Bit-Tiefe, Channels detektiert
   - BPM-Schätzung (siehe `audio-analysis.md`)
   - Onset-Detection mit Marker-Vorschlägen
   - Loop-Point-Vorschläge
3. **Sample-Speicherort** im Projekt wählen (Default: `samples/`)
4. **Voice erstellen** — neue TS-Datei mit Sample als Source, Default-Parameter
5. **Editor öffnet** sich mit der neuen Voice

### Slice-Tool (siehe `audio-analysis.md`)

Bei einem Sample mit erkannten Onsets:

- **Slice at Beats** — zerschneidet Sample an Beat-Grenzen, erzeugt N Voices
- **Slice at Onsets** — feiner, an jedem Onset
- **Manuelles Slicing** über Selection-Bereiche

Jede Slice wird zu einer eigenen Voice mit dem Sub-Sample als Source.

## Live-Preview-Verhalten

Beim Editing einer Voice:

- Änderungen werden sofort hörbar bei nächstem Trigger
- Bei aktivem Loop-Modus werden Änderungen mit dem nächsten Loop-Cycle wirksam (kein Audio-Drop)
- Modulation-Änderungen werden live während Playback wirksam (Smoothing für klick-freie Übergänge)

## Voice-Pool und Cross-Mode-Konsequenzen

Der Voice-Pool ist eine zentrale Bibliothek aller Voices im Projekt. Änderungen propagieren in alle nutzenden Patterns:

- Voice umbenennen → TS-Imports in Patterns aktualisieren (via TS-Language-Server)
- Voice-Parameter ändern → Live-Auswirkung in aktiven Patterns
- Voice löschen → Warnung mit Liste der nutzenden Patterns

## Importierbare Voice-Quellen

- **Eigenes Format:** `.ts`-Voice-Files aus anderen Projekten
- **npm-Packages:** `npm install @amigo-labs/sfx-pack-retro`
- **Sample-Files:** Direkter Import erzeugt automatisch eine neue Voice mit Defaults

## Phase 2 — Erweiterungen

Diese Features sind in Phase 1 *nicht* enthalten:

- **Freier DSP-Graph** statt fixes Skelett (Sample → Mod → Filter → FX)
- **Oszillator-Sources** (synthese-from-scratch, Sfxr-Style)
- **Custom-Effekte** als WebAssembly-Plugins
- **Legacy-Import** von Sfxr-/SoundFont-Files

## Implementierungs-Reihenfolge

Für die Phase-1-Implementation:

1. Sample-Import + Basis-Playback (ohne Modulation/Filter/FX)
2. Waveform/Spektrogramm-View mit Selection
3. Envelope-Modulation (amp + pitch)
4. Filter-Sektion
5. Basis-FX (Distortion, Delay, Reverb)
6. Externalized Parameters
7. Polyphonie-Management
8. LFO-Modulation
9. Erweiterte FX (Chorus, Bitcrusher, EQ3)
10. Audacity-Style Labels und Marker
11. Slice-Tools (Integration mit `audio-analysis.md`)
12. MIDI-Integration
