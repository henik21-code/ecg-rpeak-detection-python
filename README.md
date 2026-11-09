# ECG R-Peak Detection on the MIT-BIH Arrhythmia Database

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/henik21-code/ecg-rpeak-detection-python/blob/main/ecg_rpeak_detection.ipynb)

Detection of heartbeats (R-peaks) in single-lead ECG, validated against
expert annotations on 36 records of the MIT-BIH Arrhythmia Database.
Companion to [ecg-filter-ltspice](https://github.com/henik21-code/ecg-filter-ltspice),
which builds the analog filter in LTspice. This project does the digital
filtering and detection on real patient data.

## Result in brief

| Set | Detector | Sensitivity | Precision |
|---|---|---|---|
| Development (10 records) | fixed threshold (baseline) | 90.18 % | 76.13 % |
| Development (10 records) | adaptive (final) | 97.55 % | 97.15 % |
| Unseen (26 records) | adaptive (final) | 98.90 % | 99.96 % |

Precision meets the 99 % target on unseen data. Sensitivity misses it by
about 0.1 percentage point. The detector was not retuned after the
unseen test, so that the test stays independent.

## 1. Specification

| Parameter | Target |
|---|---|
| Sensitivity (real beats found / real beats) | above 99 % |
| Precision (real beats / beats reported) | above 99 % |
| Matching tolerance | 150 ms around each expert label |

## 2. Data

MIT-BIH Arrhythmia Database (PhysioNet): 48 half-hour two-channel
ambulatory ECG recordings sampled at 360 Hz, with beat-by-beat expert
annotations. This project uses lead MLII and the first 5 minutes of each
record. The data were recorded in the USA, so mains interference appears
at 60 Hz (the raw spectrum shows the spike at 60 Hz, not 50 Hz).

Records are split into two sets:

- **Development set (10 records):** 100, 101, 103, 105, 108, 119, 200,
  203, 207, 219. These include noisy and difficult records and were
  inspected while designing the detector.
- **Unseen set (26 records):** 106, 115, 116, 121, 122, 123, 124, 201,
  202, 205, 209, 210, 212, 213, 214, 215, 220, 221, 222, 223, 228, 230,
  231, 232, 233, 234. They were not used for any design decision.

## 3. Method

**3.1 Signal conditioning.** The raw spectrum shows baseline drift below
0.5 Hz, the ECG band up to about 40 Hz, and a sharp mains spike at 60 Hz.
The signal is filtered with a 4th-order Butterworth band-pass (0.5 to
40 Hz) and a 60 Hz notch (Q = 30). The filters are applied forward and
backward (zero phase), so they add no delay. The magnitude response is
then applied twice, which moves the effective -3 dB points inward.

**3.2 Baseline detector.** Upward peaks in the filtered signal with a
fixed prominence of 0.5 mV and at least 250 ms between peaks.

**3.3 Adaptive detector (simplified Pan-Tompkins).**
1. Band-pass the raw signal at 5 to 15 Hz to keep fast QRS slopes and
   suppress slow T-waves.
2. Square the derivative and smooth it with a 150 ms moving average.
3. Threshold at 30 % of the local 95th percentile of this signal (4 s
   window), so the threshold follows the signal level. Candidates must
   be at least 250 ms apart.
4. Place each beat at the largest deflection (upward or downward) of the
   filtered signal within 100 ms of the candidate.

**3.4 Evaluation.** A detection is a true positive (TP) if it lies within
150 ms of an expert beat label, and each label is matched once. Unmatched
labels are false negatives (FN). Unmatched detections are false positives
(FP). Sensitivity = TP / (TP + FN). Precision = TP / (TP + FP).

## 4. Results

**Development set.** The baseline detector fails on records with tall
T-waves (it counts a second beat on each T-wave, for example record 103)
and on records where beats point downward (record 108). The adaptive
detector fixes both: records 103, 108 and 119 reach 100 % sensitivity
and 100 % precision.

**Unseen set.** 98.90 % sensitivity and 99.96 % precision over 26
records. Most records reach 100 %. The weakest are 228 (86.0 %
sensitivity), 210 (96.0 %) and 222 (96.5 %). The per-record tables are
in the notebook.

## 5. Error analysis

The two weakest records, 228 (unseen) and 207 (development), were
inspected around their first missed beat.

- **Record 228:** 49 of 350 beats missed. The record mixes tall beats
  (about 2.5 mV) with small beats (about 0.5 to 0.7 mV). The threshold
  follows the local level set by the tall beats, and QRS energy grows
  roughly with the square of the amplitude, so small beats between tall
  ones fall below it.
- **Record 207:** 37 of 268 beats missed. Some beats are wide with slow
  slopes and carry little energy in the 5 to 15 Hz band, so they are
  missed.

These explanations come from a few inspected segments, not from a
classification of every missed beat.

## 6. Limitations

- 36 of the 48 records were tested, using the first 5 minutes and lead
  MLII only. Not tested: 102, 104, 107, 217 (paced rhythms, usually
  excluded) and 109, 111, 112, 113, 114, 117, 118, 208.
- Beat detection only, with no classification of beat type.
- Offline processing: zero-phase filters and whole-record statistics. A
  real-time device would need causal filters and would add delay.
- The parameters (5 to 15 Hz band, ratio 0.3, 4 s window) were chosen by
  inspecting the development records, not by a systematic search.
- The sensitivity target was missed by about 0.1 percentage point on the
  unseen set.

## 7. Next steps

1. Tune the threshold ratio on a training subset with cross-validation
   and report the result on held-out records.
2. Normalize each beat's energy by its neighbours to handle mixed beat
   sizes.
3. Compare against an open-source reference detector (for example the
   XQRS detector in the `wfdb` package).
4. Build a causal, real-time version and measure its delay.

## 8. How to run

Click the "Open In Colab" badge, then Runtime, then Run all (about 3
minutes). An internet connection is needed because the data are
downloaded from PhysioNet. Dependencies: wfdb, numpy, scipy, pandas,
matplotlib.

## 9. References

- Moody GB, Mark RG. The impact of the MIT-BIH Arrhythmia Database.
  IEEE Engineering in Medicine and Biology 20(3):45-50 (2001).
- Goldberger AL et al. PhysioBank, PhysioToolkit, and PhysioNet:
  components of a new research resource for complex physiologic signals.
  Circulation 101(23):e215-e220 (2000).
- Pan J, Tompkins WJ. A real-time QRS detection algorithm. IEEE
  Transactions on Biomedical Engineering 32(3):230-236 (1985).
