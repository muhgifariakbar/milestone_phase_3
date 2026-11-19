# 03 · TCO 12 Bulan + Keputusan Deployment · H8Mart Quick-Commerce Ops

**Nama:** Gifari

Asumsi umum: 1 bulan = 730 jam (8.760 jam/tahun); kurs asumsi **Rp16.500 per US$1** dan **Rp19.000 per EUR1** (belum diverifikasi, cek kurs BI pada hari submit); tarif admin Rp50.000/jam; beban ~**9.000 eksekusi/bulan** (~5.400 webhook order + 2.880 sinkron + 720 pertanyaan AI + 30 digest + event WF4).

## 1. Opsi yang Dibandingkan

| Opsi | Bentuk | Biaya awal | Biaya bulanan | Biaya 12 bulan | Catatan kepatuhan / data |
| --- | --- | --- | --- | --- | --- |
| Cloud (n8n Cloud) | Paket **Pro** (Starter hanya 2.500 eksekusi/bulan, tidak cukup untuk ~9.000) + PostgreSQL terkelola eksternal untuk data H8Mart | US$0 | €50 + DB US$15 (asumsi, belum ada kuotasi) + admin 0,5 jam | €50 x 12 = €600 = Rp11,40 jt; DB US$15 x 12 = US$180 = Rp2,97 jt; admin 6 jam x Rp50.000 = Rp0,30 jt; **total Rp14,67 jt** | Data order dan pelanggan diproses di infrastruktur pihak ketiga (lokasi region perlu dipastikan); kuota eksekusi 10.000 hampir habis |
| Self-hosted (VPS + Docker Compose) | AWS EC2 `t3.small` Jakarta + Caddy + DuckDNS, stack milestone lengkap | Rp0 (AWS) + waktu setup | Lihat rincian di bawah: US$25,92 + admin 2 jam | US$25,92 x 12 = US$311,04 = Rp5,13 jt; admin 24 jam x Rp50.000 = Rp1,20 jt; **total Rp6,33 jt** | Data tetap di server yang dikontrol sendiri (region Jakarta); tanggung jawab patch dan backup di tim sendiri |
| On-premise (server H8Mart) | Server fisik di kantor | Asumsi Rp15 jt (server; belum ada kuotasi) | Listrik + internet asumsi Rp0,4 jt + admin 4 jam (Rp0,2 jt) | 15 jt + (0,4 + 0,2) x 12 = **Rp22,2 jt** | Kontrol penuh dan data tidak keluar; butuh IP publik tetap dan HTTPS sendiri; satu titik gagal (listrik, internet) |

**Rincian self-hosted per bulan (730 jam):**
EC2 `t3.small` Jakarta US$0,0264/jam x 730 = US$19,27; IPv4 publik US$0,005/jam x 730 = US$3,65; EBS gp3 20 GiB ~US$0,10/GB-bulan = US$2,00; snapshot backup EBS (asumsi ~US$1,00); domain DuckDNS gratis. **Total US$25,92 = Rp427.680**; waktu admin 2 jam x Rp50.000 = Rp100.000.
Sumber harga EC2/IPv4/EBS: tabel estimasi di `SETUP.md` (dicek pada 30 Okt 2026 dari halaman harga AWS; halaman harga on-demand AWS tidak memuat tabel per instance saat saya cek, jadi **verifikasi ulang di AWS Pricing Calculator** sebelum submit). Sumber harga n8n Cloud: https://n8n.io/pricing/ (Starter €20/bulan dan Pro €50/bulan, penagihan tahunan, 2,5 ribu dan 10 ribu eksekusi/bulan; diakses 19 Nov 2026). Harga DB eksternal dan server on-prem adalah **asumsi**, bukan kuotasi.

## 2. Keputusan & Justifikasi

**Dipilih: self-hosted VPS (AWS EC2 t3.small + Docker Compose + Caddy HTTPS).**

1. **Biaya:** Rp6,33 jt/12 bulan vs Rp14,67 jt (Cloud Pro + DB) dan Rp22,2 jt (on-prem): ~57% lebih murah dari Cloud dan tidak dibatasi kuota eksekusi (Cloud Starter 2.500 tidak cukup).
2. **Teknis:** workflow butuh PostgreSQL `h8mart`, mock API, dan Uptime Kuma di jaringan internal yang sama dengan n8n (nama service `postgres`, `mock-api`, `uptime-kuma`) serta endpoint MCP internal; di Cloud semuanya harus dibuka ke internet.
3. **Syarat kepatuhan:** data order dan pelanggan adalah data pribadi; **UU No. 27 Tahun 2022 tentang Pelindungan Data Pribadi** mewajibkan pengendali melindungi keamanan data dari akses tidak sah. Self-hosted memungkinkan data tetap di region Jakarta dengan Security Group (hanya 80/443 terbuka, DB dan MCP tidak terjangkau dari internet). *(Catatan: pasal spesifik yang kamu baca perlu kamu tulis sendiri di sini sebelum submit; saya tidak menyebut nomor pasal karena belum dibaca.)*

## 3. Rencana Deployment (final)

| Item | Nilai |
| --- | --- |
| Penyedia & spesifikasi instance | AWS EC2 `t3.small` (2 vCPU, 2 GiB RAM, x86_64), region Asia Pacific (Jakarta), Ubuntu Server 24.04 LTS, 20 GiB gp3, swap 2 GiB (RAM pas-pasan) |
| `compose.yaml`: service yang dijalankan | `n8n` (2.44.7), `task-runners`, `postgres` (16.15-alpine, seed `core`/`ops`), `mock-api`, `uptime-kuma`; `compose.vps.yaml` menambah `caddy` (satu-satunya yang membuka port 80/443) |
| Variabel penting di `.env` (tanpa nilai rahasia) | `DOMAIN_NAME`, `SSL_EMAIL`, `N8N_ENCRYPTION_KEY`, `PG_SUPERUSER_PASSWORD`, `PG_READER_PASSWORD`, `PG_WRITER_PASSWORD`, `MOCK_API_KEY`, `N8N_RUNNERS_AUTH_TOKEN`, `WEBHOOK_TOKEN`, `N8N_VERSION` (semua diganti nilai acak di server; `.env` tidak di-commit) |
| Database n8n | SQLite bawaan di volume `n8n_data`; PostgreSQL di stack hanya untuk data H8Mart |
| HTTPS (domain + reverse proxy + sertifikat) | Domain `h8mart-gifari.duckdns.org` (DuckDNS) -> Caddy -> `n8n:5678`; sertifikat Let's Encrypt otomatis; endpoint `/mcp/*`, `/mcp-test/*`, `/mcp-server/*` diblokir 403 oleh Caddyfile |
| Production URL yang dipanggil penilai | `https://h8mart-gifari.duckdns.org/webhook/order-created` (WF1), `/webhook/merchant-question` (WF3), `/webhook/ops-event` (WF4); header `X-H8Mart-Token` (token dikirim lewat DM, bukan di repo). Editor: https://h8mart-gifari.duckdns.org/home/workflows |
| Backup (apa, seberapa sering, uji restore) | Snapshot EBS volume (mencakup `n8n_data`, `pg_data`, `kuma_data`) mingguan; export JSON workflow di repo git. **Uji restore belum dilakukan** (belum ada bukti; isi setelah dijalankan) |

## 4. Bukti Verifikasi

| Bukti | File |
| --- | --- |
| Browser HTTPS valid | `evidence/deploy/01-https.png` |
| `docker compose ps` di VPS | `evidence/deploy/02-compose-ps.png` |
| `curl` Production URL dari luar jaringan VPS | `evidence/deploy/03-curl.txt` |
