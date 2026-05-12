---
title: amigo-trommel — Pattern-Modus
status: draft
phase: 1
parent: amigo-trommel.md
last_updated: 2026-05-12
---

# Pattern-Modus

Der Pattern-Modus ist die zweite Stufe im Trommel-Workflow. Hier werden Patterns aus Voices komponiert. Patterns sind Strudel-basiert und haben zwei Sichten: Grid-View (klickbar) und Code-View (textbasiert).

## Pattern-Konzept

Ein Pattern ist ein Strudel-Programm, das zyklisch ausgewertet wird und Voice-Trigger-Events erzeugt. Patterns werden als TypeScript-Module gespeichert (`patterns/<name>.ts`).

### Beispiel

```typescript
import { stack, s, note } from '@strudel/core';
import { kick, snare, hihat, bass } from '../voices/drums';

export default stack(
  s("bd*4").gain(0.8),                    // Kick auf jedes Viertel
  s("~ sd ~ sd"),                         // Snare auf 2 und 4
  s("hh*8"),                              // Hi-Hats auf jedes Achtel
  note("c2 c2 e2 g2").s("bass").slow(2)   // Bass-Linie, halbe Geschwindigkeit
);
```

## Strudel als Pattern-Sprache

amigo-trommel verwendet [Strudel](https://strudel.cc) direkt als Pattern-Sprache. Strudel ist eine JavaScript-Portierung von TidalCycles, MIT-lizenziert.

### Warum Strudel

- **Bewährt:** Strudel und TidalCycles haben aktive Communities und 15+ Jahre Erfahrung in Cycle-basierter Pattern-Komposition
- **Mächtig:** Polyrhythmik, Transformationen, generative Funktionen, Mini-Notation für kompakte Patterns
- **Type-System-fähig:** Strudel-Funktionen können mit TypeScript-Typen verwendet werden
- **Live-Coding-optimiert:** Code-Änderungen werden im nächsten Cycle wirksam, kein Stop-Edit-Start

### Strudel-Integration in Trommel

Strudel wird direkt als npm-Dependency verwendet. Trommel ergänzt:

- **Voice-Output-Target:** Strudel-Sample-/Sound-References (`s("bd")`) werden auf Trommel-Voices gemappt
- **Type-Definitions:** TS-Typen für Voice-IDs, Parameter, Strudel-Methoden
- **Pattern-Analyse:** Statische Analyse für Grid-View-Kompatibilität
- **Time-Aware-Bibliotheken:** Zugriff auf Trommel-Voice-Pool, Tempo-Sync, etc.

### Voice-Referenzen in Strudel

Strudel-Sounds (`s("name")`) werden auf Trommel-Voices gemappt. Drei Wege:

```typescript
// 1. Direkte Voice-Referenz über Import
import { kick } from '../voices/drums';
stack(kick.trigger()); // explizit getriggert

// 2. Strudel-Sound-Notation mit Voice-Pool-Lookup
s("kick");             // sucht Voice 'kick' im Voice-Pool
s("kick snare hihat"); // sequenziell

// 3. Strudel-Aliases
s("bd")                // standard alias für 'kick'
s("sd")                // 'snare'
s("hh")                // 'hihat'
```

Voice-Pool-Lookup nutzt die exportierten Namen aus den `voices/`-Files.

## Zwei Sichten

Pattern-Modus zeigt eine von zwei Sichten pro Track:

- **Grid-View** — visuelles Step-Grid für klick-basiertes Editing
- **Code-View** — textbasierter Strudel-Editor

Beide Sichten sind **zwei Sichten auf dieselbe Datenquelle** (den Strudel-Code). Es gibt keine separaten „Grid-Daten" und „Code-Daten" — Edits in einer Sicht propagieren in die andere.

### Wechsel zwischen Sichten

- **Toggle pro Track:** Jeder Track im Pattern hat einen Grid/Code-Toggle
- **Default:** Neu erstellte Tracks starten in Grid-View
- **Auto-Switch:** Bei Code-Änderungen, die nicht grid-darstellbar sind, schaltet die UI auf Code-View und zeigt Hinweis

## Grid-View

### Step-Grid-UI

```
        Step:  1   2   3   4   5   6   7   8   9  10  11  12  13  14  15  16
        ────────────────────────────────────────────────────────────────────
Kick    ▶ █   ·   ·   ·   █   ·   ·   ·   █   ·   ·   ·   █   ·   ·   ·
Snare   ▶ ·   ·   ·   ·   █   ·   ·   ·   ·   ·   ·   ·   █   ·   ·   ·
HiHat   ▶ █   ·   █   ·   █   ·   █   ·   █   ·   █   ·   █   ·   █   ·
```

Jede Zeile ist ein Track, jede Spalte ein Step. Klick toggelt einen Step.

### Steps und Pattern-Länge

- **Step-Auflösung:** Wählbar pro Pattern: 8, 16, 32, 64
- **Pattern-Länge in Cycles:** Default 1 Cycle, einstellbar (Strudel-`cycles`)
- **Sub-Steps:** Per Right-Click oder Drag in eine Zelle → öffnet Sub-Step-Menü ([bd bd], etc.)

### Velocity pro Step

- Step kann zusätzliche Velocity haben (0–127 oder 0.0–1.0)
- Visualisierung: Step-Höhe oder Farbintensität
- Editing: Vertikales Drag in der Step-Zelle oder Right-Click-Menü

### Probability pro Step

- Step kann eine Wahrscheinlichkeit haben (0–100%)
- Visualisierung: Step-Transparenz oder Punkt-Symbol
- Editing: Modifier-Klick (z. B. Shift+Click setzt 50%)

### Micro-Timing

- **Swing:** Pattern-globaler Swing-Wert (0–100%)
- **Humanize:** Zufällige Timing-Variation (0–10ms)
- **Per-Step-Offset:** Einzelne Steps können um ±50% des Step-Intervalls versetzt werden

### Generierter Strudel-Code

Grid-View generiert immer äquivalenten Strudel-Code:

| Grid | Strudel |
|------|---------|
| Kick auf Steps 1,5,9,13 | `s("bd ~ ~ ~ bd ~ ~ ~ bd ~ ~ ~ bd ~ ~ ~")` oder kompakt `s("bd*4")` |
| Snare auf 5,13 | `s("~ ~ ~ ~ sd ~ ~ ~ ~ ~ ~ ~ sd ~ ~ ~")` oder `s("~ sd ~ sd")` |
| Velocity 0.8 auf Kick | `s("bd*4").gain(0.8)` |
| Probability 50% auf HiHat | `s("hh?*8")` |

Trommel generiert *bevorzugt die kompakte Notation* (z. B. `bd*4` statt `bd ~ ~ ~ bd ~ ~ ~`), um den Code lesbar zu halten.

### Grid-Kompatibilität

Ein Strudel-Pattern ist **grid-kompatibel**, wenn es nur Features verwendet, die in einem Step-Grid darstellbar sind.

**Grid-kompatibel:**

- Mini-Notation mit Steps und Ruhen: `s("bd ~ sd ~")`
- Repetition: `s("bd*4")`
- Konkatenation: `s("bd!2")` (zweimal abspielen)
- Sub-Steps: `s("[bd bd] sd")`
- Velocity: `.gain(n)`
- Probability per Step: `s("bd?")`
- Stack mehrerer Tracks: `stack(s("bd*4"), s("hh*8"))`
- Slow/Fast als Pattern-Länge: `.slow(2)`, `.fast(2)` (verändert globale Cycle-Länge)

**Nicht grid-kompatibel:**

- Transformationen über Cycles: `.every(n, fn)`
- Generative Funktionen: `.degrade()`, `.sometimes(fn)`, `.choose([...])`
- Polymetrik mit unterschiedlich langen Sub-Patterns
- Funktions-Komposition: `.fast(2).rev().degrade()`
- Custom Strudel-Erweiterungen

### Track-Status bei Inkompatibilität

Wenn ein Track nicht-grid-kompatible Features enthält:

- Grid-Toggle ist deaktiviert
- UI zeigt: „Dieser Track enthält Strudel-Features, die nicht im Grid darstellbar sind. Nur Code-Editing möglich."
- Andere Tracks im selben Pattern bleiben in Grid-View nutzbar (sofern grid-kompatibel)

## Code-View

### Editor

CodeMirror 6 mit Strudel-Mode:

- **Syntax-Highlighting** für Strudel-Mini-Notation und JS
- **Autocomplete** für Voice-Namen aus dem Voice-Pool
- **Inline-Errors** bei Syntax-Fehlern
- **Live-Eval** während Playback (Strudel-Standard)

### Voice-Pool-Autocomplete

Beim Tippen von `s("` schlägt CodeMirror alle verfügbaren Voices vor:

```
s("|
   ├── kick
   ├── snare
   ├── hihat
   ├── laser
   └── coin
```

### Live-Reload während Playback

Strudel-Standard: Code wird bei jedem Speichern (oder konfigurierbarem Trigger) evaluiert. Änderungen werden im nächsten Cycle wirksam.

Tasten-Bindings (Strudel-konform):

- `Ctrl+Enter` / `Cmd+Enter`: Code evaluieren
- `Ctrl+.` / `Cmd+.`: Stop

### Pattern-Visualisierung

Neben dem Code-Editor:

- **Multi-Track-Waveform-View** (Audacity-Style): jede Voice als eigener Track, mit Live-Audio-Visualisierung
- **Cycle-Indicator:** vertikale Linie zeigt aktuelle Position im Cycle
- **Solo/Mute-Buttons** pro Track

## Pattern-Modus UI

### Layout

```
┌──────────────────────────────────────────────────────────────────────┐
│ [Voice-Modus]  [Pattern-Modus]                                       │
├──────────────────┬───────────────────────────────────────────────────┤
│ Pattern-Browser  │ Pattern-Editor: combat-beat                       │
│                  │                                                   │
│ • intro-loop     │ ┌───────────────────────────────────────────┐    │
│ • combat-beat ●  │ │ [Grid] [Code]  Tempo: 120 BPM  Swing: 0  │    │
│ • outro          │ ├───────────────────────────────────────────┤    │
│                  │ │  Step:  1  2  3  4  5  6  7  8 ...        │    │
│                  │ │  Kick  ▶ █  ·  ·  ·  █  ·  ·  ·           │    │
│ [+ New Pattern]  │ │  Snare ▶ ·  ·  ·  ·  █  ·  ·  ·           │    │
│                  │ │  HiHat ▶ █  ·  █  ·  █  ·  █  ·           │    │
│                  │ │                                            │    │
│                  │ │  [+ Track]                                 │    │
│                  │ └───────────────────────────────────────────┘    │
│                  │                                                   │
│                  │ Mixer ──────────────────────────────────────      │
│                  │ Kick   [▬▬▬▬░░] Solo/Mute   FX: [Comp] [EQ]      │
│                  │ Snare  [▬▬▬░░░] Solo/Mute   FX: [Reverb]         │
│                  │ HiHat  [▬▬▬▬░░] Solo/Mute                        │
│                  │ Master [▬▬▬▬▬░]              FX: [Limiter]       │
├──────────────────┴───────────────────────────────────────────────────┤
│ [▶ Play] [⏸ Pause] [⏹ Stop] [⟲ Loop]  Cycle: 1/4  MIDI: ●           │
└──────────────────────────────────────────────────────────────────────┘
```

### Grid-View-Layout

```
┌───────────────────────────────────────────────────────────────┐
│ Tempo: 120 BPM     Steps: 16     Swing: 0%     Humanize: 0ms │
├───────────────────────────────────────────────────────────────┤
│        Step:  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 │
│        ──────────────────────────────────────────────────────│
│ Kick   [G][C] █  ·  ·  ·  █  ·  ·  ·  █  ·  ·  ·  █  ·  ·  ·│
│ Snare  [G][C] ·  ·  ·  ·  █  ·  ·  ·  ·  ·  ·  ·  █  ·  ·  ·│
│ HiHat  [G][C] █  ·  █  ·  █  ·  █  ·  █  ·  █  ·  █  ·  █  ·│
│ [+ Track]                                                     │
└───────────────────────────────────────────────────────────────┘
```

`[G][C]` = Grid/Code-Toggle pro Track

### Code-View-Layout

```
┌───────────────────────────────────────────────────────────────────────┐
│ Tempo: 120 BPM                                                        │
├──────────────────────────────────────────┬────────────────────────────┤
│ Editor:                                  │ Waveform-View:             │
│                                          │                            │
│ import { stack, s } from '@strudel...    │ Kick  ▒▓▓▒▒▒▒▒▒▓▓▒▒▒▒▒    │
│ import { kick, snare } from '...';       │ Snare ░░░░▓▓▒▒░░░░▓▓▒▒    │
│                                          │ HiHat ▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒    │
│ export default stack(                    │ Master▓▒▓▒▓▒▓▒▓▒▓▒▓▒▓▒    │
│   s("bd*4").gain(0.8),                   │                            │
│   s("~ sd ~ sd"),                        │ Solo/Mute pro Track        │
│   s("hh*8").every(4, rev)                │                            │
│ );                                       │ Cycle: ▮▮▮▯▯▯▯▯ 25%        │
│                                          │                            │
└──────────────────────────────────────────┴────────────────────────────┘
```

## Tracks

### Track-Konzept

Ein Track ist eine einzelne Voice-Reihe in einem Pattern. Tracks haben:

- **Voice-Referenz** (welche Voice triggert dieser Track)
- **Sicht** (Grid oder Code)
- **Strudel-Code** (immer vorhanden)
- **Mixer-Settings** (Gain, Pan, Solo, Mute)
- **Optional: Track-spezifische FX-Chain** (Bus-Level FX)

### Track-Hinzufügen

```
[+ Track]
    ↓
Voice-Picker: [kick] [snare] [hihat] [laser] [+ from Voice-Pool]
    ↓
Default-Pattern: leeres Grid mit Anzahl Steps des Patterns
```

### Track-Reihenfolge

Tracks können per Drag-and-Drop umgeordnet werden. Reihenfolge entspricht der Reihenfolge in `stack(...)` im generierten Code.

## Mixer

Jedes Pattern hat einen Mixer mit:

- **Pro Track:** Gain, Pan, Solo, Mute, optionale Bus-FX
- **Master:** Master-Gain, optionale Master-FX (Compressor, EQ, Limiter)
- **Send-Busses (optional):** Reverb-Bus, Delay-Bus für gemeinsame Effekte

### Mixer-Persistenz

Mixer-Einstellungen werden als Teil des Pattern-TS-Files gespeichert:

```typescript
import { stack, s } from '@strudel/core';
import { definePattern } from '@amigo/trommel';

export default definePattern({
  pattern: stack(
    s("bd*4"),
    s("~ sd ~ sd"),
    s("hh*8")
  ),
  tempo: 120,
  mixer: {
    tracks: {
      kick: { gain: 1.0, pan: 0 },
      snare: { gain: 0.8, pan: 0, fx: [reverb({ mix: 0.2 })] },
      hihat: { gain: 0.7, pan: 0.2 }
    },
    master: {
      gain: 0.9,
      fx: [compressor({ threshold: -10, ratio: 4 })]
    }
  }
});
```

## Playback-Verhalten

### Im Pattern-Modus

- **Play-Button:** Aktives Pattern wird kontinuierlich geloopt
- **Code-Änderungen:** Werden im nächsten Cycle wirksam (Strudel-Standard)
- **Voice-Änderungen** (vom Voice-Modus aus): Werden bei nächstem Trigger wirksam
- **Mixer-Änderungen:** Live wirksam mit Parameter-Smoothing

### Cycle-Indikator

Während Playback zeigt eine visuelle Cycle-Position:

- Im Grid-View: vertikale Linie über dem Grid wandert durch Steps
- Im Code-View: Highlight auf aktuell evaluiertem Code-Bereich (wenn Strudel das unterstützt)

## Pattern-zu-Pattern-Konvertierung

Patterns aus dem Grid-View können als Strudel-Code „eingefroren" werden — d. h. das Pattern-File wird zu reinem Strudel-Code ohne Grid-Repräsentation. Der umgekehrte Weg (Strudel-Code zu Grid) funktioniert automatisch, solange der Code grid-kompatibel ist.

## Phase 2 — Erweiterungen

Diese Features sind in Phase 1 *nicht* enthalten:

- **Song-Mode** mit Pattern-Chaining (linearer Song aus mehreren Patterns)
- **Pattern-Variationen** (A/B/C-Variants per Pattern)
- **Real-time Pattern-Switching** mit Beat-Sync
- **Pattern-Library** als Browse-View über alle Patterns im Projekt
- **Pattern-Templates** (Genre-Presets: House, DnB, Chiptune, etc.)
- **AI-Pattern-Suggestion** basierend auf existierenden Patterns

## Implementierungs-Reihenfolge

Für die Phase-1-Implementation:

1. Grid-View mit Basis-Step-Toggling (ohne Velocity/Probability)
2. Strudel-Integration und Code-Generation aus Grid
3. Code-View mit CodeMirror und Strudel-Eval
4. Bidirektionale Sync zwischen Grid und Code
5. Auto-Detection von Grid-Kompatibilität
6. Velocity per Step
7. Probability per Step
8. Sub-Steps und Mini-Notation-Editing im Grid
9. Multi-Track-Waveform-View
10. Mixer mit Gain/Pan/Solo/Mute
11. Bus-FX auf Tracks
12. Master-FX
13. Micro-Timing (Swing, Humanize)
14. MIDI-Sync und MIDI-Trigger
15. Pattern-Import (MIDI, Strudel-Snippets)
