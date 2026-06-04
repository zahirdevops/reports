# api_v2_go

Port bertahap (strangler-fig) dari `api_v2` (Lumen 5.4 / PHP 5.6) ke **Go**.

Fondasi ini menyediakan kerangka HTTP server multi-tenant yang meniru lifecycle
request `api_v2`, sebagai pola untuk memindahkan domain (Sales, Purchases, Journals, dst.)
satu per satu.

## Stack

- Router: [chi](https://github.com/go-chi/chi)
- ORM: [GORM](https://gorm.io) (driver mysql & pgsql/postgres)
- Config: [viper](https://github.com/spf13/viper) (baca `.env`)
- Auth: [golang-jwt](https://github.com/golang-jwt/jwt) (HS256)
- Cache: [go-redis](https://github.com/redis/go-redis) *(disiapkan, belum dipakai)*

## Struktur

```
cmd/api/main.go            # entrypoint + graceful shutdown
internal/
  config/                  # loader .env (key kompatibel api_v2)
  database/                # Manager: koneksi central + per-company (pengganti ZahirORM)
  httpx/                   # response & error standar (port APIHelper)
  middleware/              # cors, locale, auth(JWT), acl, tenant, transaction, logger
  server/                  # router + registrasi route bergaya metadata
  modules/ping/            # endpoint contoh /api/v2/ping
```

## Pemetaan dari api_v2

| api_v2 (PHP) | api_v2_go |
|---|---|
| `Connection.php` (god-middleware) | rantai `internal/middleware/*` |
| `DBHelper::setDBCompany()` | `database.Manager.CompanyFor(slug)` |
| metadata route di `routes/web.php` | `middleware.RouteMeta` per-route |
| `PermissionException::checkPermission` | `middleware.ACL` |
| commit/rollback company route | `middleware.Transaction` |

## Menjalankan (via Docker — tanpa Go di host)

```bash
cp .env.example .env          # atau salin dari ../api_v2/.env

# Build + run container (default port host 8099 -> 8080 di container)
make run

# atau via compose (background)
make up
make logs
```

Pakai Podman? Override saja:

```bash
make build DOCKER=podman
```

Uji:

```bash
curl -s http://localhost:8099/api/v2/ping
# {"pong":true,"service":"api_v2_go","version":"0.1.0"}
```

> Catatan: build memakai `--network=host` (lihat `BUILD_FLAGS` di Makefile)
> agar `go mod download` & `apk add` bisa mengakses jaringan pada
> Podman rootless. Binary dikompilasi di dalam container Linux
> (`CGO_ENABLED=0`), sehingga tidak ada isu linker macOS (LC_UUID).

> Koneksi DB bersifat opsional saat boot. Bila DB central tidak tersedia,
> server tetap jalan untuk route global (`/api/v2/ping`); route bertipe `company`
> baru aktif saat koneksi central berhasil.

## Belum di-port (catatan untuk tahap berikut)

- Driver Firebird / SQL Server / MongoDB dari ZahirORM
- Dekripsi kredensial DB company (`Convert::encryptDecrypt`)
- Cookie auth `zor` (terenkripsi)
- Integrasi: Zahir ID (OAuth), payment, courier, notification, marketplace, S3, Telegram/WhatsApp
- Report PDF (dompdf), queue worker
# api_v2_go
