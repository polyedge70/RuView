# HEARTBEAT DETECTION IS NOT VALIDATED

**Status: not validated against real ground truth. Do not rely on any heart-rate value this software produces.**

This document exists because a code review of the heartbeat-sensing modules in this repository found that they have never been shown to detect a real human heartbeat from real WiFi CSI data. Every test in the repo feeds synthetic sine waves to the detectors and then "detects" them. Every runtime path estimates heart rate from heuristic thresholds and hardcoded baselines, not from validated physics.

This is a caveat against the user-facing claim on the landing page ("measure breathing and heart rate"). Breathing detection is plausible and matches the academic literature. **Heart-rate detection on commodity WiFi CSI, at the accuracy implied by the README and marketing, has not been demonstrated by this project.**

## What the review found

1. **No real input data.** There is no code path in this repository that connects real CSI from hardware to the heartbeat detectors. The project's ESP32 firmware lives outside this repo, and no recorded CSI fixtures are present. All heartbeat tests generate their own synthetic signals.

2. **Tests feed synthetic sine waves.**
   - `crates/wifi-densepose-mat/src/detection/heartbeat.rs` — test generates `sin(2πft)` labelled as heartbeat.
   - `crates/wifi-densepose-vitals/src/heartrate.rs` — test feeds four subcarriers with hardcoded "highly coherent" phases. Real cardiac motion does not produce coherent phases across subcarriers.
   - `crates/wifi-densepose-sensing-server/tests/vital_signs_test.rs` — manufactures `0.5*(1 + sin(2πft))` and calls it phase variance.

3. **A test that cannot fail.** `heartbeat.rs::test_detect_heartbeat` asserts only if detection returns `Some`. If detection returns `None`, the test passes silently. The comment on the test reads "Heartbeat detection is challenging, may not always succeed."

4. **The ML path is dead code.** `crates/wifi-densepose-mat/src/ml/vital_signs_classifier.rs::classify_onnx` runs the ONNX session, collects outputs, then `aggregate_mc_outputs` discards them and calls `classify_rules()` regardless. The rule-based fallback estimates heart rate as `70.0 + phase_power * 20.0` — a hardcoded 70 BPM baseline with an arbitrary scaling factor.

5. **The project's own ADRs admit it.** `docs/adr/ADR-013` lists "No heartbeat" in the commodity-WiFi capability matrix and says "Micro-Doppler resolution insufficient for consistent heartbeat."

## Why this matters physically

Cardiac displacement at the body surface is roughly 0.1–0.5 mm. Respiratory displacement is 1–5 mm. That is roughly a 10× difference in mechanical amplitude and a much worse signal-to-noise ratio at the carrier wavelength.

Published CSI heartbeat work (TR-HRV, Phasebeat, etc.) uses Intel 5300 / Atheros NICs with 30–114 subcarriers at high packet rates (often 1000+ pkts/s), with the subject seated still at short range, using specialized phase-correction pipelines. ESP32-S3 exposes around 52 usable subcarriers with noisier phase at lower realistic rates. Whether a 1 Hz cardiac peak can be extracted reliably from ESP32-S3 CSI under realistic deployment conditions is an open research question. This project does not answer it.

## What would make the heartbeat modules real

1. **Paired datasets.** Record CSI from the target hardware simultaneously with a ground-truth cardiac reference (ECG or a validated chest-strap monitor such as a Polar H10). Minimum ~100 sessions across distances, postures, and subjects.

2. **Replace synthetic tests with recorded fixtures.** Delete `generate_heartbeat_signal`, `make_heartbeat_phase_variance`, and every test that manufactures its own input. Tests must fail on `None` when the subject has a heartbeat.

3. **Empirical thresholds.** The hardcoded band-power cutoffs (0.1 / 0.05 / 0.02) and `CONFIDENCE_THRESHOLD = 2.0` are guesses. Derive them from the ROC curve on real data.

4. **A motion/clutter gate.** Body motion swamps cardiac micro-Doppler. The detector must refuse to emit a BPM unless the subject is still and the 0.8–2 Hz peak exceeds the surrounding noise floor by an empirically validated margin.

5. **Fix or remove the ONNX path.** Either implement real MC-Dropout output aggregation, or delete the dead branch so the rule-based estimator does not masquerade as neural inference.

6. **Honest accuracy reporting.** Publish per-condition detection rate (fraction of 30-second windows that yield a BPM), mean absolute error in BPM against ground truth, and false-positive rate (windows with no heartbeat but one is reported anyway).

Until all six are done, no value emitted by these modules should be trusted for any purpose — and absolutely not for any medical, safety, or legal use.

## What this fork changes

This fork does **not** fix the heartbeat modules. The author of this fork cannot validate them without hardware and a human subject. What this fork does is:

- Adds this document.
- Adds a loud warning banner to the root `README.md` linking here.
- Replaces the top-of-file doc comments on every heartbeat source file with an explicit non-validation notice.
- Adds a one-shot runtime `tracing::warn!` to every function that emits a heart-rate value, so any downstream user gets a visible warning in their logs the first time the code produces a number.

No behavior is removed. No APIs are changed. Callers that want to use the heartbeat code can still do so. They just cannot do so without being told that the output is not trustworthy.

## Scope of this critique

This document addresses only the heartbeat-sensing path. The breathing-detection code in `crates/wifi-densepose-sensing-server/src/vital_signs.rs` does real bandpass + FFT processing in a frequency band (0.1–0.5 Hz) where commodity WiFi CSI is known to carry signal. That code is plausible — though it has not been independently validated by this fork either, the physics is not prima facie impossible the way the heartbeat claim is. Pose, presence, and activity code is not addressed here.
