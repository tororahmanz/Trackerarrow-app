# TRACKERARROW — Dokumen Requirement Aplikasi

| | |
|---|---|
| **Platform** | Streamlit (Python) |
| **Versi dokumen** | 0.2 |
| **Tanggal** | 2026-10-20 |
| **Status** | Perencanaan — belum ada kode |

---

## 0. Keputusan yang sudah diambil

| # | Topik | Keputusan |
|---|---|---|
| 1 | Chain | **EVM + Solana** |
| 2 | Winrate | **Dihitung otomatis** dari data on-chain |
| 3 | Notifikasi | **Telegram** (per user) |
| 4 | Pengguna | **Multi-user**. Halaman Settings & Admin **hanya untuk owner** |
| 5 | Hosting | **VPS** (Docker Compose) |

---

## 1. Ringkasan

trackerarrow adalah aplikasi web berbasis Streamlit untuk banyak pengguna yang bisa:

- Menyimpan wallet crypto (EVM & Solana), memberi nama, icon, dan memasukkannya ke **list** buatan sendiri.
- Melihat **kategori winrate** dan **kategori aset** setiap wallet, yang dihitung otomatis.
- Memetakan **relasi antar wallet** dalam graf visual yang interaktif dan ringan.
- Mengetahui **sudah berapa lama** sebuah wallet ditrack.
- Menerima **notifikasi Telegram** setiap kali wallet yang ditrack bertransaksi.

---

## 2. Pengguna & Hak Akses

### 2.1 Peran

| Peran | Siapa | Akses |
|---|---|---|
| 👑 **Owner** | Anda (ditentukan lewat `OWNER_EMAIL` di `.env`) | Semua halaman, termasuk ⚙️ Settings dan 🛡️ Admin |
| 👤 **User** | Pengguna terdaftar | Dashboard, Wallets, Lists, Relations, Alerts, Profil |

- Hanya ada **satu owner**. Peran owner tidak bisa diberikan lewat UI.
- Halaman Settings dan Admin **tidak tampil** di menu user biasa. Kalau diakses langsung lewat URL, halaman menolak akses.

### 2.2 Login

- Login memakai fitur bawaan Streamlit `st.login()` (OpenID Connect), misalnya **Google login**.
- User baru otomatis terdaftar saat login pertama kali.
- Owner bisa mengatur mode pendaftaran: **terbuka**, **pakai undangan**, atau **ditutup**.
- Owner bisa memblokir atau mengaktifkan kembali user.

### 2.3 Isolasi data antar user

- **Data on-chain bersifat global** (dipakai bersama): winrate, aset, riwayat transaksi. Wallet yang sama hanya dipantau dan dihitung satu kali.
- **Data milik user bersifat privat**: nama/alias, icon, catatan, tag, list, relasi, dan riwayat alert. User A tidak bisa melihat label atau list milik user B.

### 2.4 Kuota (diatur owner)

- Jumlah maksimal wallet yang ditrack per user (default 100).
- Jumlah maksimal list per user (default 20).
- Jumlah maksimal notifikasi Telegram per user per jam (default 120). Ini mencegah spam dan menjaga batas Telegram.

---

## 3. Manajemen Wallet

### 3.1 Data global per wallet (otomatis, sama untuk semua user)

- `address` dan `chain` (unik per pasangan). Format divalidasi:
  - EVM: checksum `0x…`, 42 karakter.
  - Solana: base58, 32–44 karakter.
- Winrate, jumlah trade, realized PnL, kategori winrate (lihat §4).
- Total aset (USD) dan kategori aset (lihat §5).
- Waktu terakhir aktif dan waktu terakhir statistik dihitung.

### 3.2 Data privat per user

- **Alias** (contoh: `the wallet`, `Insider Pepe`).
- **Icon**: emoji atau gambar upload (png/svg, maks 256 KB).
- **Tag** bebas (contoh: `sniper`, `early buyer`, `dev`).
- **Catatan** (teks bebas).
- **Notifikasi**: ON/OFF per wallet.
- **Minimum amount**: notif hanya muncul jika transaksi ≥ $X (opsional, per wallet).
- **`tracked_since`**: waktu wallet pertama kali ditambahkan oleh user ini (lihat §7).

### 3.3 Chain yang didukung

| Keluarga | Chain (versi awal) | Emoji default | Explorer |
|---|---|---|---|
| EVM | Ethereum | 💎 | etherscan.io |
| EVM | Base | 🔵 | basescan.org |
| EVM | BNB Chain | 🟡 | bscscan.com |
| EVM | Arbitrum | 🔷 | arbiscan.io |
| EVM | Robinhood | 🐦 | robin.etherscan.io |
| Solana | Solana | 🟣 | solscan.io |

- Owner bisa menambah atau mengubah chain, emoji, dan URL explorer di halaman Settings.
- Dukungan chain EVM baru bergantung pada tersedianya API dari penyedia data (lihat §11).

#### Hasil pengecekan dukungan API (20 Okt 2026)

| Penyedia | Robinhood (chain ID 4663) | BNB Chain (chain ID 56) |
|---|---|---|
| **Alchemy** (RPC, webhook, Transfers API) | ✅ Mainnet sudah live, termasuk webhook | ✅ Didukung, termasuk webhook |
| **DexScreener** (harga & mcap) | ✅ Sudah dites langsung: token STOX muncul dengan chainId `robinhood` | ✅ chainId `bsc` |
| **Etherscan API V2** | ✅ Diumumkan didukung (RobinScan). Belum pasti masuk paket gratis | ⚠️ Menurut sumber pihak ketiga, tidak masuk paket gratis (paket Lite ±$49/bln) |
| **Alternatif gratis** | Blockscout (`robinhoodchain.blockscout.com`) | BSCTrace (NodeReal MegaNode) |

**Keputusan:** **Alchemy** menjadi sumber utama untuk **semua chain EVM**. Webhook dipakai untuk notifikasi real-time, dan `alchemy_getAssetTransfers` dipakai untuk polling cadangan dan menghitung winrate. Etherscan tidak wajib, sehingga BNB Chain tidak menambah biaya. Daftar chain di dashboard Alchemy tetap dicek ulang saat setup.

### 3.4 Fitur tambahan

- Tambah wallet satu per satu lewat form, atau bulk import CSV/JSON (langsung bisa dipilih list tujuannya).
- Export wallet dan list milik sendiri ke CSV/JSON.
- Cari dan filter berdasarkan chain, kategori, list, tag, dan lama ditrack.
- Menghapus wallet hanya menghapus data privat milik user itu (alias, relasi, keanggotaan list). Data global tetap ada selama masih ditrack user lain.

---

## 4. Kategori WINRATE (5 tingkat, otomatis)

### 4.1 Tingkatan

| Ikon | Nama | Winrate default | Makna |
|---|---|---|---|
| ⚡ | **Split Arrow** | ≥ 85% | Elite — panahnya membelah panah lain di tengah sasaran |
| 🎯 | **Bullseye** | 75–84% | Hampir selalu tepat sasaran |
| 🏹 | **Sharpshooter** | 60–74% | Konsisten profit |
| 🪶 | **Archer** | 45–59% | Rata-rata, kadang meleset |
| 🍂 | **Stray Arrow** | < 45% | Sering meleset, cocok untuk counter-trade |
| 🆕 | *Unranked* | — | Data belum cukup (jumlah trade di bawah minimum) |

- **Semua batas angka bisa diubah owner** di halaman ⚙️ Settings. Sistem memvalidasi bahwa batasnya berurutan dan tidak tumpang tindih.
- Nama dan ikon kategori juga bisa diubah owner.

### 4.2 Cara menghitung winrate

1. Ambil semua **swap** wallet dalam jendela waktu tertentu (default **30 hari**; owner bisa pilih 7/30/90 hari).
2. Kelompokkan swap **per token**. Hitung modal (cost basis) dengan metode **FIFO**. Nilai USD diambil saat transaksi terjadi.
3. Sebuah token dihitung sebagai **1 trade selesai** jika sudah ada penjualan (minimal sebagian).
4. Trade dihitung **WIN** jika realized PnL > **+X%** (default 0%, bisa diatur owner).
5. **Winrate = jumlah WIN ÷ jumlah trade selesai × 100%.**
6. Jika jumlah trade selesai < **minimum** (default 5), wallet berstatus **🆕 Unranked**.
7. Token yang masuk lewat transfer biasa atau airdrop (modal tidak diketahui) **tidak ikut dihitung** dalam winrate.
8. Jadwal hitung ulang: setiap **6 jam** untuk semua wallet, dan langsung setelah ada transaksi baru pada wallet itu.

Statistik pendukung yang ikut ditampilkan: jumlah trade, realized PnL (USD), rata-rata lama memegang token, dan token paling menguntungkan.

---

## 5. Kategori ASET (4 tingkat, otomatis)

| Ikon | Nama | Total aset default (USD) |
|---|---|---|
| 🐋 | **Whale** | ≥ $1.000.000 |
| 🦈 | **Shark** | $100.000 – $999.999 |
| 🐬 | **Dolphin** | $10.000 – $99.999 |
| 🐟 | **Shrimp** | < $10.000 |

- **Total aset** = saldo koin native + saldo token × harga saat ini.
- Token spam atau dengan likuiditas kecil (default < $1.000) tidak ikut dihitung.
- Dihitung ulang setiap 6 jam. Batas angka bisa diubah owner di ⚙️ Settings.
- **Label gabungan** ditampilkan seperti `⚡🐋 Split Arrow Whale`. Kombinasi 5 × 4 = 20 profil, dipakai untuk filter dan warna node di graf.

---

## 6. List Wallet (per user)

- Setiap user bisa membuat **list** dengan nama sendiri, misalnya `Wallet Track`, `KOL Solana`, `Insider Base`.
- Data per list: nama, icon/emoji, deskripsi, warna, dan **notifikasi ON/OFF untuk seluruh list**.
- Satu wallet bisa masuk ke **beberapa list** sekaligus.
- Tiap anggota list menyimpan `added_at`, yaitu kapan wallet itu dimasukkan ke list tersebut.
- Fitur list:
  - Tambah, rename, dan hapus list. Menghapus list **tidak** menghapus wallet di dalamnya.
  - Pindahkan atau salin wallet antar list, termasuk pilih banyak wallet sekaligus.
  - Ringkasan list: jumlah wallet, rata-rata winrate, total aset, dan jumlah alert 24 jam terakhir.
  - Filter graf relasi dan feed alert berdasarkan list.
- Nama list dipakai pada baris `📁` di notifikasi (lihat §9.3).

---

## 7. Lama Tracking

- Sistem menyimpan **`tracked_since`** untuk setiap pasangan user–wallet. Nilainya diisi saat wallet pertama kali ditambahkan.
- Ditampilkan di:
  - **Tabel wallet**: kolom "Ditrack" yang bisa diurutkan, contoh `69 hari`.
  - **Detail wallet**: `⏱️ Ditrack sejak 12 Agu 2026 · 69 hari`.
  - **Tampilan list**: "ditambahkan ke list ini 12 hari lalu" (dari `added_at`).
- Format lama tracking: `< 1 jam`, `5 jam`, `3 hari`, `2 bln 4 hari`, `1 thn 2 bln`.
- Statistik tambahan per wallet:
  - Jumlah alert sejak mulai ditrack.
  - Perubahan aset sejak mulai ditrack (contoh: `$45K → $120K (+167%)`). Nilai aset saat wallet pertama kali ditambahkan disimpan sebagai `asset_at_track`.
- Jika wallet dihapus lalu ditambahkan lagi, `tracked_since` dihitung ulang dari awal. Riwayat lama tetap tersimpan di log.

---

## 8. Relasi Antar Wallet (per user)

### 8.1 Data per relasi

- `wallet_from`, `wallet_to`.
- Nama relasi: bebas, atau pilih dari daftar siap pakai:
  - 💸 **Funding** — wallet A mendanai wallet B
  - 👥 **Same Owner** — diduga pemiliknya sama
  - 🪞 **Copy Trader** — B meniru trade A
  - 🕸️ **Cluster** — bagian dari kelompok insider/bundle
  - 🔁 **Transfer** — sering transfer bolak-balik
  - 🏦 **CEX Deposit** — terhubung ke deposit address exchange yang sama
- Arah: satu arah (A → B) atau dua arah (A ↔ B).
- Tingkat keyakinan 1–5, menentukan ketebalan garis.
- Warna garis (opsional) dan catatan.
- Relasi bersifat **privat** per user. Relasi lintas chain diperbolehkan, misalnya karena bridge.

### 8.2 Visualisasi graf

- **Node** = wallet, ditampilkan dengan icon dan alias. **Warna** node menunjukkan kategori winrate, **ukuran** node menunjukkan kategori aset.
- **Edge** = relasi, dengan label nama relasi dan panah arah.
- Interaksi: drag, zoom, pan. Klik node untuk menampilkan detail wallet di sidebar, termasuk lama tracking.
- Filter: list, kategori, chain, jenis relasi.
- **Mode fokus**: pilih satu wallet, lalu tampilkan hanya wallet yang terhubung dalam 1–2 langkah.
- Tombol physics ON/OFF. Posisi node disimpan agar layout tidak berubah setiap refresh.
- Target performa: tetap lancar untuk ±500 node dan 1.000 edge.
- Library: **streamlit-agraph** (berbasis vis.js).

---

## 9. Tracking Transaksi & Notifikasi Telegram

### 9.1 Deteksi transaksi

- Ditangani oleh **worker terpisah** yang jalan 24 jam, bukan di dalam proses Streamlit.
- **Utama — webhook** (real-time, hemat kuota):
  - Solana: **Helius Webhooks** (enhanced transactions sudah memuat data swap).
  - EVM: **Alchemy Notify** (Address Activity).
  - Webhook diterima oleh service **FastAPI** kecil di VPS.
- **Cadangan — polling** tiap N detik (default 30), untuk jaga-jaga jika webhook terputus:
  - EVM: `alchemy_getAssetTransfers` dari Alchemy.
  - Solana: Helius RPC.
  - Etherscan API V2 atau Blockscout bisa dipakai sebagai cadangan kedua (opsional).
- Hanya wallet yang ditrack **minimal oleh 1 user** dengan notifikasi aktif yang dipantau.
- Jenis aksi yang dideteksi:
  - 📈 BUY, 📉 SELL (swap)
  - 🔄 TRANSFER IN/OUT
  - (opsional) ADD/REMOVE LP
- Anti-duplikat: simpan `chain + tx_hash` yang sudah diproses.
- **Fan-out**: satu transaksi terdeteksi sekali, lalu dikirim ke setiap user yang mentrack wallet itu, memakai alias dan list milik masing-masing user.

### 9.2 Data pendukung

- Harga, market cap, nama, dan simbol token: **DexScreener API**. Cadangan: GeckoTerminal.
- Link explorer diambil dari tabel chain (§3.3).

### 9.3 Template notifikasi (wajib persis seperti ini)

```
        📣 WALLET TRACKER ALERT 🔭

🪙 {TOKEN_NAME} | {TOKEN_SYMBOL} ({CHAIN_EMOJI} {CHAIN_NAME})
{TOKEN_CONTRACT_ADDRESS}

🎯 Target: {WALLET_ICON} @{WALLET_ALIAS} ({ADDR_SHORT}) ({EXPLORER_URL}/address/{WALLET_ADDRESS})
📁 {LIST_NAME}
{ACTION_EMOJI} Action: {ACTION}
💰 Amount: {AMOUNT_USD}$
📊 Mcap: ${MCAP}
```

Aturan placeholder:

| Placeholder | Aturan |
|---|---|
| `WALLET_ICON`, `WALLET_ALIAS` | Milik user penerima. Jika alias kosong, pakai `ADDR_SHORT` |
| `ADDR_SHORT` | 6 karakter awal + `...` + 4 karakter akhir → `0x96eb...1a5c` |
| `LIST_NAME` | Nama list yang memuat wallet ini. Jika lebih dari satu, digabung dengan `, `. Jika tidak ada, `Wallet Track` |
| `ACTION_EMOJI` | 📈 BUY · 📉 SELL · 🔄 TRANSFER |
| `AMOUNT_USD` | 2 desimal → `285.51` |
| `MCAP` | Pemisah ribuan koma → `21,325` |
| `EXPLORER_URL` | Untuk Solana memakai path `/account/` |

Baris tambahan yang bisa dinyalakan user di halaman Profil:

- `🏷️ ⚡🐋 Split Arrow Whale` — label kategori wallet
- `⏱️ Tracked: 69 hari` — lama tracking
- `🔗 Tx: <link transaksi>` — link ke transaksi

Contoh hasil:

```
        📣 WALLET TRACKER ALERT 🔭

🪙 STOXPREPS | STOX (🐦 Robinhood)
0xb4d3a6097b765d094c0950ac03e917a2773eef4d

🎯 Target: 📂 @the wallet (0x96eb...1a5c) (https://robin.etherscan.io/address/0x96eb6b1b30ce315ff1acf21e4f29318a93cf1a5c)
📁 Wallet Track
📉 Action: SELL
💰 Amount: 285.51$
📊 Mcap: $21,325
```

### 9.4 Menghubungkan Telegram (per user)

1. User membuka halaman 👤 Profil dan klik **"Hubungkan Telegram"**.
2. Aplikasi membuat deep link sekali pakai `t.me/<bot>?start=<kode>` yang berlaku 10 menit.
3. User menekan **Start** di bot. Bot menyimpan `chat_id` user tersebut.
4. User bisa memutuskan koneksi kapan saja, atau lewat perintah `/stop` di bot.

Perintah bot:

| Perintah | Fungsi |
|---|---|
| `/start` | Menghubungkan akun |
| `/stop` | Memutuskan akun |
| `/mute <durasi>` | Membisukan notifikasi sementara |
| `/lists` | Melihat list dan status notifikasinya |
| `/status` | Cek koneksi |

### 9.5 Pengiriman

- Antrean pengiriman menjaga batas Telegram (±30 pesan/detik global, 1 pesan/detik per chat). Ada retry dengan backoff.
- Semua alert disimpan di riwayat. Riwayat bisa difilter dan di-export.
- Feed **Live Alerts** di dalam aplikasi dengan auto-refresh.

---

## 10. Halaman Aplikasi (Streamlit multipage)

| Halaman | Akses | Isi |
|---|---|---|
| 🏠 Dashboard | Semua | Ringkasan wallet per kategori, alert terbaru, wallet teraktif, wallet yang paling lama ditrack |
| 👛 Wallets | Semua | Tabel wallet + kolom "Ditrack", form tambah/edit, import/export |
| 📋 Lists | Semua | Kelola list, isi tiap list, ringkasan per list |
| 🕸️ Relations | Semua | Graf visual + form relasi |
| 🔔 Alerts | Semua | Feed live + riwayat |
| 👤 Profil | Semua | Hubungkan Telegram, preferensi notifikasi, baris tambahan template, zona waktu |
| ⚙️ Settings | **Owner** | Batas kategori winrate & aset, parameter winrate (jendela waktu, minimum trade, batas WIN), chain & explorer, API key, token bot Telegram, interval polling, kuota default |
| 🛡️ Admin | **Owner** | Daftar user (blokir/aktifkan, kuota per user), mode pendaftaran, status worker & webhook, pemakaian kuota API, log error, broadcast pesan ke semua user via Telegram |

---

## 11. Arsitektur & Teknologi

```
                   ┌────────────── VPS (Docker Compose) ───────────────┐
 Browser ──HTTPS──▶│ Caddy ──▶ Streamlit app (UI, multi-user)           │
                   │       └─▶ FastAPI (webhook receiver) ──┐           │
 Helius/Alchemy ──▶│                                        ▼           │
                   │               Worker (scheduler, stats, notifier)  │
                   │                         │                          │
                   │                    PostgreSQL                      │
                   └────────────────────────┬───────────────────────────┘
                                            ▼
                                     Telegram Bot API
```

| Komponen | Teknologi |
|---|---|
| UI | Streamlit (≥ 1.42, untuk `st.login`) |
| Auth | OIDC (Google) melalui `st.login` + Authlib |
| Database | PostgreSQL 16 + SQLAlchemy 2 + Alembic (migrasi) |
| Worker | Python + APScheduler (polling, hitung statistik, antrean notifikasi) |
| Webhook | FastAPI + Uvicorn |
| Data on-chain | Helius (Solana), Alchemy (semua EVM, termasuk Robinhood & BNB). Etherscan V2 / Blockscout opsional |
| Harga & mcap | DexScreener, GeckoTerminal |
| Graf | streamlit-agraph |
| Telegram | python-telegram-bot |
| Reverse proxy | Caddy (HTTPS otomatis) |
| Deploy | Docker Compose di VPS (minimal 2 vCPU / 4 GB RAM) |

### 11.1 Skema database (garis besar)

```
users(id, email, name, role[owner|user], status, tg_chat_id, timezone, prefs_json, created_at)
chains(id, key, family[evm|solana], name, emoji, explorer_url, enabled)
wallets(id, chain_id, address, winrate, trades_count, realized_pnl, winrate_cat,
        asset_usd, asset_cat, last_active_at, stats_updated_at)            -- global
user_wallets(id, user_id, wallet_id, alias, icon, tags, notes, notify_on,
             min_amount_usd, tracked_since, deleted_at)                    -- privat
lists(id, user_id, name, icon, color, description, notify_on, created_at)
list_items(list_id, user_wallet_id, added_at)
relations(id, user_id, from_uw_id, to_uw_id, name, type, direction, strength, color, notes)
transactions(id, wallet_id, chain_id, tx_hash, action, token_address, token_symbol,
             amount_usd, mcap, block_time)                                 -- global
alerts(id, user_id, transaction_id, sent_at, status)
graph_layouts(user_id, user_wallet_id, x, y)
settings(key, value)                                                       -- global, owner
audit_log(id, actor_id, action, payload, created_at)
```

---

## 12. Non-Fungsional

- **Kecepatan**: halaman terbuka < 2 detik untuk user dengan 500 wallet.
- **Kecepatan notifikasi**: < 10 detik setelah transaksi on-chain (webhook), < 60 detik (polling).
- **Keamanan**:
  - API key dan token bot hanya disimpan di `.env` atau terenkripsi di database. Tidak pernah ditampilkan utuh di UI.
  - Semua query database selalu difilter berdasarkan `user_id`.
  - Pengecekan peran owner dilakukan di server untuk setiap halaman admin.
  - HTTPS wajib.
- **Read-only**: aplikasi tidak pernah meminta private key atau seed phrase.
- **Ketahanan**: retry dengan backoff, menghormati rate limit API, dan service restart otomatis (`restart: unless-stopped`).
- **Backup**: `pg_dump` harian, disimpan 7 hari.
- **Disclaimer** di aplikasi: data hanya informasi, bukan saran finansial.

---

## 13. Roadmap

| Versi | Cakupan |
|---|---|
| **MVP** | Login + peran owner, CRUD wallet & list, lama tracking, relasi + graf, polling EVM (1 chain) + Solana, notifikasi Telegram dengan template, halaman Settings & Admin dasar |
| **v0.2** | Webhook Helius/Alchemy, winrate & aset otomatis, multi-chain EVM, import/export, kuota user |
| **v0.3** | Deteksi relasi otomatis (sumber funding pertama), statistik "sejak ditrack", broadcast admin, perintah bot `/mute` dan `/lists` |
