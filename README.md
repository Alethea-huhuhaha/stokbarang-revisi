# Stok Barang

Aplikasi manajemen stok barang berbasis **PHP + MySQL/MariaDB**. Bisa dijalankan
langsung dengan **XAMPP** atau lewat **Docker Compose**.

Fitur: login, dashboard, kategori, data barang, barang masuk, barang keluar,
dan laporan stok.

---

## Teknologi

- PHP **8.2** + Apache (`php:8.2-apache`)
- Ekstensi PHP: `mysqli`, `pdo_mysql`, `opcache`
- MariaDB **11.4**
- Docker Compose (Compose v2)
- phpMyAdmin 5 (opsional, hanya untuk maintenance)

---

## Struktur Project

```text
stokbarang/
├── docker-compose.yml       # Service web, db, phpMyAdmin
├── .env.example             # Template konfigurasi (copy ke .env)
├── docker/
│   ├── php/
│   │   ├── Dockerfile       # Image PHP + ekstensi
│   │   ├── php.ini          # Konfigurasi PHP
│   │   └── apache.conf      # Konfigurasi Apache + blokir file sensitif
│   └── initdb/
│       └── 01-schema.sql    # Schema awal (hanya saat volume DB kosong)
├── assets/  css/  js/       # Aset frontend
├── partials/                # Header, sidebar, topbar, footer
├── fuction.php              # Koneksi database + helper
├── index.php                # Dashboard
├── login.php / logout.php   # Autentikasi
├── kategori.php             # CRUD kategori
├── barang.php               # CRUD barang
├── masuk.php / keluar.php   # Transaksi stok
├── laporan.php              # Laporan
└── database_update.sql      # Migration schema (manual)
```

DocumentRoot container adalah **root folder project ini**, jadi aplikasi diakses
langsung dari `/` (bukan `/stokbarang/`).

---

## Menjalankan dengan Docker

### 1. Siapkan environment

```bash
cp .env.example .env
```

Ubah minimal password database di `.env`:

```env
APP_PORT=8080
DB_ROOT_PASSWORD=password-root-yang-kuat
DB_PASSWORD=password-app-yang-kuat
```

### 2. Jalankan

```bash
docker compose up -d --build
```

Cek status & log:

```bash
docker compose ps
docker compose logs -f web
```

Buka `http://localhost:8080`.

Login awal (hanya jika database dibuat dari schema bawaan):

```text
Email    : admin@example.com
Password : admin123
```

Segera ganti akun tersebut setelah login pertama.

### 3. phpMyAdmin (opsional)

```bash
docker compose --profile tools up -d
```

Buka `http://localhost:8081`, login memakai user database dari `.env`.
Matikan kembali setelah selesai:

```bash
docker compose --profile tools stop phpmyadmin
```

---

## Deploy ke VPS

```bash
sudo mkdir -p /var/www/stokbarang
sudo chown "$USER":"$USER" /var/www/stokbarang

git clone https://github.com/Alethea-huhuhaha/stokbarang-revisi.git /var/www/stokbarang
cd /var/www/stokbarang

cp .env.example .env
nano .env                 # ganti password database
docker compose up -d --build
```

Akses `http://IP-SERVER:8080`.

Pastikan port `APP_PORT` sudah dibuka di firewall. Jangan lupa memasang reverse
proxy + HTTPS (Nginx/Caddy/Traefik) jika dipakai publik.

---

## Konfigurasi Environment

Semua konfigurasi dibaca dari `.env` di root project.

| Variabel | Default | Keterangan |
|---|---|---|
| `APP_PORT` | `8080` | Port website di host |
| `PMA_PORT` | `8081` | Port phpMyAdmin di host |
| `TZ` | `Asia/Jakarta` | Zona waktu container |
| `DB_ROOT_PASSWORD` | – | Password root MariaDB |
| `DB_DATABASE` | `stokbarang` | Nama database aplikasi |
| `DB_USERNAME` | `stokbarang` | User database aplikasi |
| `DB_PASSWORD` | – | Password user database aplikasi |

`fuction.php` membaca `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, dan `DB_PASS`.
Jika variabel tidak ada (misalnya saat dijalankan lewat XAMPP), nilainya jatuh
ke default `localhost` / `root` / tanpa password / database `stokbarang`.

> **Penting:** `DB_USERNAME` dan `DB_PASSWORD` hanya dipakai saat volume database
> pertama kali dibuat. Jika diubah setelah database ada, ubah juga user di dalam
> database atau buat ulang volume (data akan hilang).

---

## Import Database yang Sudah Ada (dari XAMPP)

Schema di `docker/initdb/` hanya dijalankan saat volume database **masih kosong**.

**Cara A — sebelum `docker compose up` pertama:**

```bash
# export dari XAMPP
mysqldump -u root stokbarang > stokbarang.sql

# taruh di folder initdb, lalu jalankan
cp stokbarang.sql docker/initdb/
docker compose up -d --build
```

Setelah import berhasil, hapus dump dari `docker/initdb/` agar tidak ikut
ter-commit (sudah di-ignore oleh `.gitignore`).

**Cara B — container sudah berjalan:**

```bash
docker compose cp stokbarang.sql db:/tmp/stokbarang.sql
docker compose exec db sh -c \
  'mariadb -uroot -p"$MARIADB_ROOT_PASSWORD" stokbarang < /tmp/stokbarang.sql'
```

Jika dump berisi `CREATE DATABASE` sendiri, hilangkan nama database di akhir
perintah.

**Update schema** (misalnya setelah menarik kode baru):

```bash
docker compose cp database_update.sql db:/tmp/update.sql
docker compose exec db sh -c \
  'mariadb -uroot -p"$MARIADB_ROOT_PASSWORD" stokbarang < /tmp/update.sql'
```

---

## Update Project

```bash
git pull origin main
docker compose restart web        # source di-mount, cukup restart
```

| Perubahan | Perintah |
|---|---|
| File PHP/CSS/JS/HTML | `docker compose restart web` |
| `docker-compose.yml` berubah | `docker compose up -d` |
| `docker/php/` (Dockerfile, php.ini, apache.conf) berubah | `docker compose up -d --build` |
| Image MariaDB/phpMyAdmin ingin diperbarui | `docker compose pull db phpmyadmin && docker compose up -d` |

---

## Backup & Restore

Backup database:

```bash
docker compose exec -T db sh -c \
  'mariadb-dump -uroot -p"$MARIADB_ROOT_PASSWORD" --databases "$MARIADB_DATABASE"' \
  > "backup-$(date +%F).sql"
```

Restore:

```bash
docker compose cp backup-2026-01-01.sql db:/tmp/restore.sql
docker compose exec db sh -c \
  'mariadb -uroot -p"$MARIADB_ROOT_PASSWORD" < /tmp/restore.sql'
```

Jangan jalankan `docker compose down -v` kecuali memang ingin menghapus data
database (volume `db_data`).

---

## Troubleshooting

**Port sudah dipakai**

```bash
sudo ss -tlnp | grep :8080
```

Ubah `APP_PORT` di `.env`, lalu `docker compose up -d`.

**Database gagal connect**

```bash
docker compose ps
docker compose logs db
docker compose exec web php -r \
  '$c = mysqli_connect("db", getenv("DB_USER"), getenv("DB_PASS"), getenv("DB_NAME")); echo $c ? "OK\n" : "GAGAL\n";'
```

**Error 500 / halaman kosong**

```bash
docker compose logs -f web
```

`display_errors` dimatikan di production. Untuk debugging sementara, ubah
`display_errors = On` di `docker/php/php.ini`, lalu
`docker compose restart web`.

**403 / permission**

```bash
sudo chmod -R a+rX /var/www/stokbarang
```

Jangan memakai permission `777`.

---

## Development dengan XAMPP

Project ini tetap bisa dijalankan tanpa Docker:

1. Taruh folder ini di `C:\xampp\htdocs\stokbarang`.
2. Jalankan Apache + MySQL dari XAMPP.
3. Buat database `stokbarang` (import dump atau `docker/initdb/01-schema.sql`).
4. Buka `http://localhost/stokbarang/`.

`fuction.php` otomatis memakai koneksi default XAMPP (`localhost`, user `root`,
tanpa password) jika environment variable Docker tidak tersedia.

---

## Keamanan

- Ganti semua password default di `.env` dan akun login aplikasi.
- Port database tidak di-publish secara default.
- `.env`, folder `docker/`, `.git`, file `*.sql`, `*.md`, `*.yml` diblokir
  Apache sehingga tidak bisa diunduh dari browser.
- Gunakan HTTPS (reverse proxy) di depan port `8080`.
- Jangan biarkan phpMyAdmin aktif dalam jangka panjang.
- Backup database secara rutin.
- **Catatan teknis:** `login.php` masih menyimpan password sebagai plain-text
  dan menyusun query login dengan string interpolation. Sebelum dipakai publik,
  migrasikan ke `password_hash()`/`password_verify()` dan prepared statement.

Dokumentasi operasional singkat ada di [`DOCKER.md`](DOCKER.md).
