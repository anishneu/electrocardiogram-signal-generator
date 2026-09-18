# Architecture

## Overview

The application has two largely independent subsystems wired together through
`og.acm.ecg.GraphPanel`:

1. **Signal generation and UI** (`og.acm.ecg`) — synthesizes and plots the ECG
   waveform.
2. **Detection and classification** (`no.hig.imt3591`) — analyzes the live
   waveform and classifies the physiological state it represents.

```
MainApplication
 ├─ ParameterWindow  (edits EcgParameters)
 ├─ LogWindow         (text log sink)
 └─ GraphPanel
     ├─ EcgCalc               synthesizes ECG samples from EcgParameters
     ├─ QRSDetector           finds Q/R/S points in the live signal
     │   └─ IRDetection (RDetectionFactory: QuickSort- or SimulatedAnnealing-based)
     └─ DecisionMaking        classifies sensor state via an ID3 tree
         ├─ DecisionTreeFactory  loads/saves training data (set.json / initset.json)
         └─ no.hig.imt3591.id3.DecisionTree  generic ID3 implementation
```

## Signal generation — `og.acm.ecg.EcgCalc`

`EcgCalc` is a Java port of the classic **ECGSYN** algorithm described in:

> McSharry, P.E., Clifford, G.D., Tarassenko, L., Smith, L.A.,
> "A dynamical model for generating synthetic electrocardiogram signals,"
> *IEEE Transactions on Biomedical Engineering*, 50(3), 2003.

The model integrates a system of ODEs representing a trajectory that traces the
P, Q, R, S, and T waves as it circles a unit circle in (x, y) with a
z-coordinate representing the ECG voltage. Each extremum is parameterized by:

- `theta[i]` — angular position on the circle (wave timing)
- `a[i]` — Gaussian amplitude
- `b[i]` — Gaussian width

RR-interval variability is generated from a power spectrum with a low-frequency
(Mayer wave, ~0.1 Hz) and high-frequency (respiratory, ~0.25 Hz) component,
controlled by `flo`/`fhi`, their standard deviations, and `lfhfratio`. Heart rate
mean/std (`hrmean`, `hrstd`), sampling frequency (`sfecg`, internal `sf`), noise
amplitude (`anoise`), and plot amplitude round out the parameter set exposed by
`EcgParameters` and edited via `ParameterWindow`.

## QRS detection — `no.hig.imt3591.ecg`

`QRSDetector` buffers the last `numObservations` plotted points and, on each
batch, asks an `IRDetection` implementation (selected by `RDetectionFactory`) to
locate the R peak — either by sorting observations (`QuickSort`) or via a
simulated-annealing search (`SimulatedAnnealing`) — from which Q and S are
derived. `IRDetection` receives frequency/timestamp data usable for further
analysis (e.g. heart-rate estimation).

## Stress classification — decision tree

`StressIndicator` tracks short-term variability of pulse, oxygen saturation, and
skin conductance (via `VariabilityIdentifier`) and forwards each reading to
`DecisionMaking`, a singleton that:

1. Wraps the reading as a `SensorObservation`.
2. Searches an ID3 `DecisionTree<SensorObservation>` (built by
   `DecisionTreeFactory` from `set.json`) for a matching `ITreeResult`.
3. Invokes the resulting outcome — `IgnoreActionOutput`, `ReduceSpeedOutput`, or
   `PerformAntiStressOutput` — each of which can act on the `GraphPanel`.

`no.hig.imt3591.id3` is a small, reusable ID3 decision-tree implementation:
`Observation`/`AttributeType`/`@Attribute` describe a labelled sample and its
fields via reflection, `Entropy` computes information gain, and
`DecisionTree` recursively splits observations into `AttributeNode`/`LeafNode`
subclasses (`Alpha*` for categorical, `Beta*` for continuous attributes) until a
pure or empty partition is reached.

The training set (`set.json`) can be extended live from the UI: labelling the
current reading and clicking **Sample** appends a `LearningObservation` to the
file; **Clear** restores it from the shipped seed data (`initset.json`).

## Data flow summary

```
EcgParameters --> EcgCalc --> GraphPanel (plot)
                                 │
                                 ├─> QRSDetector --> Q/R/S points, frequency
                                 │
                                 └─> StressIndicator --> DecisionMaking --> outcome action
                                                              │
                                                       DecisionTreeFactory <--> set.json
```
