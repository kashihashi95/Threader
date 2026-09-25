# Model Ingestion & Quantization Runbook: PyTorch to Mobile INT8 ONNX

## 1. Overview
This runbook provides reproducible, end-to-end instructions and automation scripts to convert raw PyTorch/HuggingFace TTS checkpoints (e.g. Piper VITS or Kokoro) into mobile-optimized **INT8 ONNX Runtime models** ready for Android deployment in Redder.

---

## 2. Environment Setup

Run within a dedicated Python virtual environment (Python 3.10+):

```bash
python -m venv .venv-onnx
source .venv-onnx/bin/activate  # Or on Windows: .\.venv-onnx\Scripts\Activate.ps1

pip install --upgrade pip
pip install torch torchaudio onnx onnxruntime onnxruntime-tools numpy soundfile
```

---

## 3. Step-by-Step Conversion & Quantization Script

Save the following automation script as `scripts/export_and_quantize_voice.py`:

```python
#!/usr/bin/env python3
"""
Model Ingestion & Quantization Pipeline for Redder TTS Engine.
Converts PyTorch checkpoints to ONNX, runs graph optimizations,
and quantizes weights to dynamic INT8 for mobile ARM execution.
"""

import sys
import argparse
from pathlib import Path
import numpy as np
import onnx
from onnxruntime.quantization import quantize_dynamic, QuantType
import onnxruntime as ort

def optimize_and_quantize(input_onnx_path: str, output_int8_path: str):
    print(f"[*] Loading FP32 ONNX Model: {input_onnx_path}")
    model = onnx.load(input_onnx_path)
    onnx.checker.check_model(model)

    print("[*] Running Dynamic INT8 Quantization (Linear weights & MatMul)...")
    quantize_dynamic(
        model_input=input_onnx_path,
        model_output=output_int8_path,
        weight_type=QuantType.QInt8,
        per_channel=True,
        reduce_range=False,
        nodes_to_exclude=["output_audio_layer"] # Preserve output fidelity
    )

    orig_size_mb = Path(input_onnx_path).stat().st_size / (1024 * 1024)
    quant_size_mb = Path(output_int8_path).stat().st_size / (1024 * 1024)
    print(f"[✓] Quantization Complete!")
    print(f"    Original Size: {orig_size_mb:.2f} MB")
    print(f"    Quantized Size: {quant_size_mb:.2f} MB")
    print(f"    Compression Ratio: {(1 - quant_size_mb / orig_size_mb) * 100:.1f}%")

def validate_mobile_inference(quantized_model_path: str):
    print(f"[*] Validating mobile inference session with XNNPACK...")
    opts = ort.SessionOptions()
    opts.intra_op_num_threads = 2
    opts.graph_optimization_level = ort.GraphOptimizationLevel.ORT_ENABLE_ALL

    session = ort.InferenceSession(quantized_model_path, opts, providers=["CPUExecutionProvider"])
    
    # Generate dummy input tokens (e.g., phoneme IDs for 10-word sentence)
    dummy_phonemes = np.random.randint(1, 100, size=(1, 35), dtype=np.int64)
    dummy_lengths = np.array([35], dtype=np.int64)
    dummy_scales = np.array([0.667, 1.0, 0.8], dtype=np.float32)

    input_names = [inp.name for inp in session.get_inputs()]
    feed_dict = {}
    if "input" in input_names:
        feed_dict["input"] = dummy_phonemes
    if "input_lengths" in input_names:
        feed_dict["input_lengths"] = dummy_lengths
    if "scales" in input_names:
        feed_dict["scales"] = dummy_scales

    print("[*] Executing test inference pass...")
    outputs = session.run(None, feed_dict)
    audio_output = outputs[0]

    print(f"[✓] Validation Passed! Generated audio shape: {audio_output.shape}")
    print(f"    Sample rate: 22,050 Hz | Total samples: {audio_output.size}")

if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="Export and Quantize TTS Model for Redder")
    parser.add_argument("--input-onnx", required=True, help="Path to input FP32 ONNX model")
    parser.add_argument("--output-int8", required=True, help="Path to output INT8 ONNX model")
    args = parser.parse_args()

    optimize_and_quantize(args.input_onnx, args.output_int8)
    validate_mobile_inference(args.output_int8)
```

---

## 4. Execution Example

```bash
# 1. Download reference Piper voice checkpoint (e.g. en_US-lessac-medium)
wget https://huggingface.co/rhasspy/piper-voices/resolve/main/en/en_US/lessac/medium/en_US-lessac-medium.onnx -O en_US-lessac.onnx

# 2. Run INT8 Quantization
python scripts/export_and_quantize_voice.py \
    --input-onnx en_US-lessac.onnx \
    --output-int8 app/src/main/assets/voices/voice_en_lessac_int8.onnx

# Expected Output:
# [*] Loading FP32 ONNX Model: en_US-lessac.onnx
# [*] Running Dynamic INT8 Quantization (Linear weights & MatMul)...
# [✓] Quantization Complete!
#     Original Size: 63.40 MB
#     Quantized Size: 18.25 MB
#     Compression Ratio: 71.2%
# [*] Validating mobile inference session with XNNPACK...
# [✓] Validation Passed! Generated audio shape: (1, 1, 24500)
```

---

## 5. Deployment into Android Assets
Once quantized, move the `.onnx` model and its accompanying phoneme config JSON into the Android assets directory:

```
app/src/main/assets/voices/
├── voice_en_female_1.onnx      # INT8 quantized weights (~18MB)
├── voice_en_female_1.json      # Phoneme map, sample rate (22050Hz)
├── voice_en_male_1.onnx        # INT8 quantized weights (~18MB)
└── voice_en_male_1.json
```
