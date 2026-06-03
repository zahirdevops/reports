# Stress Test Report: Lumen vs Golang
**Endpoint:** `GET /api/v2/user_companies`
**Tanggal:** 2026-06-04 | **Tool:** ApacheBench 2.3 | **Monitoring:** Grafana (Kubernetes)

---

## Executive Summary

```
┌─────────────────────────────────────────────────────────────────┐
│              HASIL KESELURUHAN: GOLANG UNGGUL                   │
├──────────────────────┬──────────────┬──────────────┬───────────┤
│ Metrik               │    Lumen     │   Golang     │  Rasio    │
├──────────────────────┼──────────────┼──────────────┼───────────┤
│ Throughput (RPS)     │  0.79–1.90   │  14–27       │ Go 14–18x │
│ P95 Latency          │  62–125 det  │  0.9–7.6 det │ Go 16–67x │
│ Failure Rate         │  1–7.2%      │  0%          │ Go ∞ baik │
│ Memory Peak          │  763 MiB     │  95 MiB      │ Go 8x     │
│ CPU Peak             │  0.600 cores │  0.056 cores │ Go 10.7x  │
│ Response Size        │  92 KB       │  21 KB       │ Go 4.4x   │
└──────────────────────┴──────────────┴──────────────┴───────────┘
```

---

## Grafik Perbandingan

### Throughput (Requests per Second) — Lebih Tinggi Lebih Baik

```
  Light Load (10c)
  Golang  ████████████████████████████████████ 14.38 req/s
  Lumen   ██ 0.79 req/s

  Medium Load (50c)
  Golang  ████████████████████████████████████████████████████ 26.60 req/s
  Lumen   ████ 1.90 req/s

  Heavy Load (100c)
  Golang  █████████████████████████████████████████████████████ 27.18 req/s
  Lumen   ██ ~1-2 req/s (est.)
```

### P95 Latency (ms) — Lebih Rendah Lebih Baik

```
  Light Load (10c)
  Golang  █ 925ms
  Lumen   ████████████████████████████████████████████████████████████████ 62,684ms

  Medium Load (50c)
  Golang  ██ 2,655ms
  Lumen   ████████████████████████████████████████████████████████████████ 125,293ms

  Heavy Load (100c)
  Golang  ████ 7,636ms
  Lumen   ████████████████████████████████████████████████████████████████ >125,000ms
```

### Memory Usage Peak — Lebih Rendah Lebih Baik

```
  Golang  ████████████ 95.4 MiB
  Lumen   ████████████████████████████████████████████████████████████████████████████████████████████████ 763 MiB

  Rasio: Lumen menggunakan 8x lebih banyak memori
```

### CPU Usage Peak — Lebih Rendah Lebih Baik

```
  Golang  ████ 0.056 cores
  Lumen   ████████████████████████████████████████████████████████████████████████████████████████████████ 0.600 cores

  Rasio: Lumen menggunakan 10.7x lebih banyak CPU
```

### Failure Rate — Lebih Rendah Lebih Baik

```
  Light Load (10c)
  Golang  ▪ 0.0%
  Lumen   █ 1.0%

  Medium Load (50c)
  Golang  ▪ 0.0%
  Lumen   ███████ 7.2%

  Heavy Load (100c)
  Golang  ▪ 0.0%
  Lumen   ███████████████ >15% (est.)
```

---

## Tabel Perbandingan Lengkap

| Item Pengujian | Lumen | Golang |
|---|---|---|
| **BASELINE (Single Request)** | | |
| Response Time | 931ms | 454ms |
| TTFB | 853ms | 453ms |
| Connect Time | 44ms | 104ms |
| Response Size | 92,818 bytes | 21,071 bytes |
| HTTP Status | 200 OK | 200 OK |
| **LIGHT LOAD (10 concurrent / 100 requests)** | | |
| Requests per Second | 0.79 req/s | 14.38 req/s |
| Durasi Total Test | 126.23 detik | 6.95 detik |
| Median Latency (P50) | 946ms | 604ms |
| P75 Latency | 1,247ms | 755ms |
| P90 Latency | 32,530ms | 829ms |
| P95 Latency | 62,684ms | 925ms |
| P99 Latency | 125,306ms | 1,063ms |
| Max Latency | 125,306ms | 1,063ms |
| Failed Requests | 1 / 100 (1%) | 0 / 100 (0%) |
| Transfer Rate | 71.83 KB/s | 306.40 KB/s |
| **MEDIUM LOAD (50 concurrent / 500 requests)** | | |
| Requests per Second | 1.90 req/s | 26.60 req/s |
| Durasi Total Test | 262.95 detik | 18.80 detik |
| Median Latency (P50) | 1,117ms | 1,720ms |
| P75 Latency | 7,036ms | 2,045ms |
| P90 Latency | 95,172ms | 2,373ms |
| P95 Latency | 125,293ms | 2,655ms |
| P99 Latency | 125,415ms | 3,463ms |
| Max Latency | 126,310ms | 3,831ms |
| Failed Requests | 36 / 500 (7.2%) | 0 / 500 (0%) |
| Transfer Rate | 161.68 KB/s | 566.70 KB/s |
| **HEAVY LOAD (100 concurrent / 1000 requests)** | | |
| Requests per Second | ~1–2 req/s (est.) | 27.18 req/s |
| Durasi Total Test | >500 detik (est.) | 36.79 detik |
| Median Latency (P50) | >50,000ms (est.) | 2,791ms |
| P75 Latency | >100,000ms (est.) | 3,965ms |
| P90 Latency | >125,000ms (est.) | 5,635ms |
| P95 Latency | >125,000ms (est.) | 7,636ms |
| P99 Latency | >125,000ms (est.) | 16,181ms |
| Max Latency | >125,000ms (est.) | 22,684ms |
| Failed Requests | >15% (est.) | 0 / 1000 (0%) |
| Transfer Rate | <100 KB/s (est.) | 579.07 KB/s |
| **RESOURCE USAGE (Kubernetes / Grafana)** | | |
| Memory — Baseline | ~0 MiB | ~0 MiB |
| Memory — Peak saat test | **763 MiB** | **95.4 MiB** |
| Memory — Rasio peak | 8x lebih besar | baseline |
| Memory — Recovery | Kembali ke 0 setelah ~5 menit | Kembali ke 0 dalam ~2 menit |
| CPU — Baseline | ~0 cores | ~0 cores |
| CPU — Peak saat test | **0.600 cores** | **0.056 cores** |
| CPU — Rasio peak | 10.7x lebih tinggi | baseline |
| CPU — Durasi spike | ~8 menit (05:00–05:08) | ~4 menit (05:00–05:04) |
| Deployment name | dpl-zo-v2-api | dpl-zo-v2-api-go |
| **RINGKASAN KEUNGGULAN** | | |
| Throughput vs lawan | 14–18x lebih rendah | **14–18x lebih tinggi** |
| P95 Latency vs lawan | 16–67x lebih lambat | **16–67x lebih cepat** |
| Reliabilitas | Gagal mulai 10c | **0 kegagalan** |
| Memory efisiensi | 763 MiB peak | **95 MiB peak (8x lebih hemat)** |
| CPU efisiensi | 0.600 cores peak | **0.056 cores (10.7x lebih hemat)** |
| Response payload | 92KB (boros) | **21KB (efisien)** |

> *Heavy load Lumen tidak dijalankan penuh — medium load sudah menunjukkan degradasi kritis (P95 = 125 detik, failure 7.2%, memory spike ke 763 MiB)*

---

## Analisis Resource Usage

### Memory

Grafana menunjukkan pola yang sangat berbeda antara keduanya:

- **Lumen** membutuhkan hingga **763 MiB** di puncak load. Ini konsisten dengan model PHP-FPM di mana setiap worker proses memuat ulang seluruh framework (Lumen bootstrap, Eloquent ORM, autoloader) untuk setiap request. Di bawah 50 concurrent users, PHP-FPM spawn banyak worker proses secara bersamaan, masing-masing mengonsumsi memory terpisah tanpa sharing.

- **Golang** hanya menyentuh **95.4 MiB** di puncak — **8x lebih hemat**. Golang menggunakan goroutine yang berbagi heap, sehingga overhead per-request jauh lebih kecil. GC Golang juga efisien — memory kembali normal dalam ~2 menit setelah test selesai, vs Lumen yang butuh ~5 menit.

### CPU

- **Lumen** spike ke **0.600 cores** dan berlangsung selama ~8 menit. Tingginya CPU usage di PHP berkaitan dengan overhead bootstrapping framework per request, serialisasi JSON, dan query execution yang tidak dioptimasi.

- **Golang** hanya menyentuh **0.056 cores** — **10.7x lebih efisien**. Spike lebih singkat (~4 menit) dan turun tajam karena Golang selesai melayani request jauh lebih cepat.

### Implikasi Biaya (Kubernetes)

Dengan resource usage Lumen yang 8–10x lebih besar, untuk traffic production yang sama dibutuhkan:
- ~8x lebih banyak memory quota di pod
- ~10x lebih banyak CPU request/limit
- Berpotensi butuh horizontal pod autoscaling lebih agresif
- Biaya infrastruktur cloud yang signifikan lebih tinggi

---

## Analisis & Temuan Kritis

### 1. Throughput
Golang secara konsisten **14–18x lebih tinggi throughput** di semua level load. Golang juga menunjukkan scalability yang baik: RPS naik dari 14 → 27 saat concurrency meningkat. Lumen hampir stagnan di 0.79 → 1.90 RPS.

### 2. Tail Latency (P90/P95/P99)
Perbedaan paling ekstrem ada di tail latency. Light load (10c): Go P95 = **0.9 detik**, Lumen P95 = **62.7 detik** (selisih 67x). Pola ini mengindikasikan PHP-FPM worker pool exhaustion — saat semua worker terpakai, request berikutnya antri dan menunggu.

### 3. Reliabilitas
- **Golang**: 0 failed requests dari total 1,600 request (semua skenario)
- **Lumen**: Gagal bahkan di light load (1%), meningkat ke 7.2% di medium load

Kegagalan Lumen dikategorikan sebagai **Length mismatch** — response terpotong karena PHP-FPM worker timeout sebelum selesai mengirim response.

### 4. Response Payload
Lumen mengembalikan **92KB vs 21KB** (4.4x lebih besar) untuk query identik. Kemungkinan penyebab:
- Eloquent ORM menyertakan relasi yang tidak di-filter
- Serialisasi default yang tidak dikustomisasi
- Payload besar memperparah latency karena waktu transfer lebih lama

### 5. Pola Saturasi
Golang plateau di ~27 RPS mengindikasikan bottleneck ada di **upstream** (database), bukan di server Golang. Lumen bahkan tidak mencapai bottleneck upstream karena saturasi di layer PHP-FPM terlebih dahulu.

---

## Rekomendasi

| Prioritas | Item | Dampak |
|---|---|---|
| Kritis | Investigasi response Lumen 4.4x lebih besar (over-fetching) | Latency + bandwidth |
| Kritis | Tambah PHP-FPM worker pool jika tetap pakai Lumen | Throughput & failure rate |
| Tinggi | Implementasi response caching (Redis) untuk query berat | Latency reduction |
| Tinggi | Set memory limit pod Lumen minimum 1 GiB di Kubernetes | Stability |
| Sedang | Audit query N+1 dengan Laravel Debugbar/Telescope | Response time |
| Sedang | Tambah database connection pooling (PgBouncer) | Concurrency |
| Rendah | Profiling PHP dengan Blackfire/Tideways | Identifikasi hotspot |

---

## Kesimpulan

**Golang unggul di semua dimensi pengujian** — throughput, latency, reliabilitas, dan efisiensi resource:

| Dimensi | Pemenang | Margin |
|---|---|---|
| Throughput | **Golang** | 14–18x lebih tinggi |
| P95 Latency | **Golang** | 16–67x lebih cepat |
| Reliabilitas | **Golang** | 0% vs 1–7.2% failure |
| Memory Usage | **Golang** | 8x lebih hemat (95MB vs 763MB) |
| CPU Usage | **Golang** | 10.7x lebih efisien (0.056 vs 0.600 cores) |
| Response Size | **Golang** | 4.4x lebih kecil (21KB vs 92KB) |

Lumen menunjukkan degradasi serius mulai dari **10 concurrent users** — P90 sudah mencapai 32 detik, jauh di luar batas acceptable untuk production API. Untuk traffic production, Golang adalah pilihan yang lebih aman, efisien, dan scalable.

---

*Report generated: 2026-06-04 | Tool: ApacheBench 2.3 + Grafana (Kubernetes) | Runtime: TLSv1.3*
