# Electrocardiogram Signal Generator

A Java Swing desktop application that synthesizes realistic ECG (electrocardiogram)
waveforms in real time, detects QRS complexes as they are plotted, and classifies
cardiac stress events using a small machine-learning decision tree.

## Features

- **Synthetic ECG generation** — implements the McSharry dynamical model
  (McSharry et al., *"A dynamical model for generating synthetic electrocardiogram
  signals"*, IEEE Trans. Biomed. Eng., 2003) to produce physiologically realistic
  waveforms from a configurable P‑Q‑R‑S‑T morphology.
- **Live plotting** — animates the generated signal on a scrolling graph.
- **Tunable parameters** — heart rate (mean/std), LF/HF ratio, sampling frequency,
  noise amplitude, plot amplitude, random seed, and per-wave (P/Q/R/S/T) morphology
  coefficients, adjustable from the Parameters window while the signal runs.
- **QRS detection** — locates the Q, R and S points in the live signal, using a
  choice of peak-detection strategies (quicksort-based or simulated annealing).
- **Stress / cardiac-event classification** — an ID3 decision tree, trained from a
  JSON sample set, classifies incoming pulse/oxygen/skin-conductance readings as
  `nothing`, `heartProblem`, or `stressed`, and triggers a corresponding action
  (ignore, reduce speed, perform anti-stress routine). New labelled samples can be
  captured from the UI to extend the training set.
- **Export** — save a generated trace as CSV, tab-delimited, or a custom-delimited
  text file.
- **Logging** — a log window records generated values and detector output.

## Project layout

```
electrocardiogram-signal-generator/
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

## Requirements

- JDK 11 or later
- Maven 3.6+

## Building

```bash
mvn package
```

This compiles the sources, runs the unit tests, and produces
`target/electrocardiogram-signal-generator.jar` along with its runtime
dependencies in `target/lib/`.

## Running

The application reads `RES/img/`, `initset.json`, and `set.json` from its current
working directory (they are not bundled as classpath resources), so run it from
the project root.

Via Maven:

```bash
mvn exec:java
```

Or, after `mvn package`, from the project root:

```bash
java -cp "target/electrocardiogram-signal-generator.jar;target/lib/*" og.acm.ecg.MainApplication
```

(use `:` instead of `;` as the classpath separator on Linux/macOS)

## Testing

```bash
mvn test
```

Unit tests cover the ID3 decision-tree engine (`no.hig.imt3591.id3`): entropy
calculation and tree construction/search.

## Usage

1. Launch the app — the main window opens with the live plot area.
2. Click **Parameters** to adjust heart rate, noise, sampling frequency, and
   P/Q/R/S/T waveform coefficients; the plot updates as the signal is generated.
3. Click **Show log** to inspect generated values and detector/classifier output.
4. Use the **Learning** panel to label the current sensor state (`nothing`,
   `heartProblem`, `stressed`) and add it to the training set with **Sample**, or
   reset the training set back to the seed data with **Clear**.
5. Use the export controls to save a generated trace to disk as CSV, tab, or a
   custom delimiter.

## Notes on this repository

The application was originally built and iterated on as a set of NetBeans/IntelliJ
projects; this repository restructures the most recent version (`og.acm.ecg` +
`no.hig.imt3591`) into a standard Maven layout for building, testing, and
distribution without an IDE. The `no.hig.imt3591` package originates from an
academic project (IMT3591, Høgskolen i Gjøvik) that extended the original signal
generator with QRS detection and the stress-classification decision tree.

## License

Apache License 2.0 — see [LICENSE](LICENSE).
