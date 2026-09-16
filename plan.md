# Plan — Menstabilkan samp-discord-connector: fix timeout init di Linux

## Diagnosis (hasil audit kode + panel verifikasi, 2026-09-12)

Gejala: `timeout while initializing data.` + retry senyap selamanya, **hanya di Linux**; token & intents sama jalan di Windows.

Rantai kegagalan (semua terverifikasi file:line):

1. `Http` ctor spawn `NetworkThreadFunc` (src/Http.cpp:9-16).
2. Statement pertama: `if (!Connect()) return;` (src/Http.cpp:81-82). `Connect()` = resolve `discord.com:443` **sinkron tanpa deadline** (:260-268), TCP connect (:270-277), TLS handshake (:280-286). Error sekecil apa pun → thread keluar, **tidak pernah di-restart** (`ReconnectRetry` :304-329 hanya terjangkau dari dalam loop antrean yang tak pernah dimasuki).
3. `GET /gateway` sudah di-push fire-and-forget ke `m_Queue` (src/Network.cpp:9-29, src/Http.cpp:370). Konsumennya mati → callback tidak pernah jalan → `WebSocket::Initialize` **tidak pernah dipanggil** (Network.cpp:28 satu-satunya call-site).
4. Tanpa WebSocket → tidak ada HELLO/IDENTIFY/READY → `IsInitialized()` Guild/User/Channel/Command selamanya false (Guild.cpp:446, User.cpp:51, Channel.cpp:343, Command.cpp:149) → `WaitForInitialization` habis 20s (main.cpp:43-65) → log persis yang terlihat.
5. Retry 60s (main.cpp:135-147) menghancurkan + membuat `Http` baru → **satu percobaan Connect lagi** → gagal lagi → selamanya, senyap.
6. Semua error ditulis ke samplog (`Logger::Get()->Log(ERROR, ...)`), **bukan `logprintf`** → di server tanpa samplog aktif, operator tidak melihat apa-apa. Ini yang membuat token salah / intents mati / DNS mampet terlihat identik.

Pemicu spesifik Linux (kode-identik, perilaku runtime beda):
- **DNS**: glibc `getaddrinfo` serial per nameserver, timeout 5s×attempt — upstream UDP/53 yang di-drop (container/iptables) = stall 10-20s, persis melewati deadline 20s. Windows: DNS Client service, failover paralel ~1-2s.
- **Resolver 32-bit**: plugin .so 32-bit butuh NSS 32-bit (`libc6-i386` / `libnss_dns.so.2`). Image Docker ramping sering tidak punya → resolve gagal total meski `dig` (64-bit) sukses.
- **IPv6-first**: AAAA Cloudflare dijawab dulu; di jaringan container routinya sering blackhole (SYN hilang, bukan RST) → connect menggantung hingga `tcp_syn_retries` (~130an detik) atau deadline 30s beast.

Yang **sudah terbantahkan** (jangan kejar arah ini): API v10/gateway v10 masih normal 2026; handler tidak menumpuk antar-retry (WebSocket ikut dihancurkan); bukan 4014 intents (jalur itu mencetak pesan intent via logprintf, tidak ada di log); bukan token salah murni (token sama, Windows sukses).

---

## Fase 0 — Diagnostik operator (tanpa recompile, 5 menit, jalankan dulu)

Di shell container Linux (`/home/container`):

```bash
# 1. DNS: apakah resolve cepat?
time getent ahostsv4 discord.com
time getent ahostsv6 discord.com   # kalau menggantung >5s = IPv6 bermasalah

# 2. IPv4 vs IPv6 connect
curl -4 -sv --max-time 10 https://discord.com/api/v10/gateway   # harus 200 <2s
curl -6 -sv --max-time 10 https://discord.com/api/v10/gateway   # hang = blackhole v6

# 3. Resolver 32-bit (karena plugin ini .so x86!)
file components/discord-connector.so                            # pastikan ELF 32-bit
ldd -r components/discord-connector.so 2>&1 | grep -i nss
dpkg --print-foreign-architectures                              # harus ada i386
```

Mitigasi instan jika kena: `apt-get install libc6:i386` (untuk NSS 32-bit), set `nameserver 1.1.1.1` di `/etc/resolv.conf`, atau `ip6tables -A OUTPUT -p tcp --dport 443 -d <v6 discord> -j REJECT --reject-with icmp6-adm-prohibited` (REJECT, bukan DROP, supaya connect gagal instan bukan menggantung).

## Fase 1 — Perbaikan kode (P0: plugin tidak boleh mati senyap)

### 1.1 Thread HTTP jangan pernah keluar saat Connect gagal
`src/Http.cpp` `NetworkThreadFunc` (baris 81-82): ganti

```cpp
if (!Connect())
    return;
```

menjadi loop: `while (m_NetworkThreadRunning && !Connect()) sleep(2s)` — thread tetap hidup, antrean tetap menunggu. `Connect()` yang gagal di tengah jalan harus `m_SslStream.reset()` dulu supaya object tidak bocor. Deadlock `~Http` (join ke thread retry) aman karena loop cek `m_NetworkThreadRunning` tiap iterasi.

### 1.2 Deadline nyata untuk resolve + connect + handshake
- `resolve` sinkron tidak punya timeout asio → ubah `Http::Connect()` ke async-resolve dengan `steady_timer::expires_after(5s)` + `resolver.cancel()` (pola standar beast), **atau** (lebih malas, cukup) coba resolve dulu via cache: simpan hasil resolve sukses terakhir dan pakai sebagai alamat cadangan.
- Untuk `tcp_stream` sync: `expires_after` **dihormati** di operasi sinkron (verifier code-truth membenarkan), tapi letakkan **sebelum** `resolve` tidak bisa — jadi bagian resolve saja yang perlu diperbaiki. Set connect deadline 10s, handshake 10s.
- Urutkan hasil resolve: **coba IPv4 dahulu** (stable-partition `results` sebelum connect loop) — menghilangkan happy-eyeballs blackhole tanpa konfigurasi host. Terapkan juga di `WebSocket::OnResolve` (src/WebSocket.cpp:57-81) yang sudah async+expires 30s; turunkan 30s → 10s dan tetap iterate endpoint (`async_connect` beast hanya mencoba results yang diberikan sekaligus — pastikan fallback per-endpoint eksplisit dengan deadline pendek).

### 1.3 Semua kegagalan init harus tercetak ke konsol
Tambah `logprintf` (bukan hanya samplog) di:
- `Http::Connect()` tiap kegagalan (resolve/connect/TLS) — src/Http.cpp:253, 265, 274, 283
- `Network::Initialize` callback `/gateway` dengan status != 200 — src/Network.cpp:14-17
- `WebSocket::OnResolve/OnConnect/OnSslHandshake/OnHandshake` — src/WebSocket.cpp:64, 92, 113, 150
- Message format: `>> discord-connector: <what> <detail>` supaya muncul walau tanpa samplog.

### 1.4 Bedakan pesan: TOKEN vs INTENTS vs NETWORK
`WebSocket::OnRead` (src/WebSocket.cpp:246-292):
- Handle close code **4004** (Authentication failed) → `logprintf("bot token ditolak Discord — periksa discord_bot_token/DCC_BOT_TOKEN")`, **jangan** reconnect (perbaiki dulu sebelum retry).
- **4013/4014** (disallowed intents) → pesan intent sudah ada untuk 4014 via `stream_truncated`; tambahkan untuk jalur `closed` lain + code 4013, dan jangan reconnect tanpa instruksi.
- **op 9 Invalid Session** setelah Identify (src/WebSocket.cpp:403-405) sekarang `Identify()` ulang selamanya tanpa log → tambahkan counter; ≥2× beruntun tanpa READY → `logprintf` "gateway menolak sesi (token/intents?)".

### 1.5 Startup banner diagnostik
Di `Load()`/`onLoad()` (src/main.cpp:78-158, 358-435): cetak (tanpa value token): sumber token (env `DCC_BOT_TOKEN` / server.cfg / config.json open.mp), nilai `intents` heks, versi plugin, platform. Menghemat separuh debugging orang.

## Fase 2 — P1: robisius & koreksi kecil

- **`Command.cpp:147-200`**: `m_InitGuilds` di-decrement **di dalam DAN di luar** callback per guild → bisa mencapai 0 lebih awal / berputar; `m_Initialized++` bisa dobel atau tidak sama sekali tergantung timing. Rewrite sederhana: count `m_InitGuilds--` hanya sekali di dalam callback; jika `GetGuilds()` kosong, langsung parse global + increment. (Belum difinalkan panel — perlu unit-test manual, tandai `// ponytail:` kalau dipertahankan sederhana.)
- **`WebSocket::OnRead` parse exception**: `json::parse` (WebSocket.cpp:295) tanpa try → payload rusak melempar exception di luar try → **io_context thread mati** → stall senyap permanen. Bungkus try/catch, log, `Read()` lagi.
- **`Disconnect()` saat `_websocket` null** (WebSocket.cpp:173): reconnect timer tidak ter-arm → dead-end. Guard + log.
- Retry loop main (main.cpp:135-149): cap jumlah percobaan + backoff, dan setelah sukses `break` log "successfully loaded" — saat ini sukses telat tidak terdengar sama sekali.

## Fase 3 — Verifikasi & rilis

1. Build Linux 32-bit (CMake+Conan seperti README).
2. Repro environment: matikan DNS (`iptables OUTPUT dport 53 DROP`) → plugin harus tetap hidup, log jelas, connect lagi saat DNS pulih tanpa restart server.
3. Token salah sengaja → harus muncul pesan "token ditolak (4004)", bukan timeout senyap.
4. Intent privileged mati → pesan intent (jalur 4014) muncul via logprintf.
5. VPS tanpa IPv6 sehat → inisialisasi selesai < 20s (IPv4-first).
6. Uji jalur SA:MP (`plugins`) **dan** open.mp (`components`) — keduanya berbagi `WaitForInitialization`, fix berlaku dua-duanya.

## Prioritas
- **Fase 0** jalankan sekarang di servermu (5 menit, menentukan apakah host-side atau butuh build ulang).
- **1.1 + 1.3 + 1.4** = inti perbaikan (diff kecil, menghilangkan seluruh kelas "silent death").
- **1.2** = penstabil Linux-specific. Sisanya P1.
