---
title: amigo-trommel — Spec-Übersicht
status: draft
phase: 1
last_updated: 2026-05-12
---

# amigo-trommel — Spec-Übersicht

Diese Spec-Sammlung beschreibt **amigo-trommel**, ein TypeScript-natives Game-Audio-Authoring-Tool mit Sample-basierten Voices, Strudel-Pattern-System und Engine-Integration für die Amigo Engine.

## Status

Phase 1, alle Specs `draft`. Voraussetzung für Implementation: **Pincel zu v1.0 bringen**.

## Spec-Dokumente

### Architektur und Workflow

- **[amigo-trommel.md](./amigo-trommel.md)** — Hauptspec mit Architektur-Übersicht, Workflow, Cross-Mode-Concerns, Phase-Definition
- **[amigo-trommel/voice-mode.md](./amigo-trommel/voice-mode.md)** — Voice-Modus im Detail
- **[amigo-trommel/pattern-mode.md](./amigo-trommel/pattern-mode.md)** — Pattern-Modus mit Grid-View und Code-View
- **[amigo-trommel/audio-analysis.md](./amigo-trommel/audio-analysis.md)** — Klassische DSP-Audio-Analyse (Beat, Onset, Loop, Silence)

### Sub-Komponenten

- **[amigo-audio-core.md](./amigo-audio-core.md)** — Rust-Crate mit DSP, Voice-Runtime, Mixer
- **[amigo-trommel-runtime.md](./amigo-trommel-runtime.md)** — TypeScript/JavaScript-Layer mit Strudel-Integration
- **[amigo-trommel-build.md](./amigo-trommel-build.md)** — CLI Build-Tool für Engine-Bundle-Erzeugung

### Konventionen und Formate

- **[import-export-formats.md](./import-export-formats.md)** — Vollständige Import/Export-Format-Spec

## Phase-1-Scope

- Voice-Modus mit Sample-Import, Modulation, Filter, FX-Chain
- Pattern-Modus mit Grid-View und Code-View (Strudel)
- Klassische DSP-Audio-Analyse (Beat, Onset, Loop, Silence)
- MIDI-Integration (In/Out/Export)
- Engine-Export als `.trommel`-Bundle
- Plattformen: Web (PWA) + Desktop (Tauri v2)
- Audacity-Style UX (Waveform/Spektrogramm, Labels, Selection, Effekt-Chain)

## Phase-2-Ausblick (separate Specs später)

- Browser-only ML-Drum-Transcription via ONNX Runtime Web
- Legacy-Importer (Sfxr-Familie, SoundFont, DecentSampler)
- Freier DSP-Graph für Voices (statt fixes Skelett)
- Oscillator-Sources (synthesis-from-scratch)
- Mobile-Apps (iOS/Android via Tauri)
- AI-Integration (Voice-Generation, Pattern-Suggestion)
- Community-Voice-/Pattern-Library
- Embed-Player für Web-Use-Cases

## Verwandte Specs außerhalb dieser Sammlung

- **amigo-engine Audio-Integration** — Erweiterung der bestehenden Engine-Audio-Spec um Trommel-Asset-Konsum, Voice-Live-Synthese, Pattern-Triggering, **Adaptive Music** (lebt in der Engine, nicht in Trommel)
- **Pincel** — Schwester-Tool für Pixel-Art unter amigo-labs

## Stack-Zusammenfassung

| Schicht | Technologie |
|---|---|
| Frontend | Svelte 5 |
| Audio-DSP | Rust (`amigo-audio-core`), kompiliert nach WASM und nativ |
| Audio-Output Web | AudioWorklet |
| Audio-Output Native | cpal |
| TS-Kompilierung | SWC |
| JS-Runtime Web | Browser-natives JS |
| JS-Runtime Native | QuickJS-NG via rquickjs |
| Pattern-Sprache | Strudel (direkt verwendet) |
| Desktop-Wrapper | Tauri v2 |
| Build-Tool | Vite (Editor) + custom Rust-CLI (Bundle-Build) |

## Source-of-Truth-Prinzip

Alle Voices, Patterns und Konfigurationen sind **TypeScript-Code** im Projekt-Verzeichnis. Git-diff-bar, type-safe, im JS-Ecosystem natürlich teilbar (npm-Packages).

Das Tool ist eine **GUI über TypeScript-Code**, nicht ein eigenständiges proprietäres Format. Direkter Editor-Zugriff über VSCode/JetBrains funktioniert parallel zur Trommel-UI.

## Repository-Struktur

```
amigo-trommel/                       # Monorepo
├── crates/
│   └── amigo-audio-core/            # Rust-DSP-Crate
├── packages/
│   ├── amigo-trommel-runtime/       # JS-Layer
│   └── amigo-trommel-build/         # Build-Tool (Rust + JS Runtime via rquickjs)
├── apps/
│   └── amigo-trommel/               # Editor-App (Svelte + Tauri)
└── docs/
    └── specs/audio/                 # Diese Specs
```

## Implementierungs-Strategie

Trommel ist absichtlich modular aufgebaut, sodass die einzelnen Komponenten in einer sinnvollen Reihenfolge entstehen können:

1. **`amigo-audio-core` Basis-Funktionalität** (Sample-Player, Mixer)
2. **`amigo-trommel-runtime` Voice-API**
3. **Voice-Modus in der Editor-App** (Minimal: Sample-Import + Trigger)
4. **`amigo-audio-core` DSP-Erweiterung** (Modulation, Filter, FX)
5. **Voice-Modus Vollausbau** (Audacity-Style UX)
6. **Strudel-Integration in `amigo-trommel-runtime`**
7. **Pattern-Modus mit Grid-View und Code-View**
8. **Audio-Analyse-Modul**
9. **`amigo-trommel-build` MVP**
10. **MIDI-Integration**
11. **Engine-Integration in amigo-engine**
12. **Polishing und Performance-Optimierung**

Details pro Komponente in den jeweiligen Specs unter „Implementierungs-Reihenfolge".
