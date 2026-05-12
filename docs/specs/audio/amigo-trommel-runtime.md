---
title: amigo-trommel-runtime
status: draft
phase: 1
parent: amigo-trommel.md
last_updated: 2026-05-12
---

# amigo-trommel-runtime

JavaScript-/TypeScript-Layer zwischen Strudel und `amigo-audio-core`. Bietet die TypeScript-API für Voice-Definitionen, Pattern-Eval-Integration und Type-Safety.

## Zweck

`amigo-trommel-runtime` ist die Brücke zwischen:

- **Strudel** (Pattern-Sprache, evaluiert Patterns zu Events)
- **TypeScript-Voice-Definitions** (vom Nutzer geschrieben)
- **amigo-audio-core** (Rust-DSP-Engine, instanziiert Voices)

Die Runtime läuft auf Web (direkt im Browser) und auf Native (über QuickJS-NG in `amigo-trommel`-Tauri-App und in der `amigo-engine`).

## Plattform-Targets

- **Browser** (PWA): Browser-natives JS (V8/SpiderMonkey/JavaScriptCore)
- **Tauri Desktop**: QuickJS-NG via rquickjs
- **amigo-engine**: QuickJS-NG via rquickjs

Verhalten ist semantisch identisch über alle Plattformen, kleine Performance-Unterschiede sind erlaubt.

## Package-Struktur

```
amigo-trommel-runtime/
├── src/
│   ├── index.ts                  # Public API
│   ├── voice/
│   │   ├── define-voice.ts       # defineVoice()-API
│   │   ├── voice-source.ts       # sample(), oscillator() (Phase 2)
│   │   ├── modulation.ts         # envelope(), lfo()
│   │   ├── filter.ts             # filter()
│   │   └── fx/                   # distortion(), delay(), reverb(), ...
│   ├── pattern/
│   │   ├── define-pattern.ts     # definePattern()-API
│   │   ├── strudel-bridge.ts     # Strudel-Integration
│   │   └── voice-output.ts       # Strudel-Output-Target
│   ├── analysis/
│   │   └── audio-analysis.ts     # Audio-Analyse-API (wrapped WASM)
│   ├── engine-bridge/
│   │   ├── web.ts                # Browser/AudioWorklet-Bridge
│   │   └── native.ts             # QuickJS/Tauri-Bridge
│   └── types/
│       └── ...                   # Type-Definitionen
├── package.json
└── tsconfig.json
```

## Public API

### Voice-Definition-API

```typescript
import { defineVoice, sample, envelope, filter, distortion } from '@amigo/trommel';

export const kick = defineVoice({
  source: sample({ file: '../samples/kick.wav' }),
  modulation: [
    envelope({ target: 'amp', attack: 0, decay: 0.3, sustain: 0, release: 0.1 })
  ],
  filter: filter({ type: 'lowpass', cutoff: 800 }),
  fx: [distortion({ drive: 0.2 })],
  externalParams: ['pitch', 'punch']
});
```

### Pattern-Definition-API

```typescript
import { definePattern } from '@amigo/trommel';
import { stack, s } from '@strudel/core';
import { kick, snare } from '../voices/drums';

export default definePattern({
  pattern: stack(
    s("bd*4"),
    s("~ sd ~ sd")
  ),
  tempo: 120,
  mixer: {
    tracks: {
      kick: { gain: 1.0, pan: 0 },
      snare: { gain: 0.8 }
    }
  }
});
```

### Engine-Bridge-API

Die Runtime hat eine plattform-unabhängige API zum Triggern von Voices über `amigo-audio-core`:

```typescript
import { createAudioEngine } from '@amigo/trommel';

const engine = await createAudioEngine({
  sampleRate: 48000,
  bufferSize: 128
});

// Voice laden
await engine.loadVoice('kick', kickDefinition);

// Voice triggern
const instanceId = engine.trigger('kick', {
  velocity: 0.8,
  pitch: 0,
  pan: 0
});

// Voice releasen
engine.release(instanceId);
```

Die `createAudioEngine`-Factory wählt automatisch die richtige Implementation (Browser oder Native).

## Type-System

### Voice-Type

```typescript
export interface VoiceDefinition<TParams extends string = string> {
  source: VoiceSource;
  modulation?: ModulationSlot[];
  filter?: FilterConfig;
  fx?: FxConfig[];
  externalParams?: TParams[];
  paramBindings?: ParamBindings<TParams>;
  polyphony?: PolyphonyConfig;
}

export type Voice<TParams extends string = string> = {
  __voiceDefinition: VoiceDefinition<TParams>;
  __voiceId: string;
  trigger(params?: VoiceTriggerParams<TParams>): VoiceTriggerNode;
  release(): void;
};

export type VoiceTriggerParams<TParams extends string> = {
  velocity?: number;
  pitch?: number;
  pan?: number;
} & {
  [K in TParams]?: ParamValue;
};
```

### Type-Safe Voice-Trigger

TypeScript erkennt die deklarierten `externalParams`:

```typescript
const kick = defineVoice({
  source: sample({ file: 'kick.wav' }),
  externalParams: ['pitch', 'punch']
});

kick.trigger({ pitch: 1.5, punch: 0.8 });     // ✓ OK
kick.trigger({ pitch: 1.5, brightness: 1 });   // ✗ Compile-Error
```

### Pattern-Type

```typescript
export interface PatternDefinition {
  pattern: StrudelPattern;
  tempo?: number;
  mixer?: MixerConfig;
  swing?: number;
  humanize?: number;
}

export type StrudelPattern = unknown; // Strudel's Pattern-Typ, re-exportiert
```

## Strudel-Integration

### Strudel-Setup

Die Runtime initialisiert Strudel mit einem Custom-Output-Target:

```typescript
import { evalScope, initAudio } from '@strudel/core';

async function setupStrudel(engine: AudioEngine) {
  await initAudio();
  
  // Custom Output: Strudel-Sound-Refs auf Trommel-Voices mappen
  evalScope(import('@strudel/core'));
  evalScope({
    s: (soundName: string) => {
      // Strudel-Pattern, das Voice via engine triggert
      return strudelPatternWithCustomTrigger(soundName, engine);
    }
  });
}
```

### Voice-Output-Target

Strudel triggert standardmäßig Sample-basierte Sounds. Trommel ersetzt das durch Voice-Trigger:

```typescript
function strudelPatternWithCustomTrigger(
  voiceName: string,
  engine: AudioEngine
): StrudelPattern {
  return pattern.withValue((value, context) => {
    const params = extractStrudelParams(context); // gain, pan, etc.
    engine.trigger(voiceName, {
      velocity: params.gain ?? 1.0,
      pan: params.pan ?? 0,
      ...extractCustomParams(context)
    });
  });
}
```

### Strudel-Aliases

Strudel hat Standard-Aliases wie `bd` für Kick, `sd` für Snare. Trommel respektiert diese:

```typescript
const STRUDEL_ALIASES = {
  bd: 'kick',
  sd: 'snare',
  hh: 'hihat',
  oh: 'openhat',
  // ...
};
```

Wenn Voice mit Strudel-Alias-Name existiert, wird sie über Alias triggerbar. Custom-Voices nutzen direkt ihren Namen.

## Engine-Bridge-Implementations

### Web (Browser)

```typescript
// engine-bridge/web.ts

export class WebAudioEngine implements AudioEngine {
  private audioContext: AudioContext;
  private workletNode: AudioWorkletNode;
  
  constructor(config: AudioEngineConfig) {
    this.audioContext = new AudioContext({ sampleRate: config.sampleRate });
    // ... Worklet-Setup
  }
  
  async loadVoice(id: string, definition: VoiceDefinition): Promise<void> {
    this.workletNode.port.postMessage({
      type: 'load_voice',
      id,
      definition: serializeVoiceDefinition(definition)
    });
  }
  
  trigger(voiceId: string, params: VoiceTriggerParams): InstanceId {
    const instanceId = generateInstanceId();
    this.workletNode.port.postMessage({
      type: 'trigger',
      voiceId,
      instanceId,
      params
    });
    return instanceId;
  }
  
  // ...
}
```

Im AudioWorklet läuft das `amigo-audio-core` als WASM:

```javascript
// audio-worklet-processor.js (in trommel app)

class TrommelProcessor extends AudioWorkletProcessor {
  constructor(options) {
    super();
    this.wasmEngine = null;
    this.port.onmessage = this.handleMessage.bind(this);
    this.loadWasm(options.processorOptions.wasmBytes);
  }
  
  async loadWasm(bytes) {
    const module = await WebAssembly.compile(bytes);
    const instance = await WebAssembly.instantiate(module);
    this.wasmEngine = new instance.exports.WebAudioEngine(sampleRate, 128);
  }
  
  process(inputs, outputs, parameters) {
    if (!this.wasmEngine) return true;
    const output = outputs[0][0]; // mono erstmal
    this.wasmEngine.process(output);
    return true;
  }
  
  handleMessage(event) {
    // load_voice, trigger, release, ... an wasmEngine weiterleiten
  }
}

registerProcessor('trommel-processor', TrommelProcessor);
```

### Native (QuickJS in Tauri/Engine)

```typescript
// engine-bridge/native.ts

export class NativeAudioEngine implements AudioEngine {
  // Tauri/Engine stellt globale Bridge-Funktionen bereit
  // (über QuickJS-Function-Bindings)
  
  async loadVoice(id: string, definition: VoiceDefinition): Promise<void> {
    globalThis.__trommel_native__.loadVoice(id, JSON.stringify(definition));
  }
  
  trigger(voiceId: string, params: VoiceTriggerParams): InstanceId {
    return globalThis.__trommel_native__.trigger(voiceId, JSON.stringify(params));
  }
  
  // ...
}
```

Auf Rust-Seite werden diese Funktionen über rquickjs registriert und an die native `amigo-audio-core`-API weitergeleitet.

## Audio-Analyse-API

JS-Wrapper für die Rust-Analyse-Funktionen aus `amigo-audio-core`:

```typescript
import { analyzeBeats, analyzeOnsets, suggestLoopPoints } from '@amigo/trommel/analysis';

const buffer: ArrayBuffer = await loadAudioFile('sample.wav');

const beats = await analyzeBeats(buffer);
console.log(beats.bpm, beats.confidence);
console.log(beats.beats); // Array of BeatMarker

const onsets = await analyzeOnsets(buffer);
const loops = await suggestLoopPoints(buffer, beats.bpm);
```

Implementation:

- Web: WASM-Funktionen werden in Web Worker ausgeführt (non-blocking)
- Native: direkter rquickjs-Bridge-Call

## Serialization

Voice- und Pattern-Definitionen müssen serialisierbar sein:

- **Für AudioWorklet:** JSON-Serialization über MessagePort
- **Für QuickJS-Bridge:** JSON-Serialization über Function-Calls
- **Für `.trommel`-Bundle-Build:** JSON-Serialization für statischen Output

```typescript
export function serializeVoiceDefinition(def: VoiceDefinition): string {
  return JSON.stringify(def);
}

export function deserializeVoiceDefinition(json: string): VoiceDefinition {
  return JSON.parse(json);
}
```

Voice-Definitionen sind absichtlich datenlastig (keine Closures, keine Funktionen), damit Serialization trivial bleibt.

## Strudel-Pattern-Serialization

Strudel-Patterns enthalten Closures und können nicht direkt JSON-serialisiert werden. Zwei Strategien:

**Strategie A: Pattern-Source-Code wird gespeichert** (Phase 1)

Im `.trommel`-Bundle werden Patterns als JS-Source-Strings gespeichert. Engine evaluiert sie zur Laufzeit via QuickJS:

```json
{
  "patterns": {
    "combat": {
      "source": "stack(s('bd*4'), s('~ sd ~ sd'))",
      "type": "dynamic"
    }
  }
}
```

**Strategie B: Pattern wird pre-evaluated als Event-Stream** (für static Patterns)

Wenn ein Pattern keine Live-Parameter hat, wird es zur Build-Zeit evaluiert und als JSON-Event-Stream gespeichert:

```json
{
  "patterns": {
    "intro": {
      "type": "static_events",
      "duration": 4.0,
      "events": [
        { "time": 0.0, "voice": "kick", "params": {} },
        { "time": 0.5, "voice": "snare", "params": {} },
        ...
      ]
    }
  }
}
```

Phase 1 implementiert beide Strategien, `amigo-trommel-build` entscheidet automatisch welche basierend auf statischer Analyse des Pattern-Codes.

## MIDI-Integration

```typescript
import { createMidiInput, createMidiOutput } from '@amigo/trommel/midi';

const midiIn = await createMidiInput();
midiIn.on('noteOn', (note, velocity, channel) => {
  // Trigger Voice mit MIDI-Note als Pitch
  engine.trigger('kick', { pitch: note - 60, velocity: velocity / 127 });
});

const midiOut = await createMidiOutput();
midiOut.sendNoteOn(60, 127, 0);
```

Web: über Web MIDI API
Native: über `midir`-Crate via QuickJS-Bridge

## Error Handling

Alle async APIs werfen typed Errors:

```typescript
export class VoiceLoadError extends Error {
  constructor(public voiceId: string, message: string);
}

export class AudioEngineError extends Error {
  // ...
}
```

## Implementierungs-Reihenfolge

Für Phase-1-Implementation:

1. Package-Setup, Module-Struktur
2. Type-Definitionen (Voice, Pattern, Engine)
3. `defineVoice()`-API mit Source, Modulation, Filter, FX
4. Voice-Serialization (JSON)
5. WebAudioEngine via AudioWorklet
6. WASM-Loading im Worklet
7. Voice-Loading via MessagePort
8. Voice-Trigger via MessagePort
9. Strudel-Integration mit Custom-Output-Target
10. `definePattern()`-API
11. Pattern-Eval und Voice-Triggering aus Strudel-Events
12. NativeAudioEngine für QuickJS-Bridge
13. Tauri-Integration (Native-Bridge via rquickjs)
14. Audio-Analyse-API (WASM-Wrapper)
15. MIDI-Integration (Web MIDI API + native midir-Bridge)
16. Static-Pattern-Pre-Evaluation für Build-Output
