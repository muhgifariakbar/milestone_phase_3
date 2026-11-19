# 04 · Governance & Integration Readiness Checklist · H8Mart Quick-Commerce Ops

**Nama:** Gifari

Status **Siap** hanya bila buktinya sudah ada di repo atau `evidence/`. Baris yang butuh screenshot/query yang belum diambil ditulis **Belum** beserta file buktinya yang harus dibuat.

## 1. Credential & Akses

| Pemeriksaan | Status | Bukti |
| --- | --- | --- |
| Semua credential disimpan di n8n Credentials, bukan di node/Code/repo | Siap | `workflows/*.json`: semua node memakai referensi credential `GANTI-ID-*`, tanpa nilai rahasia |
| Role DB dipisah: `mcp_reader` (read-only) untuk WF2 dan membaca di WF3, `ops_writer` untuk menulis `ops.*` | Siap | Export: WF2 dan WF3 membaca memakai `Postgres mcp_reader` (`GANTI-ID-PG-MCP-READER`); WF1/WF4/WF5 dan `Buat Tiket CS` memakai `Postgres ops_writer`. Bukti penolakan `INSERT` oleh `mcp_reader`: `evidence/wf2/` (**Belum**, ambil screenshot) |
| Webhook dilindungi Header Auth | Siap | Node webhook WF1 (`order-created`), WF3 (`merchant-question`), WF4 (`ops-event`) memakai credential `Header Auth H8Mart Webhook` |
| `.env` tidak ada di repo; `.env.example` tanpa nilai nyata | Siap | `.gitignore` memuat `.env`/`.env.*`; `git ls-files` tidak memuat `.env` |
| Token push Uptime Kuma diganti `ISI_TOKEN_PUSH_KUMA` di JSON export sebelum commit | Siap | `workflows/wf1-starter.json` dan `wf5-starter.json` memuat `ISI_TOKEN_PUSH_KUMA` pada URL Kuma |
| Token bot Telegram hanya di credential n8n (bukan repo, export, screenshot) | Siap | Export hanya mereferensikan credential `Telegram H8Mart Ops` (`GANTI-ID-TELEGRAM`); cek screenshot di `evidence/` tidak menampilkan token |
| Chat Telegram dibatasi allowlist `chat_id` ke `merchant_id`; chat asing ditolak | Siap (alur) / bukti percakapan **Belum** | WF3b: `Cek Allowlist` -> `Chat Terdaftar?` -> `Balas Chat Asing`; tabel `ops.telegram_merchants`. Bukti penolakan: `evidence/wf3/` |
| Akses ke instance n8n dibatasi (user/owner, port yang terbuka) | Belum | Perlu screenshot Security Group (22 hanya My IP, 80/443 terbuka; 5690/5440/8080/3011 tidak) di `evidence/deploy/`. `Caddyfile` memblokir `/mcp*` (403) |

## 2. Kesiapan Sistem Existing

| Sistem | Cara terhubung | Rate limit / SLA yang diketahui | Perilaku bila down | Siapa dihubungi |
| --- | --- | --- | --- | --- |
| Merchant Stock API | HTTP GET `/merchants/{id}/stock` dengan header `X-Api-Key` (credential Header Auth) | Tidak ada rate limit tertulis di kontrak; M007 lambat (~2,5 dtk), M004 gagal sementara, M009 gagal permanen | Retry On Fail 4x (jeda 1 dtk); gagal permanen -> `ops.dead_letter` + alert, merchant lain tetap diproses; seluruh API down -> Error Workflow + heartbeat Kuma Down | Tim merchant integration (kontak tidak tersedia di soal; **isi**) |
| Payment API | HTTP POST `/payments/refunds` dan `/vouchers` dengan `Idempotency-Key` | Tidak ada rate limit tertulis; `503` saat gateway gangguan | Error output -> dead-letter (voucher/refund) + alert; aman diulang karena idempotent (200 `replayed`) | Finance (kontak: **isi**) |
| Notification (mock API) | HTTP POST `/notify` | Tidak ada rate limit tertulis | Hanya jejak tambahan; kegagalan tidak menghentikan alur utama | Ops |
| Telegram Bot API (digest, alert, approval, chat) | Credential Telegram API (token BotFather), node Telegram dan Telegram Trigger lewat HTTPS | Batas Telegram (~1 pesan/detik per chat, ~30 pesan/detik total) cukup untuk volume ini | Digest/alert tidak terkirim -> jejak di Notification API dan Executions; chat id salah menghasilkan `chat not found` (terjadi pada versi published, dikeluhkan penilai; harus dikoreksi di node Telegram WF1/WF4/WF5 dan diverifikasi ulang dengan screenshot chat) | Pemilik bot (Gifari) |
| PostgreSQL | Credential `mcp_reader` / `ops_writer`, host `postgres:5432` internal | Kapasitas satu instance; tidak ada SLA | Workflow gagal -> Error Workflow alert; data di volume `pg_data` + snapshot EBS | Admin (Gifari) |
| LLM provider | Credential OpenAI API, model `gpt-6-luna` | Kunci dibagi satu kelas; kuota/rate limit dari OpenAI | Agent gagal -> jalur `Fallback: AI Gagal` (tanpa jawaban karangan, eskalasi) | Instruktur (pemilik key) |

## 3. Pra-Production

| Pemeriksaan | Status | Bukti |
| --- | --- | --- |
| Error Workflow terpasang di semua workflow | Siap (export) / verifikasi di VPS | `settings.errorWorkflow` di `workflows/wf1..wf4-starter.json` (placeholder `GANTI-ID-WF5`, diganti ID WF5 saat import). Eksekusi Error Workflow: `evidence/wf5/` |
| Retry dan dead-letter diuji dengan kegagalan disengaja | Belum | Retry: Executions WF1 + `GET /admin/stock-calls`; dead-letter: `SELECT * FROM ops.dead_letter` (M009, `HM-SP-099`) di `evidence/wf1/` dan `evidence/wf5/` |
| Heartbeat + deteksi sinkron 0 baris / digest kosong + ambang durasi | Belum (alur ada) | Node `Heartbeat Uptime Kuma`, `Gagal: Sinkron 0 Baris`, `Gagal: Digest Kosong`, `Terlalu Lambat?` (60.000 ms). Bukti Kuma Up/Down dan alert: `evidence/wf5/` |
| Data pelanggan di log/alert tidak berlebihan (masking) | Siap | `Susun Pesan Alert` hanya memuat workflow, node, waktu, business key (`order_id`/`merchant_id`), dan pesan error; tidak memuat `customer_id` atau `customer_note` |
| Rencana rollback / Unpublish | Siap (rencana) | Rollback: Unpublish workflow bermasalah (jangan Unpublish WF1 karena webhook order), restore versi sebelumnya dari History n8n atau export JSON di git; data: restore snapshot EBS |
| Biaya token LLM dipantau | Belum | Perhitungan estimasi di `docs/02`; pantau token aktual dari Executions WF3 (ganti estimasi dengan angka nyata) |
