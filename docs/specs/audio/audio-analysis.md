---
title: amigo-trommel — Audio-Analyse
status: draft
phase: 1
parent: amigo-trommel.md
last_updated: 2026-05-12
---

# Audio-Analyse

Klassische DSP-basierte Audio-Analyse für Sample-Workflows im Voice-Modus. Phase-1-Implementation nutzt ausschließlich klassische DSP-Methoden in Rust, kein ML.

## Zweck

Audio-Analyse unterstützt mehrere Sample-Workflows:

- **BPM-Schätzung** beim Import von Drum-Loops und Musik-Samples
- **Beat-Markers** auf der Waveform für Loop-Point-Setting und Slicing
- **Onset-Markers** für feinere Slice-Operationen
- **Stille-Erkennung** für Auto-Trim
- **Loop-Point-Vorschläge** für saubere Loops

Alle Analyse-Funktionen laufen in `amigo-audio-core` (Rust) und sind plattform-deterministisch.

## Beat-Detection

### Algorithmus

Standard-Ansatz:

1. **Spectral Flux** berechnen — Frame-by-Frame-Änderung der Spektren als Maß für „Energie-Schübe"
2. **Onset-Function** glätten (gleitender Mittelwert)
3. **Autocorrelation** der Onset-Function über typische Beat-Intervalle (60–200 BPM)
4. **Peak-Detection** in der Autocorrelation → BPM-Schätzung
5. **Beat-Position-Tracking** via dynamische Programmierung, um konsistente Beat-Grenzen zu finden

### Output

```rust
struct BeatAnalysis {
    bpm: f32,                    // Geschätzte BPM
    confidence: f32,             // 0..1, Sicherheit der Schätzung
    beats: Vec<BeatMarker>,      // Zeitstempel der Beats
    downbeats: Vec<usize>,       // Indices in `beats`, die Downbeats sind
}

struct BeatMarker {
    time_seconds: f32,
    strength: f32,               // 0..1, Stärke des Beats
}
```

### Genauigkeit

- BPM-Schätzung typisch ±1 BPM bei klar perkussivem Material
- Bei melodischem Material ohne klare Drums: geringere Konfidenz, möglicherweise halbe/doppelte Schätzung
- Downbeat-Detection ist schwieriger und liefert oft keine Aussage (Konfidenz < 0.5)

### Konfidenz-Indikator

Trommel-UI zeigt den Konfidenz-Wert. Bei niedriger Konfidenz wird die Schätzung als „unsicher" markiert, und der Nutzer kann sie manuell überschreiben.

## Onset-Detection

### Algorithmus

Onset = ein einzelner perkussiver Anschlag, egal welches Instrument.

1. **Spectral Flux** mit feiner Frame-Auflösung (typisch 5–10ms Hop-Size)
2. **Adaptive Threshold** basierend auf lokalem Median + Standardabweichung
3. **Peak-Detection** über Threshold
4. **Mindestabstand** zwischen Onsets (typisch 30ms), um Doppel-Detection zu vermeiden

### Output

```rust
struct OnsetAnalysis {
    onsets: Vec<OnsetMarker>,
}

struct OnsetMarker {
    time_seconds: f32,
    strength: f32,               // 0..1
    spectral_centroid: f32,      // Hz, „Helligkeit" des Onsets
}
```

`spectral_centroid` ist ein Hinweis auf das Instrument:

- Niedrig (< 500 Hz): wahrscheinlich Kick oder Bass-Drum
- Mittel (500–2000 Hz): wahrscheinlich Snare oder Tom
- Hoch (> 2000 Hz): wahrscheinlich Hi-Hat oder Cymbal

**Wichtig:** Das ist eine grobe Heuristik, keine ML-basierte Klassifikation. Phase 1 nutzt es nur als Hilfe für UI-Markers, nicht für Auto-Assignment.

## Loop-Point-Erkennung

### Algorithmus

Sucht „saubere" Loop-Punkte, an denen Sample-Anfang und -Ende sich nicht hörbar unterscheiden:

1. **Zero-Crossing-Detection** — Stellen, an denen das Signal die Null-Linie kreuzt (vermeidet Klicks)
2. **Amplitude-Matching** — Stellen mit ähnlicher Amplitude wie der Sample-Anfang
3. **Spectrale Ähnlichkeit** — FFT-basierter Vergleich kurzer Fenster
4. **Beat-Alignment** (optional, wenn BPM bekannt) — Loop-Punkte an Beat-Grenzen bevorzugen

### Output

```rust
struct LoopPointSuggestions {
    suggestions: Vec<LoopPointPair>,
}

struct LoopPointPair {
    start_seconds: f32,
    end_seconds: f32,
    quality: f32,                // 0..1, geschätzte Loop-Qualität
    method: LoopMethod,
}

enum LoopMethod {
    ZeroCrossing,
    AmplitudeMatch,
    SpectralMatch,
    BeatAligned,
}
```

UI zeigt die 3–5 besten Vorschläge mit Quality-Indikator. Der Nutzer wählt einen aus oder setzt Loop-Punkte manuell.

## Stille-Erkennung

### Algorithmus

Einfach: Bereiche mit RMS-Energie unter einem Threshold gelten als Stille.

```rust
struct SilenceAnalysis {
    leading_silence: f32,        // Sekunden Stille am Anfang
    trailing_silence: f32,       // Sekunden Stille am Ende
    internal_silence: Vec<SilenceRegion>, // Stille-Bereiche im Sample
}

struct SilenceRegion {
    start_seconds: f32,
    end_seconds: f32,
}
```

UI bietet „Auto-Trim" — schneidet Leading/Trailing-Silence ab.

## Slice-Tools

### Slice at Beats

Voraussetzung: Erfolgreiche Beat-Detection.

1. Sample wird an Beat-Grenzen geschnitten
2. N Sub-Samples werden erzeugt (N = Anzahl Beats)
3. Jede Slice wird zu einer eigenen Voice mit Default-Parametern
4. Voices werden im Voice-Modus angezeigt, Nutzer kann benennen/zuordnen

### Slice at Onsets

Wie „Slice at Beats", aber feiner — an jedem erkannten Onset.

Optional: **Strength-Threshold** — nur Onsets mit `strength > X` werden für Slicing verwendet.

### Manuelles Slicing

- Nutzer markiert Bereich auf der Waveform per Selection
- Klick auf „Slice" erzeugt eine Voice aus der Selection
- Mehrere Selections können nacheinander gesliced werden

## Integration im Voice-Modus

### Auto-Analyse beim Import

```
Sample-Import-Workflow:
  1. Datei-Auswahl
  2. Sample-Decoding (symphonia)
  3. Audio-Analyse läuft im Background:
     - Beat-Detection
     - Onset-Detection
     - Loop-Point-Suggestions
     - Silence-Detection
  4. Waveform-View zeigt Marker:
     - Beats als blaue Linien
     - Onsets als graue Punkte
     - Loop-Punkt-Vorschläge als gestrichelte Linien
     - Silence-Bereiche grau abgeblendet
  5. UI zeigt Analyse-Ergebnisse:
     - BPM: 120 (Confidence: 0.92)
     - 16 Beats erkannt
     - 64 Onsets erkannt
     - Vorgeschlagene Loop: 0.0s – 8.0s
```

### Marker-Editing

Marker können vom Nutzer verschoben, hinzugefügt, gelöscht werden. Die UI unterscheidet:

- **Auto-Marker** (vom Algorithmus erzeugt) — grau, nicht editiert
- **User-Marker** (vom Nutzer angepasst) — farbig, persistent

User-Marker werden im Sample-Metadata gespeichert (zusätzliche `.trommel-meta`-Datei oder direkt im Voice-TS-File).

## Performance

### Web (WASM)

Analyse läuft in einem Web Worker, blockiert das UI nicht.

Typische Performance auf modernem Hardware:

- 30s Sample, 44.1kHz, Stereo: ~0.5–1s für komplette Analyse (Beat + Onset + Loop + Silence)
- 5-Minuten-Song: ~3–8s

### Native (Tauri)

Direkter Rust-Aufruf, ähnliche Performance, evtl. etwas schneller durch fehlende WASM-Overhead.

## Implementierungs-Bibliotheken

In `amigo-audio-core`:

- **FFT:** `rustfft` (etabliert, schnell, deterministisch)
- **Audio-Decoding:** `symphonia` (alle Audio-Formate)
- **DSP-Bausteine:** `fundsp` für komplexere Operationen, eigene Implementation für Beat/Onset
- **Resampling:** `rubato` (Sample-Rate-Konversion)

Aktive Vermeidung von ML-Dependencies in Phase 1.

## Determinismus

Alle Audio-Analyse-Funktionen sind plattform-deterministisch:

- Single-Threaded Berechnung
- Feste FFT-Implementierung (rustfft, keine plattform-spezifischen FFT-Libraries)
- Keine Floating-Point-Reordering durch SIMD-Pfade
- Reproduzierbar via Hash-Test in CI

## API

### Rust (in `amigo-audio-core`)

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

### JavaScript (in `amigo-trommel-runtime`)

```typescript
interface AudioAnalysis {
  analyzeBeats(buffer: AudioBuffer): Promise<BeatAnalysis>;
  analyzeOnsets(buffer: AudioBuffer): Promise<OnsetAnalysis>;
  suggestLoopPoints(buffer: AudioBuffer, bpm?: number): Promise<LoopPointSuggestions>;
  analyzeSilence(buffer: AudioBuffer): Promise<SilenceAnalysis>;
}
```

Die JS-Schicht wrapped die WASM/Native-Calls und gibt typed Results zurück.

## Phase 2 — ML-basierte Drum-Transcription

Phase 1 endet hier mit klassischer DSP. Phase 2 erweitert um ML-basierte Funktionen:

- **Drum-Component-Klassifikation** (Kick/Snare/Hi-Hat/Cymbal pro Onset)
- **Drum-Pattern-Extraction** (Strudel-Pattern aus Audio generieren)
- **Source-Separation** (optional, später)

Phase 2 wird in einer separaten Spec dokumentiert.

## Implementierungs-Reihenfolge

Für Phase-1-Implementation:

1. Sample-Decoding via symphonia
2. RMS- und Spectral-Flux-Berechnung
3. Onset-Detection
4. BPM-Schätzung via Autocorrelation
5. Beat-Position-Tracking
6. Silence-Detection
7. Loop-Point-Suggestion (Zero-Crossing, Amplitude-Match)
8. Spectrale Loop-Point-Suggestion
9. Slice-Tools (Beat-basiert, Onset-basiert, manuell)
10. Web Worker Integration für non-blocking Analyse
11. Marker-Editing UI im Voice-Modus
