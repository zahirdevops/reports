# Infrastructure Optimization Report
**Tanggal:** 2026-06-04  
**Engineer:** Firmansyah  
**Scope:** MySQL RDS + PHP App Servers (11 VM) + Kubernetes (prod-zo-api-v3)

---

## 1. Ringkasan Masalah Awal

| # | Masalah | Dampak |
|---|---|---|
| 1 | Error `no free connections available to host` berulang | Request gagal, user tidak bisa akses aplikasi |
| 2 | Memory server penuh (70–87%) | Server lambat, swap terpakai |
| 3 | Kernel sysctl tidak ter-apply dengan benar | Connection queue hanya 128, rentan drop request |

---

## 2. Root Cause Analysis

### 2.1 MySQL — Zombie Connections
- `wait_timeout = 86400` (24 jam) menyebabkan koneksi idle bertahan sangat lama
- Koneksi Sleep idle hingga **4.056 detik (67 menit)**
- Connection pool sisi aplikasi (Go/Lumen) kehabisan slot karena diblokir koneksi zombie

### 2.2 Kubernetes (prod-zo-api-v3) — Pool Config Salah
- `MYSQL_MAX_OPEN_CONNS=0` → **unlimited** koneksi per pod
- `MYSQL_CONN_MAX_LIFETIME=1h` >> `wait_timeout` MySQL (300s)
- Go menganggap koneksi valid 1 jam, padahal MySQL sudah kill setelah 5 menit → error "dead connection"

### 2.3 PHP-FPM — Memory Leak & Config Berlebihan
- `pm.max_children = 700` — terlalu besar, bisa spawn ratusan worker
- `request_terminate_timeout = 18000s` (5 jam) — worker dibiarkan hidup sangat lama, memory terus tumbuh
- Worker lama tumbuh hingga **2 GB RAM per proses**
- Total PHP-FPM menghabiskan **6.5 GB dari 15 GB RAM** di satu server

### 2.4 Kernel sysctl — Config Duplikat & Tidak Aktif
- `sysctl.conf` memiliki banyak **entry duplikat yang konflik**
- `net.core.somaxconn` di config = 100.000 tapi aktif hanya **128** (default kernel)
- `net.ipv4.tcp_max_syn_backlog` di config = 12.000 tapi aktif hanya **2.048**
- `vm.swappiness = 30` terlalu tinggi → swap sering terpakai

---

## 3. Tindakan yang Dilakukan

### 3.1 MySQL RDS

**Kill zombie connections:**
```sql
KILL <id>;  -- 18 koneksi idle >5 menit di-kill manual
```

**Ubah wait_timeout** (via Alibaba Cloud RDS Console):
```
wait_timeout: 86400 → 300 (5 menit)
```

**Cleanup data:**
- Hapus **1.103.093 expired access tokens** dari `zahironline_provider.access_tokens`
- Tabel turun dari **621 MB → 6 MB** (hemat 615 MB)
- Jalankan `OPTIMIZE TABLE` untuk reclaim disk space

| Kondisi | Sebelum | Sesudah |
|---|---|---|
| Koneksi Sleep terlama | 4.056 detik | < 300 detik |
| Sleep > 5 menit | 15+ koneksi | 0 |
| `wait_timeout` | 86.400 detik | **300 detik** |
| `access_tokens` size | 621 MB | **6 MB** |

---

### 3.2 Kubernetes — ConfigMap `cm-zo-api-v3`

> **Status: Teridentifikasi, belum di-deploy** — menunggu approval

Perubahan yang direkomendasikan pada `cm-zo-api-v3`:

```env
# Sebelum
MYSQL_MAX_OPEN_CONNS=0
MYSQL_CONN_MAX_LIFETIME=1h
PG_MAX_OPEN_CONNS=0
PG_CONN_MAX_LIFETIME=1h

# Sesudah
MYSQL_MAX_OPEN_CONNS=25
MYSQL_CONN_MAX_LIFETIME=200s
PG_MAX_OPEN_CONNS=10
PG_CONN_MAX_LIFETIME=200s
```

Deployment yang terdampak (semua pakai ConfigMap yang sama):

| Deployment | Replicas |
|---|---|
| dpl-zo-api-v3 | 2 |
| dpl-zo-api-v3-to-v2 | 3 |
| dpl-zo-api-v2-dashboard | 2 |
| dpl-zo-api-v2-report | 2 |
| dpl-zo-api-v3-cogs | 2 |
| dpl-zo-api-v3-inventory | 1 |
| dpl-zo-api-v3-product | 1 |
| dpl-zo-api-v3-migrate | 1 |
| **Total** | **14 pods** |

---

### 3.3 PHP-FPM — Semua 11 App Server

**Backup:** `/etc/php-fpm.d/server.conf.bak-20260604`

**File:** `/etc/php-fpm.d/server.conf`

| Parameter | Sebelum | Sesudah |
|---|---|---|
| `pm.max_children` | 700 | **15** |
| `pm.max_spare_servers` | 300 | **5** |
| `pm.max_requests` | 1000 | **200** |
| `request_terminate_timeout` | 18000s (5 jam) | **60s** |

**Formula max_children untuk VM baru:**
```
pm.max_children = (Total RAM MB - 2048) / 512

Contoh:
  8 GB  → 12
  16 GB → 28
  32 GB → 60
```

**Revert jika ada masalah:**
```bash
sudo cp /etc/php-fpm.d/server.conf.bak-20260604 /etc/php-fpm.d/server.conf
sudo kill -USR2 $(cat /run/php-fpm/php-fpm.pid)
```

---

### 3.4 Kernel sysctl — Semua 11 App Server

**Backup:** `/etc/sysctl.conf.bak-20260604`

**File:** `/etc/sysctl.conf` (dibersihkan dari duplikat)

| Parameter | Sebelum (aktif) | Sesudah |
|---|---|---|
| `net.core.somaxconn` | **128** | **65535** |
| `net.ipv4.tcp_max_syn_backlog` | 2.048 | **65535** |
| `net.ipv4.tcp_keepalive_time` | 7200 (2 jam) | **300** |
| `net.ipv4.tcp_keepalive_intvl` | 75 | **30** |
| `net.ipv4.tcp_keepalive_probes` | 9 | **5** |
| `net.ipv4.tcp_fin_timeout` | 60 | **30** |
| `vm.swappiness` | 30 | **10** |

**Revert jika ada masalah:**
```bash
sudo cp /etc/sysctl.conf.bak-20260604 /etc/sysctl.conf && sudo sysctl -p
```

---

## 4. Hasil Akhir

### Memory Server (sebelum vs sesudah)

| Server | Sebelum | Sesudah |
|---|---|---|
| 10.10.10.75 | 🔴 87% (13.4 GB) | ✅ 4% (581 MB) |
| 10.10.10.81 | 🟠 72% (11.1 GB) | ✅ 4% (580 MB) |
| 10.10.10.80 | 🟠 71% (11.0 GB) | ✅ 4% (616 MB) |
| 10.10.10.76 | 🟠 68% (10.5 GB) | ✅ 4% (582 MB) |
| 10.10.10.77 | 🟡 66% (10.2 GB) | ✅ 4% (576 MB) |
| 10.10.10.26 | 🟡 62% (9.6 GB) | ✅ 4% (618 MB) |
| Lainnya | 40–56% | ✅ 4–7% |

### MySQL Connections

| Kondisi | Sebelum | Sesudah |
|---|---|---|
| Total koneksi | 87 | 75 |
| Koneksi Sleep > 5 menit | 15+ | **0** |
| Idle terlama (app) | 4.056 detik | < 300 detik |

### Request Activity

- **~137 request/menit per server** (terdistribusi merata)
- **~1.500 req/menit total** dari 11 server
- Semua server aktif menerima dan memproses request

---

## 5. Rekomendasi Lanjutan

| Prioritas | Item | Keterangan |
|---|---|---|
| 🔴 Tinggi | Deploy Kubernetes ConfigMap fix | `MYSQL_MAX_OPEN_CONNS=25`, `MYSQL_CONN_MAX_LIFETIME=200s` — lakukan di luar jam sibuk |
| 🟠 Sedang | Cleanup data lama di MySQL | `email_service_provider_activity` (43 GB), `global_histories` (31 GB), `slack_queue` (543 MB processed) |
| 🟠 Sedang | Investigasi `notifications_queue` | 1.09 juta row semua status=0 sejak 2019 — apakah consumer mati? |
| 🟡 Rendah | Tambahkan primary key pada 120+ tabel | Terutama di `zahironline_application` dan `hms` |
| 🟡 Rendah | Apply config ke VM template | Gunakan `server.conf` dan `sysctl.conf` final sebagai base image |

---

*Report generated: 2026-06-04*
