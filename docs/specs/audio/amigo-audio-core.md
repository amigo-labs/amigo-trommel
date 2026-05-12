---
title: amigo-audio-core
status: draft
phase: 1
parent: amigo-trommel.md
last_updated: 2026-05-12
---

# amigo-audio-core

Rust-Crate mit der zentralen Audio-Engine. Wird von `amigo-trommel`, `amigo-trommel-build` und der `amigo-engine` konsumiert.

## Zweck

`amigo-audio-core` ist die DSP- und Audio-Runtime-Schicht. Verantwortlich für:

- Sample-Player (Pitch-Shift, Loop, Reverse)
- Modulation-System (Envelopes, LFOs)
- Filter (Low/High/Band/Notch-Pass)
- Internal FX (Distortion, Delay, Reverb, etc.)
- Mixer mit Bus-Routing
- Audio-Analyse (Beat, Onset, Loop, Silence)
- Voice-Trigger-API für externe Konsumenten

Die Crate hat *keinen* Strudel-Code, *keine* TypeScript-Integration, *keinen* UI-Code. Sie ist reine Audio-Engine.

## Plattform-Targets

`amigo-audio-core` muss auf folgenden Targets bauen:

- `wasm32-unknown-unknown` (für Web/AudioWorklet)
- `x86_64-pc-windows-msvc` (Windows native)
- `x86_64-unknown-linux-gnu` (Linux native, CI)
- `aarch64-apple-darwin` (macOS Apple Silicon)
- `x86_64-apple-darwin` (macOS Intel)
- `aarch64-unknown-linux-gnu` (Linux ARM, falls Engine dort läuft)

Phase 2 ergänzt um Mobile (`aarch64-linux-android`, `aarch64-apple-ios`).

## Architektur

### Module-Struktur

```
amigo-audio-core/
├── src/
│   ├── lib.rs
│   ├── sample/             # Sample-Decoding, -Loading, -Playback
│   ├── voice/              # Voice-Definition, Voice-Instance
│   ├── modulation/         # Envelopes, LFOs
│   ├── filter/             # Filter-Implementationen
│   ├── fx/                 # Effekt-Implementationen
│   ├── mixer/              # Bus-Routing, Pan, Gain
│   ├── analysis/           # Beat/Onset/Loop/Silence-Detection
│   ├── render/
│   │   ├── deterministic.rs # CI/Pre-Render-Pfad
│   │   └── realtime.rs      # Runtime-Pfad
│   └── api/                # FFI/WASM-Bindings
├── tests/
│   ├── determinism/        # Plattform-übergreifende Bit-Identität
│   ├── fixtures/           # Test-Audio-Files
│   └── ...
└── benches/                # Performance-Benchmarks
```

### Crate-Dependencies

**Audio-I/O:**
- `symphonia` — Audio-Decoding (WAV, FLAC, OGG, Opus, MP3, AIFF)
- `hound` — WAV-Encoding
- `vorbis_rs` oder `ogg`+`lewton`/`vorbis-encoder` — OGG-Encoding
- `opus` — Opus-Encoding (kleinere WASM-Story als Vorbis)

**DSP:**
- `rustfft` — FFT für Analyse und Spektrum-Operationen
- `fundsp` — DSP-Building-Blocks (no_std-kompatibel, WASM-tauglich)
- `rubato` — Sample-Rate-Konversion

**Native Audio-Output (für Tauri):**
- `cpal` — Cross-Platform Audio I/O

**Web Audio (über JS-Bridge):**
- `wasm-bindgen` für JS-Interop
- AudioWorklet-Integration über JS-Layer

### Determinismus-Strategie

Zwei Render-Pfade:

**`render_deterministic`** (CI/Pre-Render):
- Single-Threaded
- Feste FFT-Implementation (rustfft, kein plattform-FFT)
- Keine SIMD-Pfade
- Floating-Point-Operations in deterministischer Reihenfolge
- Reproduzierbar via Hash-Test in CI über alle Plattformen

**`render_realtime`** (Runtime):
- Plattform-Optimierungen erlaubt (SIMD via `std::simd` oder explicit)
- Parallelisierung möglich für FX-Chains
- Sub-Millisekunden-Unterschiede zwischen Plattformen akzeptabel

Beide Pfade nutzen dieselben DSP-Bausteine, aber unterschiedliche Render-Wrapper.

## Voice-System

### Voice-Definition

Eine Voice wird als Datenstruktur beschrieben (von außen via JSON oder direkt in Rust konstruiert):

```rust
pub struct VoiceDefinition {
    pub source: VoiceSource,
    pub modulation: Vec<ModulationSlot>,
    pub filter: FilterConfig,
    pub fx: Vec<FxConfig>,
    pub external_params: Vec<String>,
    pub param_bindings: HashMap<String, ParamBinding>,
    pub polyphony: PolyphonyConfig,
}

pub enum VoiceSource {
    Sample {
        file_path: PathBuf,
        root_note: NoteName,
        loop_config: LoopConfig,
        start_offset: f32,
        reverse: bool,
        detune_cents: f32,
    },
    // Phase 2: Oscillator { ... }
}
```

### Voice-Instance

Pro Trigger wird eine `VoiceInstance` erzeugt:

```rust
pub struct VoiceInstance {
    definition: Arc<VoiceDefinition>,
    sample_player: SamplePlayer,
    modulation: ModulationState,
    filter: FilterState,
    fx_chain: FxChainState,
    params: VoiceParams,
    state: InstanceState,
}

pub struct VoiceParams {
    pub pitch: f32,            // Halbtöne
    pub velocity: f32,          // 0..1
    pub pan: f32,               // -1..+1
    pub external: HashMap<String, ParamValue>,
}
```

### Triggering

```rust
impl VoiceManager {
    pub fn trigger(&mut self, voice_id: VoiceId, params: VoiceParams) -> InstanceId;
    pub fn release(&mut self, instance_id: InstanceId);
    pub fn set_param(&mut self, instance_id: InstanceId, param: &str, value: ParamValue);
    pub fn stop(&mut self, instance_id: InstanceId);
}
```

Voice-Manager kümmert sich um Polyphonie-Management und Instance-Lifecycle.

## DSP-Bausteine

### Sample-Player

```rust
pub struct SamplePlayer {
    sample: Arc<Sample>,
    position: f64,             // Sub-Sample-Genauigkeit
    pitch_ratio: f32,
    loop_config: LoopConfig,
}

impl SamplePlayer {
    pub fn process(&mut self, output: &mut [f32], sample_rate: u32);
}
```

Pitch-Shifting via Resampling:

- **Linear** (schnell, qualitativ ok)
- **Hermite** (besser, mittlere Kosten)
- **Sinc** (höchste Qualität, teuer) — für Pre-Render-Pfad

### Envelope

```rust
pub struct AdsrEnvelope {
    attack: f32,
    decay: f32,
    sustain: f32,
    release: f32,
    curve: EnvelopeCurve,
    state: EnvelopeState,
}

impl AdsrEnvelope {
    pub fn process(&mut self, gate: bool, sample_rate: u32) -> f32;
}
```

### LFO

```rust
pub struct Lfo {
    waveform: LfoWaveform,
    rate_hz: f32,
    phase: f32,
    amount: f32,
}

impl Lfo {
    pub fn process(&mut self, sample_rate: u32) -> f32;
}
```

### Filter

State-Variable-Filter, stabil bei Modulation:

```rust
pub struct SvfFilter {
    filter_type: FilterType,
    cutoff_hz: f32,
    resonance: f32,
    state: SvfState,
}

impl SvfFilter {
    pub fn process(&mut self, input: f32, sample_rate: u32) -> f32;
    pub fn set_cutoff(&mut self, hz: f32);
    pub fn set_resonance(&mut self, q: f32);
}
```

### FX

Jeder FX implementiert `AudioEffect`:

```rust
pub trait AudioEffect: Send {
    fn process(&mut self, buffer: &mut [f32], sample_rate: u32);
    fn set_param(&mut self, param: &str, value: f32);
}
```

Phase-1-Effekte:

- `Distortion` (Soft, Hard, Foldback)
- `Delay` (mit Feedback, Ping-Pong-Option)
- `Reverb` (Plate, Hall, Spring — über fundsp oder eigene Implementation)
- `Chorus`
- `Bitcrusher`
- `Eq3` (3-Band)
- `Compressor` (für Bus-Level)
- `Limiter` (für Master)

## Mixer

```rust
pub struct Mixer {
    tracks: HashMap<TrackId, Track>,
    busses: HashMap<BusId, Bus>,
    master: MasterBus,
}

pub struct Track {
    gain: f32,
    pan: f32,
    solo: bool,
    mute: bool,
    fx: Vec<Box<dyn AudioEffect>>,
    sends: Vec<Send>,
}

pub struct Bus {
    gain: f32,
    fx: Vec<Box<dyn AudioEffect>>,
}

pub struct Send {
    target_bus: BusId,
    amount: f32,
}
```

Mixer-Verarbeitung pro Audio-Block:

1. Alle aktiven Voice-Instances rendern in ihre Track-Buffer
2. Track-FX werden angewendet
3. Track-Output → Track-Gain/Pan → Master-Bus
4. Send-Outputs → Send-Busse → Master
5. Master-FX → Master-Output

## Voice-Manager-API

Hauptschnittstelle für externe Konsumenten (JS-Runtime, Engine):

```rust
pub struct AudioEngine {
    voice_manager: VoiceManager,
    mixer: Mixer,
    voice_definitions: HashMap<VoiceId, Arc<VoiceDefinition>>,
}

impl AudioEngine {
    pub fn new(sample_rate: u32, buffer_size: usize) -> Self;
    
    pub fn load_voice(&mut self, id: VoiceId, definition: VoiceDefinition);
    pub fn unload_voice(&mut self, id: VoiceId);
    
    pub fn trigger(&mut self, voice_id: VoiceId, params: VoiceParams) -> InstanceId;
    pub fn release(&mut self, instance_id: InstanceId);
    pub fn set_instance_param(&mut self, instance_id: InstanceId, param: &str, value: ParamValue);
    
    pub fn process(&mut self, output: &mut [f32]);
    
    // Mixer-Operationen
    pub fn set_track_gain(&mut self, track_id: TrackId, gain: f32);
    pub fn set_track_pan(&mut self, track_id: TrackId, pan: f32);
    // ...
}
```

## Audio-Analyse-API

Siehe `audio-analysis.md` für Algorithmus-Details.

```rust
pub mod analysis {
    pub fn analyze_beats(samples: &[f32], sample_rate: u32) -> BeatAnalysis;
    pub fn analyze_onsets(samples: &[f32], sample_rate: u32) -> OnsetAnalysis;
    pub fn suggest_loop_points(
        samples: &[f32],
        sample_rate: u32,
        bpm: Option<f32>,
    ) -> LoopPointSuggestions;
    pub fn analyze_silence(samples: &[f32], sample_rate: u32) -> SilenceAnalysis;
}
```

## WASM-Bindings

Für Web-Konsum:

```rust
#[wasm_bindgen]
pub struct WebAudioEngine {
    inner: AudioEngine,
}

#[wasm_bindgen]
impl WebAudioEngine {
    #[wasm_bindgen(constructor)]
    pub fn new(sample_rate: u32, buffer_size: usize) -> Self;
    
    pub fn load_voice_json(&mut self, id: String, json: &str) -> Result<(), JsValue>;
    pub fn trigger(&mut self, voice_id: String, params_json: &str) -> u64;
    pub fn process(&mut self, output: &mut [f32]);
    // ...
}
```

AudioWorklet-Integration:

- WASM-Modul wird im AudioWorklet-Scope geladen
- `process()` wird pro Render-Quantum (128 Samples) aufgerufen
- Voice-Trigger-Events kommen via MessagePort vom Main-Thread

## Native-API für Engine

Für Konsum aus `amigo-engine`:

```rust
// Re-Export der zentralen Typen
pub use crate::AudioEngine;
pub use crate::VoiceDefinition;
pub use crate::VoiceParams;
// ...
```

Engine kann direkte Rust-API nutzen, ohne JSON-Roundtrip.

## Testing

### Determinismus-Tests

```
tests/determinism/
├── test_render_voice_kick.rs    # Render Kick mit definierten Params, vergleiche Hash
├── test_render_pattern.rs        # Render Pattern, vergleiche Hash
├── test_analyze_sample.rs        # Analyse-Output deterministisch
└── fixtures/
    ├── kick-808.wav
    ├── expected_kick_render.wav  # Reference-Output
    └── expected_kick_hash.txt
```

CI führt diese Tests auf allen Plattformen aus. Hash-Mismatch = Build-Failure.

### Performance-Benchmarks

```
benches/
├── bench_voice_trigger.rs        # Voice-Trigger-Latenz
├── bench_render_realtime.rs      # Realtime-Render-Performance
└── bench_analysis.rs             # Audio-Analyse-Performance
```

Performance-Targets:

- Voice-Trigger-Latenz: < 1ms auf modernem Hardware
- Realtime-Render: 256 Voices parallel ohne Audio-Dropouts (Buffer 128 Samples, 44.1kHz)
- Audio-Analyse (30s Sample): < 1s

## Phase 2 — Erweiterungen

- **Oscillator-Sources** (Sägezahn, Quadrat, Sinus, FM, Noise) für synthesis-from-scratch Voices
- **Freier DSP-Graph** statt fixes Skelett
- **SIMD-Optimierungen** für Realtime-Pfad
- **GPU-Beschleunigte FFT** für Analyse (WebGPU)
- **ML-Integration** für Drum-Transcription (via ONNX Runtime)

## Implementierungs-Reihenfolge

Für Phase-1-Implementation:

1. Crate-Setup, Module-Struktur, Plattform-Targets
2. Sample-Decoding via symphonia
3. Basis-Sample-Player (kein Pitch-Shift, keine Modulation)
4. WASM-Bindings und AudioWorklet-Integration
5. AdsrEnvelope
6. Pitch-Shifting via Resampling
7. SvfFilter
8. Voice-Definition und VoiceManager
9. Basis-Effekte (Distortion, Delay, Reverb)
10. Mixer mit Track/Bus/Master
11. LFO
12. Erweiterte Effekte (Chorus, Bitcrusher, EQ3, Compressor, Limiter)
13. Send-Routing im Mixer
14. Audio-Analyse-Modul (siehe audio-analysis.md)
15. Determinismus-Test-Suite
16. Performance-Benchmarks und Optimierung
17. Native-API für Engine-Integration
