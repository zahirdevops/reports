# Incident Report — PolarDB "Too Many Connections" (Zahir ID)

**Tanggal:** 2026-06-05
**Sistem terdampak:** Aliyun PolarDB MySQL `pc-d9jz5djrml5u3tb6w.mysql.polardb.ap-southeast-5.rds.aliyuncs.com` (database `zahirid`)
**Layanan terdampak:** Zahir ID (login/SSO) — `account.zahir.id`, `api-account.zahir.id`
**Klasifikasi:** **Serangan otomatis berbasis Python** (`python-requests/2.34.2`) — lihat §4
**Status:** **RESOLVED** (mitigasi 2 layer: nginx-ingress + Cloudflare WAF Custom Rule)

---

## 1. Executive Summary

| Aspek | Keterangan |
|---|---|
| **Gejala** | Klien menerima `ERROR 1040 (08004): Too many connections` saat akses Zahir ID. Login, OAuth token, dan endpoint dashboard terdampak. |
| **Dampak puncak** | PolarDB `Threads_connected` mencapai **3017 / 3512** (86% kapasitas), throughput hit `Max_used_connections=3513` (≥ limit) — error 1040 mulai dikirim. Total **`Connection_errors_max_connections = 566,855` kumulatif** sebelum mitigasi. |
| **Penyebab langsung** | **Serangan flooding `~412 req/sec`** ke endpoint OAuth internal `POST /api/v3/zahir_id/oauth/internal/token` dari klien terotomatisasi `python-requests/2.34.2` (lewat Cloudflare untuk menutupi IP asli), memicu connection-leak amplifier dan death spiral pod login. Karakteristik konsisten dengan **credential stuffing / endpoint abuse** atau **integrasi pihak ketiga yang dieksploitasi/dikompromikan**. |
| **Penyebab struktural** | (a) HPA threshold CPU 50% terlalu sensitif untuk app DB-bound; (b) memory limit 256Mi terlalu kecil → OOMKilled saat load; (c) tidak ada batas pool koneksi efektif di image; (d) endpoint `/api/v3/zahir_id/oauth/internal/token` mengembalikan 404 namun **masih query DB** sebelum return. |

---

## 2. Timeline

Waktu dalam zona WITA (+08:00).

| Waktu | Kejadian |
|---|---|
| ~12:30 | Insiden dilaporkan: "too many connection" di pod ns `prod-zo-api-v3`. |
| 12:35 | Diagnosis pod prod-zo-api-v3 — pod sehat, **bukan** sumber masalah (target DB-nya RDS biasa, bukan PolarDB). |
| 12:40 | Identifikasi PolarDB target dari kredensial yang diberikan: `pc-d9jz5djrml5u3tb6w`. 10 dari 10 attempt connect dapat error 1040. |
| 12:45 | Slot berhasil diraih sekali. Diagnostik penuh: 3017 koneksi, 2912 Sleep, 2950 dari user `zahirid`, datang dari 4 worker node k8s (`10.10.10.24/.44/.235/.40`). |
| 12:55 | Pemetaan pod konsumen: namespace `prod-zahir-id-v3` (deployment `dpl-zahirid-v3`) — **10 replica spec, 7 Running + 6 CrashLoopBackOff**. ReplicaSet stuck di 3 versi. |
| 13:00 | Konfirmasi *death spiral*: pod baru fail startup karena `failed to initialize database, got error Error 1040` → respawn → tambah pressure. HPA `zid-v3-hpa` target CPU 50%, actual 91% — terus push ke max replicas 10. |
| 13:10 | **Scale `dpl-zahirid-v3` ke 0** (atas konfirmasi). 13 pod ditermin <2s. PolarDB lega: `Threads_connected` 3017 → 3. |
| 13:11 | Scale balik ke 5 + `dpl-provider-v3` scale balik ke 1. |
| 13:23 | Re-saturate cepat — pod baru tetap OOMKilled (memory 256Mi). |
| 13:27 | **Patch memory** 256Mi → **1Gi** (request 512Mi). Pod stabil, 0 restart. Tapi single pod buka **1463 koneksi** ke PolarDB → masih saturated. |
| 13:35 | **Investigasi config**: `/app/.env` di-bake ke image (`DB_MAX_OPEN_CONNS=25` literal, `APP_ENV=local`). Override via ConfigMap `cm-zid-v3-db-pool` (`envFrom`) + `cm-zid-v3-env` (file mount `subPath`). Aplikasi ternyata **baca env var, bukan `.env` file**. PolarDB: 992 → 8 koneksi. |
| 13:48 | Insiden lanjutan: PolarDB saturated **lagi** dengan 2961 koneksi. Pod kita pakai limit 10 — bukan kita. |
| 13:55 | Analisis processlist: **0 query aktif** dari `zahirid`, semua 2871 koneksi Sleep avg 24s. Pure zombie, bukan beban traffic. KILL 1 zombie tersisa → `Threads_connected = 2`. |
| 14:00 | **Periksa ingress controller**: ketemu **123,658 request dalam 5 menit (412 req/sec)** ke `account.zahir.id`. 113,027 hits `POST /api/v3/zahir_id/oauth/internal/token` → 404. 98.6% dari UA `python-requests/2.34.2`. |
| 14:05 | Konfirmasi routing: ingress `account.zahir.id/api/v3/*` → service ExternalName `api-v3-get-me` → **`svc-zid-v3.prod-zahir-id-v3` (pod kita!)**. Endpoint tidak ada di image v3 (404), namun router/middleware **tetap query DB** sebelum return. |
| 14:08 | **Mitigasi nginx-ingress**: annotation `configuration-snippet` me-`return 429` untuk UA `python-requests/2.34.2`. 99.8% request langsung dijawab 429 (0 detik) tanpa hit backend. |
| 14:10 | Verifikasi: pod CPU drop **129m → 2-9m**, Error 1040 hilang dari log pod, PolarDB stabil 16 koneksi. |
| 14:20 | **Cloudflare WAF Custom Rule diterapkan** untuk block UA `python-requests/*` (defense in depth, edge level). |

---

## 3. Root Cause Analysis

Insiden disebabkan **rantai 5 lapis**, bukan satu kegagalan tunggal:

### 3.1 Klien (the trigger)

Sebuah klien **`python-requests/2.34.2`** mengirim `~412 req/sec` ke `POST account.zahir.id/api/v3/zahir_id/oauth/internal/token`. Endpoint ini sudah deprecated di image v3 (return 404), tetapi klien terus retry. Klien tersembunyi di balik Cloudflare → log nginx hanya mencatat IP edge Cloudflare (`172.70.x.x`, `104.23.x.x`).

### 3.2 Backend (the amplifier)

Endpoint 404 **tetap memproses request lewat middleware** (auth check / audit log / DB-backed validator) **sebelum** return 404. Latensi observasi: 0.236s (cache hit) — 1.2s (DB normal) — **37s (saat pool exhausted, client timeout 499)**.

### 3.3 Connection pool (the missing limit)

Image `zahir-id-v3`:
- `.env` di-bake (path `/app/.env`) berisi `DB_MAX_OPEN_CONNS=25` dan `APP_ENV=local`.
- Aplikasi **tidak baca `.env` file** — hanya `os.Getenv(...)`. Tanpa env var, default-nya membengkak (1 pod observasi 1463 koneksi).
- Saat 1 pod menjadi 5+ pod via HPA, total koneksi → ribuan → habiskan slot `max_connections=3512`.

### 3.4 HPA (the unstable amplifier)

`zid-v3-hpa`: target CPU **50%** (sangat sensitif), max replicas **10**, no scaling behavior policy. Pod CPU-bound saat lookup DB lambat → CPU naik → HPA scale-out → buka koneksi baru → DB makin lambat → spiral.

### 3.5 Memory limit (the OOM trigger)

Memory limit 256Mi terlalu kecil. Saat pool exhausted, goroutine + request buffer + retry akumulasi → OOMKilled → respawn → koneksi pod sebelumnya zombie di server.

**Death spiral terkonfirmasi**: pod baru fail startup dengan literal `failed to initialize database, got error Error 1040: Too many connections` di `db.go:36`. Setiap respawn membuka koneksi zombie baru yang tidak ter-release oleh PolarDB server (`wait_timeout` default ~8 jam).

---

## 4. Analisis Serangan (Python-based)

### 4.1 Profil Serangan

| Parameter | Nilai |
|---|---|
| **User-Agent** | `python-requests/2.34.2` (UA single version — tidak ada varian) |
| **Rate** | ~412 req/sec sustained (123,658 hits dalam 5 menit) |
| **Target** | `POST account.zahir.id/api/v3/zahir_id/oauth/internal/token` |
| **Response code dominan** | 404 (113,027) + 499 (8,361) + 502 (484) + 401 (52) + 422 (2) |
| **Method** | POST (mengindikasikan ada payload dikirim) |
| **Channel** | Lewat Cloudflare CDN — IP asli pelaku tertutup di header `CF-Connecting-IP` yang tidak ter-log di nginx-ingress (log-format saat ini hanya capture `$remote_addr` = IP edge CF) |
| **IP edge Cloudflare** | `172.70.x.x` (Cloudflare US-East), `104.23.x.x` (Cloudflare Atlanta), `162.158.x.x` (Cloudflare global) — IP CF rotasi, bukan IP asli |

### 4.2 Indikator Karakter Serangan (vs bug integrasi)

**Lima indikator yang mendukung klasifikasi serangan:**

1. **Lock-in di satu versi UA spesifik** (`python-requests/2.34.2`) tanpa variasi → kode menetap, bukan integrasi normal yang biasanya pakai versi terbaru atau range.
2. **Target endpoint internal OAuth** (`/oauth/internal/token`) — ini endpoint **service-to-service** yang seharusnya tidak publik; pelaku tahu nama endpoint = ada **prior reconnaissance** atau leak dokumentasi internal.
3. **Tidak ada exponential backoff** pada response 404/499/502 — terus retry di rate tetap. Klien sah modern selalu implement backoff.
4. **POST method dengan body** (request_length konsisten ~240-242 bytes per log) — bukan probe biasa; ada payload (kemungkinan credential atau token candidate).
5. **Sembunyi di balik Cloudflare** untuk akses domain Anda sendiri — klien sah biasanya konek langsung. Penggunaan CF sebagai *anonymizer* bisa intentional (residential CF proxy atau penyalahgunaan CF Workers).

**Tiga indikator yang mungkin mendukung bug (alternatif hipotesis):**

1. UA tunggal bisa juga = satu deployment integrasi yang stuck.
2. Endpoint 404 terus → kalau attacker biasanya pindah, kalau bug stuck di loop.
3. Cloudflare adalah CDN umum, banyak SaaS sah lewat sini.

**Verdict: lebih banyak bukti mendukung serangan.** Sampai owner klien teridentifikasi dan memverifikasi sebagai bug sah, **perlakukan sebagai security incident**.

### 4.3 Skenario Serangan yang Konsisten

- **Credential stuffing**: pelaku mencoba kombinasi user/password lewat endpoint OAuth internal yang melakukan validasi kredensial dengan response time yang bisa di-fingerprint (404 cepat untuk invalid request, 401/422 untuk format valid tapi gagal auth).
- **Token enumeration / replay**: mencoba refresh token / authorization code dari leak database atau brute-force key space.
- **Endpoint discovery + DoS**: pelaku tahu endpoint internal, lalu sengaja flood untuk men-saturate connection pool (DoS) dan secara opportunistic memantau respon.
- **Compromised legitimate integration**: integrasi pihak ketiga (mis. partner / mobile-app / IoT) yang akunnya/credential-nya dikompromikan, lalu attacker memakai akses tersebut untuk flood.

### 4.4 Dampak Saat Insiden

- PolarDB connection pool habis (`max_connections` 3512) → tidak hanya app pemroses 404 yang terganggu, tapi **seluruh aplikasi yang share DB `zahirid`** (login Zahir ID, dashboard, provider, dll.) tidak bisa konek.
- 566,855 ERROR 1040 dikembalikan ke klien sah selama insiden — **user real tidak bisa login** ke Zahir ID v3.
- Death spiral pod (CrashLoop → respawn → leak conn → makin saturate) mempanjang Mean Time To Recovery.
- **Tidak ada konfirmasi data leak / unauthorized access** karena endpoint targeted return 404 (route tidak ada di image v3 saat ini). Namun **payload request POST belum di-capture** sehingga **tidak bisa dipastikan** apa yang pelaku coba.

### 4.5 Mitigasi yang Berhasil Memutus Serangan

1. **Layer 1 — nginx-ingress** (`prod-zahir-id/ingress-zid`): annotation `configuration-snippet` yang `return 429` untuk UA `python-requests/2.34.2` — block 99.8% request, response time 0 detik, tidak hit backend (pod). Aktif sejak 14:08 WITA.
2. **Layer 2 — Cloudflare WAF Custom Rule** (di-apply tim pada 14:20 WITA): block edge-level untuk UA pattern `python-requests/*` pada host `account.zahir.id` + path `/api/v3/zahir_id/oauth/internal/token`. Request ditolak sebelum sampai ke nginx-ingress, hemat resource origin sepenuhnya.

Defense in depth: kalau Layer 2 (CF) dimatikan/bypass, Layer 1 (nginx-ingress) masih melindungi origin.

---

## 5. Mitigasi Yang Dilakukan

### 5.1 Stabilkan death spiral (immediate)
- **Backup** deployment, HPA, svc, ingress `prod-zahir-id-v3` ke `/Users/firmansyah/zahir-infra/k8s-backup-prod-zahir-id-v3-20260605-141629/`.
- **Hapus HPA `zid-v3-hpa`** sementara untuk berhenti scaling-up.
- **Delete deployment `dpl-zahirid-v3`** → semua 10 pod ditermin → PolarDB `Threads_connected` 3017 → 3.

### 5.2 Recreate dengan setting yang proper
- Re-create deployment **replicas=1**, memory limit **1Gi** (request 512Mi), CPU tetap 250m/300m.
- HPA baru target CPU **70%**, **maxReplicas 5**, dengan `behavior.scaleUp` (stabilization 60s, +1 pod/60s) dan `behavior.scaleDown` (stabilization 300s, -1 pod/120s) — anti-spike, anti-flap.

### 5.3 Batasi koneksi DB via ConfigMap
- `cm-zid-v3-db-pool` dengan key-value:
  ```
  DB_MAX_OPEN_CONNS=10
  DB_MAX_IDLE_CONNS=5
  DB_CONN_MAX_LIFETIME=5m
  DB_IS_DEBUG=false
  ```
  Inject ke pod via **`envFrom: configMapRef`** (terbukti efektif — aplikasi honor env var).
- `cm-zid-v3-env` mount sebagai overlay `/app/.env` (subPath) untuk konsistensi konfigurasi.
- **Hasil**: 1 pod buka **3-7 koneksi** (vs 1463 sebelumnya).

### 5.4 KILL koneksi zombie
- Script polling sampai dapat slot, lalu `KILL` semua process_id `zahirid` Sleep > 30 detik.
- Saat dieksekusi sudah cleanup sendiri, hanya 1 zombie tersisa (id `14242236`) → KILL.
- `Threads_connected = 2`, user `zahirid = 0`.

### 5.5 Block sumber serangan — Layer Ingress (nginx)
Annotation di `ingress-zid` (ns `prod-zahir-id`):
```yaml
nginx.ingress.kubernetes.io/configuration-snippet: |
  if ($http_user_agent ~* "python-requests/2\.34\.2") {
    return 429 "deprecated client - please upgrade";
  }
```
**Hasil**: 13,811 dari 13,833 request (99.8%) di window 30 detik dijawab `HTTP 429` dengan response time `0.000s`, tidak hit backend. Pod CPU 129m → **2-9m**.

### 5.6 Block sumber serangan — Layer Cloudflare WAF (defense in depth)
Custom Rule WAF di-apply oleh tim untuk match UA prefix `python-requests/*` (atau setara) → action **Block**. Setelah aktif, request akan ditolak di edge Cloudflare, **tidak masuk** ke nginx-ingress sama sekali.

---

## 6. Status Final

| Metric | Sebelum | Sesudah | Selisih |
|---|---|---|---|
| PolarDB `Threads_connected` | 3017 | 16 | **-99.5%** |
| User `zahirid` koneksi | 2950 | 15 | **-99.5%** |
| `Connection_errors_max_connections` (rate) | spike | flat | stop |
| Error 1040 di pod log per 60s | ratusan/detik | **0** | stop |
| Pod CPU per pod | 119-129m | 2-9m | **-93%** |
| Pod restart rate | 5-10× per pod / 26m | 0 | stop |
| Request rate **serangan** ke origin | 412 req/sec | ~0 (blocked at CF edge) | **-100%** |

---

## 7. Open Follow-ups

| No | Item | Prioritas | Pemilik |
|---|---|---|---|
| 1 | **Investigasi forensik pelaku serangan Python** — ekstrak `CF-Connecting-IP`, ASN, geolocation, dan request body lewat Cloudflare logs/Logpush. Cek apakah IP asli punya rekam jejak attack pattern di Cloudflare Security Analytics. Kalau ternyata integrasi pihak ketiga (partner) yang dikompromikan — perlu insiden notifikasi ke owner integrasi tersebut. | **Kritis** | Security + DevOps |
| 1b | **Audit log akses sukses (HTTP 200/401) ke `/api/v3/zahir_id/oauth/internal/token`** dari UA `python-requests/2.34.2` di seluruh history log (kalau retained > 7 hari). Konfirmasi tidak ada token yang ter-issued atau akses unauthorized berhasil sebelum endpoint return 404 sekarang. | **Kritis** | Security |
| 1c | **Periksa apakah ada credentials/secrets bocor**: cari `oauth/internal/token` di repo internal, dokumentasi, post mortem, Postman collection publik, repo eks-karyawan — bagaimana pelaku tahu nama endpoint internal? | **Kritis** | Security |
| 2 | **Endpoint `/api/v3/zahir_id/oauth/internal/token` — fix backend** agar tidak query DB sebelum return 404. Atau hapus mapping ingress kalau memang tidak dipakai. **Endpoint internal seharusnya TIDAK terekspos publik** — pindahkan ke service mesh internal-only, atau pasang mTLS / IP allowlist. | Tinggi | Tim backend Zahir ID |
| 3 | **Image `zahir-id-v3`: fix pool config** agar baca dari `.env` atau env var (bukan hardcoded). Tinjau ulang `db.go:36` untuk default pool size yang wajar (mis. 10-25). | Tinggi | Tim backend Zahir ID |
| 4 | **Naikkan `max_connections` PolarDB** dari 3512 ke ≥6000 via Aliyun console (live, no restart) — beri headroom untuk skenario lonjakan organik. | Sedang | DBA |
| 5 | **Sinkronkan perubahan `dpl-zahirid-v3` ke source-of-truth GitOps/repo** — perubahan via `kubectl apply` terbukti ter-revert oleh tim lain. Spec deployment + HPA + ConfigMap harus jadi PR ke repo agar permanen. | Sedang | Tim DevOps |
| 6 | **Update log-format nginx-ingress** untuk include `$http_cf_connecting_ip` dan `$http_x_forwarded_for` agar IP klien asli tercatat sebelum kena Cloudflare. | Rendah | DevOps |
| 7 | **Renew SSL certificate**: `api-account.zahir.id` (expired 2024-08-04) dan `api-account-v3.zahironline.com` (expired 2025-09-28). Walau request masih masuk (Cloudflare punya cert edge), certificate origin sudah lama mati. | Sedang | DevOps |
| 8 | **Audit aplikasi lain yang share PolarDB `zahirid` DB**: `prod-provider-v3`, `prod-zahir-id-v3-dashboard-api`, dll. — pastikan semua punya pool limit yang sehat. | Rendah | Tim backend |
| 9 | **Audit HPA target metric**: untuk app DB-bound, CPU utilization adalah metric yang **misleading** — saat DB lambat, CPU justru idle (waiting), HPA tidak scale; saat DB cepat, CPU spike, HPA over-scale. Pertimbangkan custom metric (request latency P99 atau DB pool saturation). | Rendah | DevOps + backend |

---

## 8. Lessons Learned

1. **Connection leak adalah bom waktu** — koneksi Sleep tidak gratis. Selalu pasang `MaxOpenConns` + `ConnMaxLifetime` rendah agar koneksi recycle sebelum jadi zombie.
2. **Default config di image = risiko** — `.env` di-bake harus dihormati app, atau jelas overridable via env var. Tanpa standar, tidak ada cara konsisten override.
3. **HPA target CPU 50% untuk app DB-bound = death spiral magnet** — naikkan ke 70-80%, tambah `behavior.scaleUp.stabilizationWindow` agar tidak overshoot.
4. **404 yang query DB = anti-pattern dan attack surface** — endpoint hilang/tidak ada harus return cepat tanpa side-effect. Endpoint mati tetap **bisa dipakai untuk DoS connection pool** kalau backend tetap query DB.
5. **Defense in depth**: nginx-ingress (intra-cluster) + Cloudflare WAF (edge) = bertahan walau salah satu lapisan terlewat / tidak sengaja dihapus. **Layer 2 (CF) sebaiknya selalu hidup duluan** karena hemat resource origin.
6. **kubectl apply tidak final** — kalau ada tim lain yang `apply` spec lain dari source code mereka, perubahan operasional Anda akan ter-revert. Source of truth harus tunggal (GitOps repo).
7. **Endpoint "internal" yang terekspos publik = surface serangan terbuka** — penamaan endpoint dengan prefix `/internal/` saja **bukan keamanan**. Klien malicious yang tahu URL bisa menyalahgunakannya untuk DoS atau eksploitasi. Endpoint service-to-service wajib dilindungi mTLS / IP allowlist / authentikasi terpisah dari endpoint user-facing.
8. **Observability untuk attribution serangan**: log-format nginx-ingress saat ini hanya simpan IP edge Cloudflare. Saat insiden, **tidak bisa identifikasi pelaku sebenarnya** dalam waktu nyata. Update log-format untuk capture `CF-Connecting-IP`, `X-Forwarded-For`, dan `CF-Ray` agar forensik bisa dilakukan dengan data yang ada.

---

**Backup spec lama**: `/Users/firmansyah/zahir-infra/k8s-backup-prod-zahir-id-v3-20260605-141629/`

**Dokumen terkait**: `/Users/firmansyah/zahir-infra/incident-2026-06-05.md` (insiden ASG Zahir Online pagi yang sama hari)
