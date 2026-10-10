# Spike: monitoring agy lewat `stream-json`

> Status: untuk dicoba di sesi Claude Code **lokal** (mesin yang sudah punya `agy` terpasang dan login).
> Dibuat 2026-10-10. Semua klaim tentang skema `stream-json` di bawah berasal dari
> [dokumentasi headless resmi](https://antigravity.google/docs/cli/headless/) dan **belum diverifikasi**
> pada agy asli. Tujuan spike ini adalah memverifikasinya.

## Tujuan

agy-bridge saat ini menjalankan `agy -p <prompt>` (mode teks), jadi tidak ada visibilitas selama
agy bekerja. Hipotesis: `--output-format stream-json` memberi event per langkah (tool call,
teks, usage) yang cukup untuk membangun pemantauan setara model job Codex
(`status` / `result` / `cancel`).

Spike ini menjawab: **apakah hipotesis itu benar, dan apa bentuk event sebenarnya?**
Hasilnya adalah fixture NDJSON nyata plus catatan temuan. Belum ada perubahan kode produksi.

## Prasyarat

- `agy` terpasang dan terautentikasi (`agy --version` berjalan).
- Repo ini di-clone dan branch `claude/dreamy-clarke-uuiqvv` di-checkout.
- Buat folder uji kecil di luar repo, mis. `~/agy-spike/` berisi 3 file teks/kode kecil.
- Simpan output mentah di `test/fixtures/stream-json/` (ganti path, nama, atau data sensitif sebelum commit).

## Eksperimen

Jalankan satu per satu dan simpan output ke file bernama sesuai nomor. Catat hasil di bagian
"Lembar temuan".

### E1: Flag tersedia

```bash
agy --version
agy --help
```

Cek bahwa `--output-format`, `--input-format`, `--conversation`, `--log-file`, `--print-timeout`
ada. Catat versinya. Dokumentasi menyebut flag headless baru ada sejak sekitar v1.1.8.

### E2: Prompt sederhana tanpa tool

```bash
agy --output-format stream-json -p "Jawab satu kalimat: apa itu NDJSON?" > e2.ndjson
```

Periksa: event `init` (isi `cwd`, `tools`, `permission_mode`), beberapa `step_update` dengan
`text_delta`, lalu satu `result`. Apakah ada `conversation_id` di `step_update` dan `result`?

### E3: Prompt yang memakai tool (kasus nyata agy-bridge)

```bash
cd ~/agy-spike
agy --dangerously-skip-permissions --add-dir "$PWD" --output-format stream-json \
  -p "Baca semua file di folder ini dan ringkas masing-masing dalam satu baris." > e3.ndjson
```

Periksa: `step_type: "tool"` dengan `tool_name`, dan `tool_info` berisi `parameters` dan `output`.
**Pertanyaan kunci:** apakah `tool_info.output` penuh atau terpotong untuk file besar? Ini
menentukan apakah jejak aktivitas aman disimpan atau perlu dipotong.

### E4: Kompatibilitas dengan `--log-file` dan deteksi kuota

```bash
agy --output-format stream-json --log-file ./e4.log -p "halo" > e4.ndjson
```

Periksa: apakah `--log-file` tetap terisi di mode stream-json? `src/quota.ts` bergantung pada log itu
untuk mendeteksi `RESOURCE_EXHAUSTED (code 429)`. Jika kuota habis, apakah stream menampilkan
error atau diam seperti mode teks? (Jika kuota sedang tidak habis, catat saja bahwa ini belum teruji.)

### E5: `conversation_id` vs `last_conversations.json`

Jalankan E3 dari dua cwd berbeda, lalu bandingkan `conversation_id` di `result` dengan isi
`~/.gemini/antigravity-cli/cache/last_conversations.json`. Lalu jalankan **dua run bersamaan di cwd yang sama**
(`&` di shell) dan lihat apakah file itu hanya menyimpan salah satu id. Ini mengonfirmasi bug race
di `runner.ts` yang bisa diperbaiki dengan membaca id dari stream.

### E6: Resume dengan stream-json

```bash
agy --output-format stream-json --conversation <ID_DARI_E3> \
  -p "Dari ringkasan tadi, file mana yang paling panjang?" > e6.ndjson
```

Periksa: apakah konteks turn sebelumnya terbawa, dan `conversation_id` sama.

### E7: Cancel

Jalankan E3 dengan prompt panjang, lalu kirim `SIGINT` (Ctrl+C) di tengah jalan.
Periksa: apakah `result` terakhir bernilai `status: "INTERRUPTED"` dan apakah exit code-nya
non-nol. Ulangi dengan `SIGTERM` ke process group (cara yang dipakai `runner.ts`), dan catat
apakah ada `result` sama sekali.

### E8: Permission tanpa `--dangerously-skip-permissions`

```bash
agy --output-format stream-json -p "Jalankan perintah: echo halo" > e8.ndjson
```

Dokumentasi bilang shell command di-soft-deny: run lanjut, exit 0, pesan di stderr. Periksa
bagaimana itu terlihat di stream (`tool_info.error`?) dan apakah `status` tetap `SUCCESS`.
Ini menentukan apakah bridge bisa menampilkan "ada tool yang ditolak" kepada Claude.

### E9 (opsional): Proses persisten dua turn

```bash
printf '%s\n' \
 '{"event":"user","message":{"content":"Sebut satu warna."}}' \
 '{"event":"user","message":{"content":"Sebut satu warna lain."}}' \
| agy --output-format stream-json --input-format stream-json > e9.ndjson
```

Periksa: dua event `result`, `num_turns` kumulatif, dan ukur waktu turn pertama vs kedua untuk
melihat apakah cold start 40–50 detik benar-benar hilang di turn kedua.

## Lembar temuan (isi setelah menjalankan)

| Eksperimen | Sesuai dokumentasi? | Catatan / deviasi |
| ---------- | ------------------- | ----------------- |
| E1 versi & flag | | |
| E2 event dasar | | |
| E3 `tool_info` penuh/terpotong | | |
| E4 `--log-file` + kuota | | |
| E5 `conversation_id` & race | | |
| E6 resume | | |
| E7 cancel / status | | |
| E8 soft-deny | | |
| E9 persisten | | |

## Kriteria lanjut

Lanjut ke implementasi jika **semua** ini benar:

1. Event `tool` memuat `tool_name` dan ringkasan parameter yang cukup untuk ditampilkan (E3).
2. `conversation_id` ada di stream (E2/E5).
3. Deteksi kuota tetap bisa dilakukan, lewat log atau lewat stream (E4).
4. Cancel menghasilkan status yang bisa dibedakan dari error (E7).

Jika poin 3 gagal, rancangan harus mempertahankan polling log file. Jika poin 1 gagal, nilai
pemantauan turun jauh dan pendekatan perlu ditinjau ulang sebelum menulis kode.

## Prompt untuk sesi Claude Code lokal

Tempel ini di sesi lokal setelah branch di-checkout:

```text
Baca docs/stream-json-spike.md dan src/runner.ts. Jalankan eksperimen E1–E8 di mesin ini
(E9 opsional) dengan agy yang terpasang. Simpan output mentah ke test/fixtures/stream-json/
(redaksi path pribadi dan data sensitif). Isi tabel "Lembar temuan" dengan hasil nyata,
bandingkan dengan klaim dokumentasi, dan beri tahu saya kriteria lanjut mana yang lolos atau gagal.
Jangan ubah src/ dan jangan delegasikan eksperimen ke agy-bridge sendiri: panggil `agy` langsung,
karena yang diuji adalah perilaku mentahnya. Jangan buat PR.
```

## Setelah spike lolos (rencana implementasi, belum dikerjakan)

1. `runner.ts`: parse NDJSON, tulis jejak ringkas per job ke file, ambil `conversation_id` dari event.
2. Tool baru: `start_job`, `job_status`, `job_result`, `job_cancel`.
3. Opsional: bungkus sebagai plugin Claude Code (`/agy:status` dan seterusnya), lalu pertimbangkan proses persisten (E9).
