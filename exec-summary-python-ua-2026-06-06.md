# Executive Summary — Trafik UA Python pada Endpoint Zahir

**Tanggal:** 2026-06-06  
**Window pengamatan:** 09:00 – 16:00 WIB (7 jam)  
**Tim:** Infrastructure

> **Catatan kerangka:** dokumen ini sengaja TIDAK menggunakan kata "pelaku" / "serangan" untuk lalu lintas yang belum diverifikasi. Trafik UA Python ke endpoint API resmi bisa jadi (a) perpanjangan serangan kemarin, (b) ekstraksi data legit oleh user/integrator, atau (c) campuran keduanya. Tanpa baseline serangan kemarin (IP/UA/endpoint), kategorisasi final perlu input dari tim/owner. Lihat bagian "Yang belum dapat dipastikan".

## TL;DR

- **155 request** memakai user-agent klien Python masuk ke endpoint Zahir dalam window 7 jam.
- **Patch ingress hari ini bekerja** untuk pola yang spesifik di-target: **6 dari 7 request** versi `2.34.2` ke `account.zahir.id /oauth/access_token` (probing OAuth) berhasil di-block 429. Ini pola yang paling mirip serangan kemarin.
- **149 request lain (96%)** memakai versi berbeda (`2.32.5`, `2.33.1`) DAN mengarah ke endpoint API resmi (zsql, python/exec) dengan kredensial valid (200/201). Apakah ini perpanjangan serangan, aktivitas legit, atau campuran, **belum dapat dipastikan**.
- Sumber dominan dari 149 itu sudah dipetakan ke 2 IP Indonesia. Satu IP terkonfirmasi memakai token user **wahyuandry94@gmail.com** dengan slug **PT ZAHIR** (company internal Zahir).

## Yang dapat dipastikan

| Fakta | Sumber |
|---|---|
| 155 request UA python-* dalam window 09:00–16:00 WIB | Loki `{app="ingress-nginx"}` |
| Patch ConfigMap nginx-ingress saat ini regex `python-requests/2\.34\.2` literal | ConfigMap `kube-system/nginx-configuration` (read langsung) |
| 6 request 2.34.2 ke `account.zahir.id /oauth/access_token` di-block 429 | Loki status code per IP+UA |
| 105 request 2.32.5 dari `103.126.86.133` ke `go.zahironline.com /api/v2/zsql` semua 201 | Loki ingress log |
| IP `103.126.86.133` memakai token milik `wahyuandry94@gmail.com`, client_name "API Zahir online" (Android), slug `ptzahirinternasional240709093431-pg.zahironline.com` ("PT ZAHIR") | Korelasi IP antara ingress log dan Go API v3 structured log; konfirmasi DB `companies` + `access_tokens` (read-only) |
| Token Android Wahyu di-issue 2026-06-06 11:52 WIB, di tengah window | DB `access_tokens.created_at` |
| 38 request 2.33.1 dari `36.75.171.15` ke `code.zahironline.com /api/v1/python/exec` + `go.zahir.ai /…/zsql`, mostly 200/201 | Loki ingress log |

## Yang BELUM dapat dipastikan

1. **Detail serangan kemarin (2026-06-05).** Saya tidak punya log baseline. Klaim "patch 2.34.2 dipasang karena kemarin pakai 2.34.2" hanya inferensi dari regex patch. Bisa jadi kemarin pakai versi lain dan patch tidak match, atau patch ditujukan ke pola berbeda.
2. **Apakah trafik hari ini terkait dengan kemarin.** 3 versi python-requests dari 3 IP/ASN berbeda lebih konsisten dengan 3 aktor terpisah ketimbang satu aktor yang downgrade. Tapi kemungkinan koordinasi tidak bisa ditolak juga tanpa data.
3. **Apakah `wahyuandry94@gmail.com` sedang melakukan kerja resmi.** Akun ini company internal Zahir, token Android sah, pola `/api/v2/zsql` adalah API resmi yang siapa pun dengan kredensial valid boleh pakai. Bisa jadi data engineer pull report. Bisa jadi credential bocor / insider misuse. **Verifikasi ke HR/manajer langsung diperlukan** sebelum menarik konklusi.
4. **Identitas pemilik token di IP `36.75.171.15`.** Belum di-cross-correlate via Go API log.
5. **Apakah endpoint `/api/v1/python/exec` di Zahir AI sedang digunakan oleh customer legit atau abuse.** Endpoint ini adalah produk feature; perlu owner Zahir AI konfirmasi siapa yang seharusnya pakai.

## Pelaku & lokasi IP (data mentah, BUKAN klasifikasi serangan)

| IP | Hit | Lokasi | ISP / ASN | Versi UA | Target dominan | Status request | Identitas (kalau ada) |
|---|---:|---|---|---|---|---|---|
| `103.126.86.133` | 105 | **Karanganyar, Jateng — Indonesia** | PT Rasi Bintang Perkasa (AS138109) | 2.32.5 | `go.zahironline.com /api/v2/zsql` | 201 sukses semua | wahyuandry94@gmail.com / PT ZAHIR (internal) |
| `36.75.171.15` | 38 | **Mataram, NTB — Indonesia** | PT Telkom / Indihome (AS7713) | 2.33.1 | `code.zahironline.com /api/v1/python/exec`, `go.zahir.ai /…/zsql` | 200/201 sukses | **Belum di-cross-correlate** |
| 6× Cloudflare edge SG | 6 | Singapore | Cloudflare (AS13335) | 2.34.2 | `account.zahir.id /oauth/access_token` | **429 di-block patch** | Real client tertutup CF |
| `178.16.53.8` | 1 | Amsterdam — Belanda | Omegatech VPS (AS202412) | 2.34.2 | `partner.zahironline.com /wp-json/wp/v2/users` | 404 | Scanner WordPress noise |
| `62.84.185.137` | 1 | Lauterbourg — Prancis | Contabo VPS (AS51167) | httpx 0.28.1 | `member.zahironline.com /robots.txt` | 200 | Crawler |
| `35.205.15.227`, `34.77.166.77` | 2 | Brussels — Belgia | Google Cloud (AS396982) | 2.32.5 | sama versi dengan IP 1; konteks lain | bervariasi | Perlu cross-check |
| (single-hit IP lain) | 2 | bervariasi | bervariasi | bervariasi | noise | bervariasi | Skip |

## Timeline (WIB, hanya hari ini)

- **~09:13** trafik python-* dari `103.126.86.133` mulai terlihat di endpoint v3 (custom_labels, check_token)
- **~10:07** trafik POST `/api/v2/zsql` dari `103.126.86.133` mulai padat (interval 3–10 detik, response 12–91 KB)
- **11:52** token Android baru di-issue untuk akun yang sama
- **~16:00** masih berlangsung saat window ditutup

## Yang sudah dilakukan

- ✅ Inventarisasi teknis lengkap (lihat `incident-2026-06-06-python-ua-ingress.md`)
- ✅ Identifikasi 1 dari 2 IP dominan via korelasi log aplikasi + DB production (read-only)
- ✅ Rencana hardening observability disusun & disimpan untuk eksekusi terjadwal

## Pertanyaan terbuka untuk pengambilan keputusan

Sebelum mengeskalasi atau melakukan tindakan irreversible (block IP, revoke token, tuduhan internal), tim manajemen perlu menjawab:

1. **Apakah `wahyuandry94@gmail.com` karyawan/kontraktor aktif?** Apakah dia berhak/diberi tugas untuk pull data via Python script ke endpoint `/api/v2/zsql`?
2. **Apa detail serangan kemarin (2026-06-05)?** Spesifik: IP source, UA, endpoint, timing. Tanpa ini kita tidak bisa konfirmasi apakah hari ini adalah perpanjangan atau insiden terpisah.
3. **Siapa owner endpoint `/api/v1/python/exec` di Zahir AI?** Mereka bisa konfirmasi apakah customer pemilik token di `36.75.171.15` adalah pengguna legit.

## Rekomendasi tindak lanjut (urut dari paling aman)

1. **Diskusi internal** (low risk): klarifikasi 3 pertanyaan terbuka di atas. Tidak ada perubahan sistem yang dilakukan sebelum ini selesai.
2. **Ketatkan patch ingress** (low risk, defensive): regex `^python-(requests|httpx|urllib|aiohttp)` di `ingress-zid` snippet — block seluruh keluarga python client untuk endpoint `/oauth/access_token`. Ini murni memperkuat patch yang sudah ada untuk pola yang terbukti probing, tidak menyentuh trafik zsql/python-exec yang masih ambigu.
3. **Cross-correlate IP `36.75.171.15`** dengan metode yang sama seperti IP pertama (Loki + DB lookup) — hasilnya bisa di-feed ke owner Zahir AI.
4. **Bila konklusi adalah misuse** (setelah no. 1 selesai): rotasi token Wahyu, force re-login, block IP via Cloudflare/SLB.
5. **Bila konklusi adalah usage legit**: dokumentasikan IP/akun di allowlist (mis. add header `X-Internal-Client: true` di klien resmi), supaya pengamatan masa depan tidak salah klasifikasi.

---

*File rinci pendamping:* `incident-2026-06-06-python-ua-ingress.md` (laporan teknis), `~/.claude/.../project_zahir_python_ua_attack.md` (memory insiden), `~/.claude/.../project_zahir_ingress_observability_plan.md` (rencana hardening).
