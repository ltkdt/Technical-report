# Signal Processing Methods for WiFi CSI Sensing

Focused brief on Section 3 ("Signal Processing of WiFi Sensing") of the survey *WiFi Sensing with Channel State Information: A Survey* (Ma, Zhou, Wang — ACM Computing Surveys 52(3), Article 46, June 2019), plus the later sections that reuse these methods.

## Question

What signal processing methods does the survey describe for CSI-based WiFi sensing (Section 3), and where are those methods reused in the rest of the paper?

## Executive Summary

1. Section 3 organizes CSI signal processing into three stages: **noise reduction** (3.1), **signal transform** (3.2), and **signal extraction** (3.3). Raw CSI → noise reduction → transform → extraction feeds modeling-based or learning-based algorithms (Section 4).
2. **Noise reduction** has two branches: phase offset removal (phase differences, linear regression) and outlier removal (moving average, median, low-pass, wavelet, Hampel filters, LOF, signal nulling).
3. **Signal transform** provides time-frequency analysis: **FFT** (dominant frequencies), **STFT** (time-resolved spectrograms of dominant frequencies), **DHT** (analytic/complex signal, instantaneous attributes), and **DWT** (multi-resolution: good frequency resolution at low frequencies and good time resolution at high frequencies).
4. **Signal extraction** isolates the target signal: filtering/thresholding (Butterworth BPF/HPF/LPF, ToF/AoA/power thresholds), signal compression (PCA, ICA, SVD, correlation, Euclidean distance), and signal composition (combining CSIs across devices, packets, and frequency bands, e.g., SpotFi, Chronos).
5. Later sections reuse these methods heavily: Section 4 uses transform outputs for physical models (Doppler shift, PDP via IFFT, MUSIC), and Section 5's per-system tables tag every surveyed system with its NR/ST/SE combination (e.g., FallDeFi: Hampel+wavelet filters, DWT+STFT, PCA; RT-Fall: STFT + BPF).
6. A practical selection rule emerges: phase-insensitive detection/recognition can use simple phase differences or amplitude-only processing, while estimation tasks (AoA/ToF localization, respiration rate) need phase-offset removal (linear regression) and frequency-domain analysis (FFT/BPF, peak detection).

## Methodology

- Source: the uploaded survey PDF (OCR text), read directly: Section 3 in full, Section 4 (algorithms that consume Section 3 outputs), and the Section 5 comparison tables that record each system's noise-reduction/transform/extraction pipeline.
- No external sources were consulted; this brief reports the survey's own content.

## Findings

### The processing pipeline

Raw CSI measurements (a 4D tensor: time series of N×M×K complex matrices) contain multipath-channel information mixed with hardware/software distortions: Cyclic Shift Diversity (CSD), Sampling Time Offset (STO), Sampling Frequency Offset (SFO), beamforming effects, plus noise and outliers. Section 3's techniques strip these and isolate the signal components that carry sensing information.

### 3.1 Noise Reduction

**Phase offset removal.** STO/SFO and packet-detection delays corrupt raw CSI phase.
- *Phase differences*: use the phase difference between adjacent time samples or adjacent subcarriers; it cancels offsets under the assumption that offsets are constant across packets/subcarriers. Inaccurate absolute phases, but recovers phase change patterns suitable for detection/recognition classifiers.
- *Linear regression*: for estimation tasks needing accurate phase (AoA/ToF), SpotFi removes STO/SFO by (multiple) linear regression over subcarrier phases.

**Outlier/noise removal.**
- *Moving Average* (incl. weighted WMA and exponentially weighted EWMA) and *Median Filter*: replace each point with the average/median of a sliding neighborhood; remove high-frequency noise.
- *Low-Pass Filter*: removes high-frequency noise, often combined with FFT.
- *Wavelet Filter*: like an LPF but built on the Discrete Wavelet Transform instead of FFT; often combined with DWT.
- *Hampel Filter*: compute window median m and standard deviation σ; if |x−m|/σ exceeds a threshold, replace x with the median (or drop it).
- *Local Outlier Factor (LOF)*: anomaly-detection method; flags points whose local density (via reachability distance to neighbors) is significantly lower than their neighbors'.
- *Signal Nulling*: WiFi-specific; cancels noise signals using hardware (directional antennas) or software (transmit beamforming).

### 3.2 Signal Transform

Time-frequency analysis of a CSI time series; the output frequencies are the frequencies of CSI change patterns (motion/breathing rhythms), not the WiFi carrier frequency.

- **FFT** — finds distinct dominant frequencies; combined with LPF for denoising or band-pass filters to capture target bands. Used when a nearby person is static vs. moving (different dominant frequencies), motion detection, and breathing estimation.
- **STFT (Short-Time Fourier Transform)** — divides the input into equal-length short segments and computes FFT per segment (windowed: X(t,k) = Σ x[n]w[n−t]e^(−jkn)). Represents the series in both time and frequency, so it identifies *when* dominant frequencies change over time. Key trade-off: a long window gives good frequency but poor time resolution; a short window the opposite (no simultaneous optimality of both).
- **DHT (Discrete Hilbert Transform)** — adds a π/2 phase shift to negative FFT frequency components (H[ω] = X[ω]·(−j·sgn(ω))); converts real-valued CSI into its analytic representation (complex helical sequence). Useful for instantaneous attributes of the CSI series.
- **DWT (Discrete Wavelet Transform)** — decomposes the signal into approximation coefficients (low-pass g) and detail coefficients (high-pass h) with downsampling. Gives good frequency resolution for low-frequency signals *and* good time resolution for high-frequency signals simultaneously, overcoming STFT's resolution trade-off. Output can feed a wavelet filter for denoising; preserves mobility information and is more robust than Doppler phase shift (per CARM, refs [98, 99]).

### 3.3 Signal Extraction

**Filtering and thresholding.**
- Band-pass filters capture target bands (e.g., adult resting respiration 12–18 breaths/min → BPF isolates chest-movement effects and rejects motion-induced high frequencies). Inputs to filtering typically come from FFT, DHT, or STFT. Butterworth filters are popular for their monotonic response and quick roll-off.
- High-pass filters remove low-frequency reflections from static objects (e.g., gesture recognition keeps only human-motion-induced changes).
- Time-domain thresholding extracts signals with certain power levels, AoAs, or ToFs; e.g., device-free tracking excludes the direct path by cutting off the shortest-ToF signals (ToFs computed via Power Delay Profile); gesture systems threshold CSI amplitude to discard non-gesture periods.

**Signal compression.** Raw CSI is large (example: 3×3 MIMO, 52 subcarriers, 100 samples, 32-bit values ≈ 187,200 bytes), so dimension reduction removes redundancy:
- *PCA*: orthogonal transform to linearly uncorrelated principal components (via SVD or eigendecomposition of the covariance/correlation matrix).
- *ICA*: assumes a mixture of statistically independent non-Gaussian components; separates them by minimizing mutual information / maximizing non-Gaussianity (kurtosis). PCA/ICA also serve feature extraction and blind signal separation; many components can be discarded.
- *SVD*, and metrics such as self/cross correlation (drop samples when adjacent CSI is highly correlated), Euclidean distance, and distribution functions.

**Signal composition.** Some applications need CSIs combined across multiple devices, packets, or frequency bands: SpotFi composes CSIs from multiple devices and packets for decimeter-level AoA/ToF localization; Chronos composes multiple frequency bands for decimeter localization from a single AP.

### Reuse in later sections

- **Section 4 (algorithms)** consumes Section 3's outputs. Modeling-based algorithms use Doppler phase shift (from FFT of CSI phase over time) for motion/speed estimation and respiration; ToF estimation uses the Power Delay Profile computed by IFFT of CSI (h(t) = Σ αₙ δ(t−τₙ)); AoA estimation uses MUSIC on the CSI covariance matrix. Hybrid pipelines chain a modeling stage with learning: WiSee extracts mobility via Doppler shift then classifies gestures with kNN; WiAG estimates position/orientation features from CFR then applies kNN.
- **Section 4 (learning-based)** explicitly notes that raw CSI is rarely fed to learners; inputs are filtered components from FFT/STFT/DWT, thresholded/subcarrier-selected data, or feature vectors from PCA/ICA/SVD-based feature engineering.
- **Section 5 (application tables)** tags each surveyed system with its Section 3 pipeline (NR/ST/SE). Representative examples:
  - *FallDeFi* (fall detection): NR Hampel + wavelet filters; ST DWT + STFT; SE PCA, interpolation, subcarrier selection, thresholding.
  - *RT-Fall* (fall detection): ST STFT; SE BPF, interpolation, feature extraction, thresholding.
  - *Wi-Dog* (school violence detection): ST DHT + STFT; SE PCA, Butterworth BPF, antenna/subcarrier selection; model uses Doppler shift, wavelet entropy, median filter.
  - *Wi-Sleep* (respiration/apnea): NR Hampel + wavelet; ST DWT; SE interpolation, periodicity-based subcarrier selection, SVD.
  - *CARM* (activity recognition): NR CFO removal; ST DWT; SE thresholding + PCA, then HMM.
  - *WiDance / WiMU* (motion/gesture): ST STFT; SE PCA, antenna selection, Butterworth BPF, thresholding; Doppler shift models.

## Source Notes

| Source | Credibility | Last updated |
| --- | --- | --- |
| Ma, Zhou & Wang, "WiFi Sensing with Channel State Information: A Survey", ACM Computing Surveys 52(3), Article 46 (DOI 10.1145/3310194) — read from the uploaded PDF | 5/5 | June 2019 |

- All content comes from the uploaded survey PDF; no external sources were used.
- Caveat: the survey is from 2019, so it predates newer CSI tooling and deep-learning pipelines (e.g., the CSI-Bench paper in your library uses amplitude-only CSI with learned normalization instead of classical transform pipelines).

## Open Questions

- The survey does not give parameter guidance (window lengths, filter orders, wavelet bases) — those live in the cited primary papers.
- Section 3 does not cover CSI ratio/receiver-diversity preprocessing or newer amplitude sanitization approaches used in recent systems; confirm with newer literature if your pipeline targets modern 802.11ax chipsets.

## Recommendations / Next Steps

- For your CSI project: pick transforms by task dynamics — FFT/BPF for periodic vital signs (respiration), STFT or DWT for non-stationary motion (falls, gestures, activities), with DWT when both time and frequency resolution matter.
- Standardize a preprocessing chain of Hampel (outliers) + Butterworth BPF (band isolation) + PCA (compression) as the baseline, matching the most common NR/ST/SE combos in the survey.
- If phase-based estimation (AoA/ToF localization) is in scope, invest in phase-offset removal (linear regression à la SpotFi) early; if only detection/recognition, amplitude-based processing is sufficient.