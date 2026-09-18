# Electrocardiogram Signal Generator

A Java Swing app that synthesizes realistic ECG waveforms, detects QRS complexes, and classifies cardiac stress — all live, in one window.

Electrocardiogram Signal Generator is a small biosignal-simulation pipeline built for experimenting with synthetic ECG data without a real sensor. It does three things at once: it synthesizes a physiologically realistic ECG trace from a configurable waveform model and animates it on a scrolling plot, it scans that live trace for Q/R/S peaks using a pluggable peak-detection strategy, and it feeds derived vitals (pulse, oxygen saturation, skin conductance) through a self-contained decision-tree classifier that flags stress or a possible cardiac event. A pretrained-equivalent seed dataset ships in the repo, so nothing needs to be labelled before you can try it — though the classifier is intentionally small and meant to be grown from the UI.

**Stack:** Java 11 · Swing · Gson · Maven · JUnit

**Author:** Anish Kuila

## Table of contents
- [What it does](#what-it-does)
- [Architecture](#architecture)
- [Verified](#verified)
- [Install](#install)
- [Quickstart](#quickstart)
- [Project structure](#project-structure)
- [Usage reference](#usage-reference)
- [Signal generation model](#signal-generation-model)
- [Limitations](#limitations)

## What it does

```
EcgParameters ──▶ [EcgCalc: McSharry dynamical model] ──▶ live plot (GraphPanel)
                                                              │
                                              ┌───────────────┴───────────────┐
                                              ▼                               ▼
                                     [QRSDetector: Q/R/S peaks]     [StressIndicator ──▶ ID3 decision tree]
                                                                              │
                                                                    outcome action (ignore / reduce speed /
                                                                              anti-stress)
```

Every generated sample runs through the same fixed pipeline: waveform synthesis never touches classification, and classification never touches waveform synthesis — they're wired together only through the values `GraphPanel` passes along. `og.acm.ecg.EcgCalc` owns signal generation, `no.hig.imt3591.ecg.QRSDetector` owns peak detection, and `no.hig.imt3591.ecg.DecisionMaking` owns classification, each independently swappable.

## Architecture

| Layer | Technology |
|---|---|
| Signal model | `og.acm.ecg.EcgCalc` — Java port of the McSharry ECGSYN dynamical model |
| UI | Java Swing — live plot, parameter editor, log window, export dialog |
| Peak detection | `no.hig.imt3591.ecg.QRSDetector` — quicksort- or simulated-annealing-based R-peak search |
| Classification | `no.hig.imt3591.id3` — generic ID3 decision tree (entropy-based splitting) |
| Data | Gson — reads/writes the JSON training set (`set.json`) |
| Build/test | Maven, JUnit 4 |

```
MainApplication
 ├─ ParameterWindow   (edits EcgParameters)
 ├─ LogWindow          (text log sink)
 └─ GraphPanel
     ├─ EcgCalc                synthesizes ECG samples from EcgParameters
     ├─ QRSDetector            finds Q/R/S points in the live signal
     │   └─ IRDetection (RDetectionFactory: QuickSort- or SimulatedAnnealing-based)
     └─ DecisionMaking         classifies sensor state via an ID3 tree
         ├─ DecisionTreeFactory   loads/saves training data (set.json / initset.json)
         └─ id3.DecisionTree      generic ID3 implementation
```

See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) for the full data-flow breakdown, including the ECGSYN parameterization and the decision-tree internals.

## Verified

Numbers below are from actually building and running this repository, not assumed:

| Check | Result |
|---|---|
| Compiles | Clean on JDK 11 (`javac`, no errors) |
| Unit tests | 4/4 passing (`no.hig.imt3591.id3`: entropy + decision-tree construction/search) |
| App launch | Starts and runs without exceptions |
| Source size | 4,686 lines across 45 `.java` files (42 main, 3 test) |
| Seed training set | 2 labelled examples in `initset.json` (`nothing`, `heartProblem`) — small by design, meant to be extended live from the UI's **Sample** control |
| Outcome classes | 3 (`nothing`, `heartProblem`, `stressed`) |

## Install

Requires JDK 11+ and Maven 3.6+.

```bash
git clone https://github.com/anishneu/electrocardiogram-signal-generator.git
cd electrocardiogram-signal-generator
mvn package
```

`mvn package` compiles the sources, runs the unit tests, and produces `target/electrocardiogram-signal-generator.jar` plus its runtime dependencies in `target/lib/`.

No Maven? See [Usage reference](#usage-reference) for a plain `javac`/`java` build.

## Quickstart

```bash
# Run the app (from the project root)
mvn exec:java
```

Or, after `mvn package`:

```bash
java -cp "target/electrocardiogram-signal-generator.jar;target/lib/*" og.acm.ecg.MainApplication
```

(use `:` instead of `;` as the classpath separator on Linux/macOS)

Always run from the project root — the app reads `RES/img/`, `initset.json`, and `set.json` as plain relative-path files, not bundled resources.

## Project structure

```
.
├── src/main/java/
│   ├── og/acm/ecg/              # Swing UI, ECG signal generator (EcgCalc), parameters, export
│   └── no/hig/imt3591/
│       ├── ecg/                 # QRS/R-peak detection, stress indicator, decision-tree wiring
│       │   └── decisions/       # Sensor observations, decision-tree factory, outcome actions
│       └── id3/                 # Generic ID3 decision-tree implementation (entropy, nodes)
├── src/test/java/
│   └── no/hig/imt3591/id3/      # Unit tests for the decision-tree engine
├── RES/img/                     # Icons used by the UI
├── initset.json                 # Seed training data for the decision tree
├── set.json                     # Working copy of the training data (edited at runtime)
├── pom.xml
└── docs/
    └── ARCHITECTURE.md          # Deeper notes on the signal model and decision engine
```

## Usage reference

| Action | Purpose |
|---|---|
| **Parameters** button | Opens the editor for heart rate, noise, sampling frequency, and P/Q/R/S/T waveform coefficients; the plot updates live |
| **Show log** button | Opens a log of generated values and detector/classifier output |
| **Sample** (Learning panel) | Labels the current sensor state (`nothing` / `heartProblem` / `stressed`) and appends it to `set.json` |
| **Clear** (Learning panel) | Resets `set.json` back to the seed data in `initset.json` |
| Export controls | Saves the generated trace as CSV, tab-delimited, or a custom-delimited text file |

Build/run without Maven (fetches the one runtime dependency directly):

```bash
curl -L -o gson.jar https://repo1.maven.org/maven2/com/google/code/gson/gson/2.11.0/gson-2.11.0.jar
javac -d out -cp gson.jar $(find src/main/java -name "*.java")
java -cp "out;gson.jar" og.acm.ecg.MainApplication
```

(use `:` instead of `;` as the classpath separator on Linux/macOS; in PowerShell, replace `$(find src/main/java -name "*.java")` with `(Get-ChildItem -Path src\main\java -Recurse -Filter *.java | ForEach-Object { $_.FullName })`)

## Signal generation model

`EcgCalc` is a Java port of the **ECGSYN** algorithm (McSharry, Clifford, Tarassenko, Smith — *"A dynamical model for generating synthetic electrocardiogram signals,"* IEEE Trans. Biomed. Eng., 2003):

```
EcgParameters (theta/a/b per P,Q,R,S,T; hrmean/hrstd; flo/fhi; lfhfratio; anoise; seed)
        │
        ▼
Integrate the ECGSYN ODE system (trajectory around a unit circle, z = ECG voltage)
        │
        ▼
RR-interval variability from a Mayer-wave (~0.1 Hz) + respiratory (~0.25 Hz) power spectrum
        │
        ▼
Sampled voltage/time series ──▶ GraphPanel
```

Each of the five waveform extrema (P, Q, R, S, T) is independently tunable via an angular position (`theta`), Gaussian amplitude (`a`), and Gaussian width (`b`), so the generated morphology — not just the heart rate — can be adjusted from the Parameters window while the signal runs.

## Limitations

- The seed training set (`initset.json`) has only 2 labelled examples — the classifier is only as useful as what you capture from the UI afterward.
- Only the ID3 decision-tree engine has unit tests; signal generation and QRS detection have no automated tests.
- Resource loading (`RES/img`, `initset.json`, `set.json`) uses relative file paths rather than classpath resources, so the app must be launched from the project root.
- GUI only — no CLI/headless mode for scripted signal generation or batch export.
- Only verified on Windows with JDK 11; other platforms/JDKs are untested.
- `no.hig.imt3591` originates from an academic project (IMT3591, Høgskolen i Gjøvik) that extended the original signal generator with QRS detection and stress classification; it's included here as-is rather than rewritten.

## License

Apache License 2.0 — see [LICENSE](LICENSE).
