# ADR-001: Selection of Piper ONNX over Kokoro-82M for Initial Android Release

## Status
Accepted

## Date
2026-09-20

## Context
Redder requires an on-device neural Text-to-Speech (TTS) synthesis engine running on Android devices ranging from budget/mid-tier (e.g., Snapdragon 695 / 7-series) to flagship silicon (e.g., Snapdragon 8 Gen 2/3, Google Tensor). The audio playback pipeline must synthesize conversational multi-speaker dialogue in real time without causing severe thermal throttling, background app kills by Android's Low Memory Killer (LMK), or unacceptable latency.

Two primary open-weights neural TTS architectures were evaluated:
1. **Kokoro-82M**: An 82-million parameter StyleTTS2/ISTFT-based model capable of remarkable naturalness and expressiveness.
2. **Piper (VITS-based)**: A compact end-to-end neural TTS architecture optimized for embedded hardware and edge devices.

Key technical criteria:
- **Zero Native C++ / NDK G2P Dependency**: Acoustic models do not synthesize raw English characters directly; they expect International Phonetic Alphabet (IPA) phonemes. The engine must resolve Grapheme-to-Phoneme (G2P) on Android without brittle native compilation toolchains.
- **Real-Time Factor (RTF)**: $\text{RTF} = \frac{\text{Synthesis Time}}{\text{Audio Duration}}$. To sustain gapless multi-speaker conversation without stutter or buffer underrun, RTF must stay comfortably below $0.4\times$ on mid-tier mobile CPUs.
- **Memory Footprint & APK Size**: Peak Resident Set Size (RSS) must remain under $200\text{ MB}$, and model asset sizes per voice profile must not bloat the base APK beyond acceptable distribution limits ($<40\text{ MB}$).

## Decision
We select **Piper ONNX (INT8 quantized)** as the default synthesis engine for the initial production release of Redder. The architecture will expose a generic `TtsEngine` abstraction to enable Kokoro-82M as an optional high-fidelity download in a subsequent milestone.

## Alternatives Considered

### Kokoro-82M (ONNX Runtime Mobile)
- **Pros**: Outstanding audio fidelity, prosody, and emotional nuance.
- **Cons**:
  1. *Missing Mobile G2P*: Kokoro requires an external G2P phonemizer (such as `espeak-ng` or `misaki`). Android has no built-in phonemizer. Integrating `espeak-ng` requires cross-compiling native C/C++ libraries via the Android NDK, packaging multiple ABI `.so` binaries (`arm64-v8a`, `armeabi-v7a`, `x86_64`), and managing native dictionary tables ($>15\text{ MB}$).
  2. *Inference Cost on Mid-tier Devices*: At 82M parameters, FP16/FP32 models take between $160\text{ MB}$ and $330\text{ MB}$ of storage. In INT8, the ONNX model is $\sim85\text{ MB}$. On ARM NEON CPU threads, RTF on mid-tier SoCs hovers between $0.8\times$ and $1.4\times$ (often slower than real-time), causing audio dropouts unless paired with Qualcomm QNN or specialized NPU drivers.
  3. *Thermal & Memory Pressure*: Sustained Kokoro inference pushes CPU cores to maximum frequencies, heating devices and triggering Android LMK when combined with ExoPlayer and Jetpack Compose.
- **Rejected for v1**: The NDK cross-compilation risk and CPU thermal profile introduce too much fragility for initial deployment.

### Android System TTS (`android.speech.tts.TextToSpeech`)
- **Pros**: Pre-installed on device, zero APK size overhead, native system maintenance.
- **Cons**: Severe lack of voice variety (often only 1-2 system voices per locale), robotic prosody, inability to control custom voice embeddings per speaker, and high inconsistency across device manufacturers (Samsung vs. Xiaomi vs. Google Pixel).
- **Rejected**: Fails the core product requirement of immersive, multi-speaker conversational dialogue.

### Cloud-based TTS (ElevenLabs, OpenAI TTS, Google Cloud TTS)
- **Pros**: Highest audio quality, zero on-device compute.
- **Cons**: Expensive API fees per comment stream, requires network connectivity for playback, introduces 300–800ms network round-trip latency per chunk, and violates user privacy expectations for offline reading.
- **Rejected**: Violates offline, zero-cost, on-device architectural goals.

## Consequences

### Positive
- **Compact Footprint**: Piper INT8 voice checkpoints are $\sim15\text{ MB}$ to $25\text{ MB}$ each.
- **Blazing Fast Mobile Inference**: Achieves RTF between $0.10\times$ and $0.25\times$ on mid-tier mobile CPUs using the ONNX Runtime XNNPACK execution provider with 2 threads.
- **Self-Contained Phonemization**: Piper's mobile runtime handles phoneme translation deterministically with minimal runtime overhead.
- **Predictable Memory Profile**: Total ONNX runtime session memory consumption stays below $80\text{ MB}$ RSS.

### Negative / Trade-offs
- **Audio Fidelity Ceiling**: While clean and intelligible, Piper's prosody is more uniform than Kokoro-82M's expressive conversational cadence.
- **Voice Bundling**: Supporting 4 distinct voices requires $\sim60\text{ MB}$ of assets, necessitating an on-demand asset pack or dynamic download on first launch to keep the initial APK install lean.

### Mitigation & Future Path
- The `TtsEngine` Kotlin interface decouples model execution from the rest of the application. Once the core pipeline stabilizes, a standalone NDK module for `espeak-ng` will be added to support Kokoro-82M as an opt-in "Ultra HD Voice Pack" for high-end devices.
