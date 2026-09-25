# Reference Audio & Synthesis Baselines: Redder TTS Quality Suite

## 1. Purpose
To detect regressions in speech synthesis quality, phonetic formatting, INT8 model quantization artifacts, or audio clipping, Redder maintains a golden dataset of test phrases paired with target physical audio baselines.

These baselines run as part of CI audio regression tests using mock or offline Piper/ONNX synthesis benchmarks.

---

## 2. Audio Specifications
- **Sampling Rate**: $22,050\text{ Hz}$ ($\pm 0\text{ Hz}$)
- **Bit Depth**: 16-bit signed PCM (Little-Endian)
- **Channels**: 1 (Mono)
- **Target RMS Signal Level**: $-18\text{ dBFS}$ to $-24\text{ dBFS}$
- **Peak Signal Level**: Maximum $-1.0\text{ dBFS}$ (Zero clipping)
- **Silence Threshold**: Initial/trailing silence $<150\text{ ms}$

---

## 3. Golden Text Inputs & Tolerance Baselines

| ID | Input Text | Word Count | Expected Duration (s) | Target Speaking Rate | Spectral Centroid Bounds | Key Phonetic Verification |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **`REF-001`** | `"Android architecture requires clean separation of concerns."` | 7 | $2.3\text{s} \pm 0.3\text{s}$ | $\sim160\text{ WPM}$ | $1,200\text{ Hz} - 2,800\text{ Hz}$ | Verifies syllable stress on `"architecture"` and `"separation"`. |
| **`REF-002`** | `"User cool coder 42 replies: What about memory overhead?"` | 9 | $2.9\text{s} \pm 0.4\text{s}$ | $\sim155\text{ WPM}$ | $1,200\text{ Hz} - 2,800\text{ Hz}$ | Verifies username formatting (`"cool coder"`) and question pitch intonation at `"overhead?"`. |
| **`REF-003`** | `"The model parameters are eighty-two million, quantized to INT8."` | 9 | $3.4\text{s} \pm 0.4\text{s}$ | $\sim150\text{ WPM}$ | $1,100\text{ Hz} - 2,900\text{ Hz}$ | Verifies numerical and acronym pronunciation (`"eighty-two"`, `"INT-eight"`). |
| **`REF-004`** | `"Post by system engineer: Warning, thermal throttling will degrade RTF below real time."` | 13 | $4.6\text{s} \pm 0.5\text{s}$ | $\sim160\text{ WPM}$ | $1,200\text{ Hz} - 2,800\text{ Hz}$ | Verifies dialogue prefix cadence (`"Post by..."`) and acronym G2P (`"R-T-F"`). |
| **`REF-005`** | `"Yes! Exactly. Couldn't agree more."` | 5 | $1.7\text{s} \pm 0.3\text{s}$ | $\sim140\text{ WPM}$ | $1,300\text{ Hz} - 3,000\text{ Hz}$ | Verifies contractions (`"Couldn't"`) and punctuation pause markers. |

---

## 4. Automated Regression Metric Formulae

In automated audio test runs, the generated PCM buffer is evaluated against these numerical boundaries:

### A. Real-Time Factor (RTF)
$$\text{RTF} = \frac{T_{\text{inference}}}{T_{\text{audio}}}$$
- **Pass Criteria**: $\text{RTF} \le 0.40$ on reference test harness.
- **Fail Criteria**: $\text{RTF} > 0.50$ triggers an immediate build failure.

### B. Signal-to-Quantization-Noise Ratio (SQNR)
Comparing unquantized FP32 reference PCM to INT8 quantized mobile output:
$$\text{SQNR} = 10 \cdot \log_{10} \left( \frac{\sum x[n]^2}{\sum (x[n] - y[n])^2} \right)$$
- **Pass Criteria**: $\text{SQNR} \ge 28\text{ dB}$.
- A drop below $28\text{ dB}$ indicates aggressive weight clamping or dynamic range distortion during quantization.

### C. Silence & Clipping Bounds
- Maximum amplitude value: $|A_{\max}| \le 32,000$ (below $32,767$ int16 max).
- Leading silence duration: $t_{\text{lead}} \le 120\text{ ms}$.
