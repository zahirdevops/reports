# Ingress python-UA traffic report — 2026-06-06

**Time window:** 09:00 – 16:00 WIB (UTC+7) — equivalent to 02:00 – 09:00 UTC, or 10:00 – 17:00 +0800 in raw log timestamps.
**Source:** internal Loki `monitoring/loki:3100`, label `{app="ingress-nginx"}` (both controller replicas merged, dedup'd).
**Filter:** lines whose access-log user-agent matches `python` (case-insensitive substring inside the UA quoted field).

## TL;DR

- **155 requests** total dengan UA `python-*` masuk ke ingress dalam window 7 jam.
- **Patch yang dipasang** di `prod-zahir-id/ingress-zid` (annotation `configuration-snippet`) hanya match versi **literal `python-requests/2.34.2`**.
- Patch **berhasil mem-block 6 request** ke `account.zahir.id /oauth/access_token` (status 429).
- Patch **gagal mem-block 149 request lain** karena 2 sumber utama melakukan rotasi versi ke `2.32.5` dan `2.33.1`.
- Endpoint sensitif yang ter-bypass: `/api/v2/zsql`, `/api/v3/products/zahir_ai/databases/<uuid>/zsql`, dan `/api/v1/python/exec` (Zahir AI code execution).
- Patch rekomendasi: ubah regex jadi `~* "^python-"` (atau `^python-(requests|httpx|urllib)`) untuk meng-cover seluruh keluarga klien Python, bukan version-pin.

## Patch saat ini (sumber kebocoran)

Lokasi: Ingress `ingress-zid` di namespace `prod-zahir-id`, annotation `nginx.ingress.kubernetes.io/configuration-snippet`:

```nginx
if ($http_user_agent ~* "python-requests/2\.34\.2") {
    return 429 "deprecated client - please upgrade";
}
```

Snippet ini ter-render 5× di `/etc/nginx/nginx.conf` di dalam pod ingress controller (1× per server block yang match). Regex match **versi tepat 2.34.2 saja** — `pip install requests==2.33.1` atau `==2.32.5` cukup untuk lolos.

## Breakdown 1 — per UA

| User-Agent | Hit | % |
|---|---:|---:|
| `python-requests/2.32.5` | 109 | 70.3% |
| `python-requests/2.33.1` | 38 | 24.5% |
| `python-requests/2.34.2` | 7 | 4.5% |
| `python-httpx/0.28.1` | 1 | 0.6% |

Hanya 7 dari 155 request yang ter-match patch — efektifitas patch ~4.5%.

## Breakdown 2 — per domain

| Domain | Hit | Endpoint dominan | Status dominan |
|---|---:|---|---|
| `go.zahironline.com` | 109 | `POST /api/v2/zsql` (105) + `POST /api/v3/zahir.ai/databases/<uuid>/zsql` (4) | 201 |
| `go.zahir.ai` | 30 | `POST /api/v3/products/zahir_ai/databases/<uuid>/zsql` | 26× 201, 4× 502 |
| `account.zahir.id` | 8 | `POST /oauth/access_token` (6) + `GET /login` (1) + `GET /` (1) | 429 (di-block), 200, 302 |
| `code.zahironline.com` | 4 | `POST /api/v1/python/exec` | 200 |
| `member.zahironline.com` | 1 | `GET /robots.txt` | 200 |
| `partner.zahironline.com` | 1 | `GET /wp-json/wp/v2/users` | 404 |

Catatan domain:
- `go.zahironline.com` jadi target terbesar (70% trafik) — semua ke `/api/v2/zsql?is_show_as_table=true`, sukses 201 dengan response body besar (12 KB – 91 KB). Database UUID konsisten = `2e0a86f4-c463-4a40-b037-72da585dfbf6`.
- `go.zahir.ai` dan `go.zahironline.com /api/v3/zahir.ai/databases/...` mengarah ke database UUID yang **sama** (`2e0a86f4-...`) → satu resource diakses lewat 2 hostname.
- `account.zahir.id /oauth/access_token` = upaya credential probing yang berhasil di-block (429).
- `partner.zahironline.com /wp-json/wp/v2/users` = WordPress user-enum scanner (404, scanner sembarangan).
- `code.zahironline.com /api/v1/python/exec` = fitur Python execution di Zahir AI; payload POST ≥ 200 KB.

## Breakdown 3 — UA × domain

| UA | Domain | Hit |
|---|---|---:|
| `python-requests/2.32.5` | `go.zahironline.com` | 105 |
| `python-requests/2.33.1` | `go.zahir.ai` | 30 |
| `python-requests/2.34.2` | `account.zahir.id` (blocked 429) | 6 |
| `python-requests/2.33.1` | `code.zahironline.com` | 4 |
| `python-requests/2.33.1` | `go.zahironline.com` | 4 |
| `python-requests/2.32.5` | `account.zahir.id` | 2 |
| `python-requests/2.32.5` | host tak terparse (`/` request) | 2 |
| `python-httpx/0.28.1` | `member.zahironline.com` | 1 |
| `python-requests/2.34.2` | `partner.zahironline.com` | 1 |

## Breakdown 4 — per source IP

| IP | Hit | UA | Catatan |
|---|---:|---|---|
| `103.126.86.133` | 105 | `python-requests/2.32.5` | Single-source flood ke `go.zahironline.com /api/v2/zsql`, semua 201. |
| `36.75.171.15` | 38 | `python-requests/2.33.1` | Multi-endpoint: `go.zahir.ai` + `code.zahironline.com /python/exec` + `go.zahironline.com /api/v3/zahir.ai/...`. |
| `172.70.142.121` | 2 | `python-requests/2.34.2` | Cloudflare edge IP (real client tersembunyi di `X-Forwarded-For`). |
| `172.70.93.110` | 2 | `python-requests/2.34.2` | Cloudflare edge IP. |
| `62.84.185.137` | 1 | `python-httpx/0.28.1` | `member.zahironline.com /robots.txt`. |
| `162.158.170.96` | 1 | `python-requests/2.34.2` | Cloudflare edge. |
| `162.158.88.90` | 1 | `python-requests/2.34.2` | Cloudflare edge. |
| `172.71.124.45` | 1 | `python-requests/2.32.5` | Cloudflare edge. |
| `162.158.189.225` | 1 | `python-requests/2.32.5` | Cloudflare edge. |
| `178.16.53.8` | 1 | `python-requests/2.34.2` | residential probe (`partner.zahironline.com /wp-json/wp/v2/users`). |
| `35.205.15.227` | 1 | `python-requests/2.32.5` | Google Cloud (GCE Belgium). |
| `34.77.166.77` | 1 | `python-requests/2.32.5` | Google Cloud (GCE Belgium). |

Catatan IP:
- 2 IP residential Indonesia (`103.126.86.133` & `36.75.171.15`) bertanggung jawab atas **143 / 155 = 92.3%** trafik.
- IP yang muncul dari range `172.70.x` / `172.71.x` / `162.158.x` adalah Cloudflare edge — real client tertutup; perlu cek `X-Forwarded-For` di upstream untuk identifikasi.
- 2 IP GCP (`35.205.15.227`, `34.77.166.77`) memakai `python-requests/2.32.5` yang sama dengan attacker utama — bisa jadi sumber yang sama men-tunnel via GCE/VPS, perlu cross-check timing.

## Breakdown 5 — path × host × status (top 12)

| Hit | Status | Domain | Path |
|---:|---|---|---|
| 105 | 201 | `go.zahironline.com` | `/api/v2/zsql` |
| 26 | 201 | `go.zahir.ai` | `/api/v3/products/zahir_ai/databases/2e0a86f4-.../zsql` |
| **6** | **429** | `account.zahir.id` | `/oauth/access_token` ← **patch effective** |
| 4 | 200 | `code.zahironline.com` | `/api/v1/python/exec` |
| 4 | 200 | `go.zahironline.com` | `/api/v3/zahir.ai/databases/2e0a86f4-.../zsql` |
| 4 | 502 | `go.zahir.ai` | `/api/v3/products/zahir_ai/databases/2e0a86f4-.../zsql` |
| 2 | 400 | (unparseable host, request to `/`) | `/` |
| 1 | 200 | `member.zahironline.com` | `/robots.txt` |
| 1 | 200 | `account.zahir.id` | `/login` |
| 1 | 302 | `account.zahir.id` | `/` |
| 1 | 404 | `partner.zahironline.com` | `/wp-json/wp/v2/users` |

## Rekomendasi tindak lanjut

### Patch ingress (prioritas tinggi)

Ubah annotation `nginx.ingress.kubernetes.io/configuration-snippet` di `prod-zahir-id/ingress-zid`:

```nginx
# sekarang (version-pinned, mudah bypass)
if ($http_user_agent ~* "python-requests/2\.34\.2") {
    return 429 "deprecated client - please upgrade";
}

# usulan (cover seluruh keluarga klien Python)
if ($http_user_agent ~* "^python-(requests|httpx|urllib|aiohttp)") {
    return 429 "deprecated client - please upgrade";
}
```

Pertimbangkan juga snippet di Ingress untuk `prod-zahir-ai-*` dan `prod-zo` namespace karena yang paling banyak ter-hit justru di sana, bukan di `prod-zahir-id`.

### Block IP

`103.126.86.133` dan `36.75.171.15` sudah jelas pola serangan (single-IP, single-UA-family, target endpoint sensitif). Block langsung di:
- ingress-nginx config (`nginx.ingress.kubernetes.io/server-snippet` deny block), atau
- Aliyun SLB security group, atau
- Cloudflare WAF (untuk domain yang fronted by CF).

### Endpoint hardening

UA filter mudah dipalsukan; attacker tinggal ganti `User-Agent` jadi `Mozilla/...`. Yang lebih ampuh:

- `/api/v1/python/exec` di `code.zahironline.com` → wajib auth + IP allowlist (siapa yang sah memakai fitur ini?).
- `/api/v2/zsql` & `/api/v3/.../zsql` → audit siapa yang punya akses ke database UUID `2e0a86f4-c463-4a40-b037-72da585dfbf6`. Apakah credential yang dipakai attacker valid? Kalau ya — rotasi.
- `account.zahir.id /oauth/access_token` → rate-limit lebih ketat untuk klien non-browser.

### Investigasi credential

105 request `python-requests/2.32.5` ke `/api/v2/zsql` semuanya return **201** — artinya attacker punya credential valid. Cek log audit aplikasi (Loki `{app="zo-api-v3"}`) untuk user/token yang dipakai dari IP `103.126.86.133`.

## Metodologi & batas

- Sumber data: ingress controller stdout → Loki internal cluster, label `{app="ingress-nginx"}`.
- LogQL filter: `|~ "\"python"` (cocokkan kuotasi pembuka UA dengan substring `python`).
- Window: 7 chunk 1 jam (UTC 02:00 – 09:00) untuk menghindari Loki `limit=5000` per query_range.
- Dedup: `(timestamp_ns, first_50_chars_of_log)` — menghilangkan duplikasi karena Loki bisa kembalikan baris yang sama dari 2 stream (per replica).
- **Yang tidak dicakup:**
  - Permintaan yang dibatalkan klien sebelum di-log oleh nginx (jarang, tapi mungkin).
  - Permintaan yang ter-route ke ingress controller selain `nginx-ingress`, kalau ada (cek `kubectl get ingressclass`).
  - X-Forwarded-For header tidak ada di log format saat ini; IP Cloudflare di-laporkan sebagai source langsung, tidak ditrace ke real client. Untuk forensik lebih dalam, tambahkan `$http_x_forwarded_for` ke `log-format-upstream` di ConfigMap `kube-system/nginx-configuration`.
