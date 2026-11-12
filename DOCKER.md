# Docker Compose — Cheatsheet

Panduan lengkap ada di [`README.md`](README.md). File ini ringkasan perintah
operasional.

## Service

| Service      | Image            | Fungsi                    |
|--------------|------------------|---------------------------|
| `web`        | PHP 8.2 + Apache | Aplikasi stok barang (`/`) |
| `db`         | MariaDB 11.4     | Database `stokbarang`     |
| `phpmyadmin` | phpMyAdmin 5     | Opsional (profil `tools`) |

## Pertama kali

```bash
cp .env.example .env      # lalu ganti password
docker compose up -d --build
docker compose ps
```

Akses `http://localhost:8080`.

## Perintah harian

```bash
docker compose logs -f web         # log aplikasi
docker compose logs -f db          # log database
docker compose restart web         # restart setelah edit file PHP
docker compose up -d               # terapkan perubahan compose
docker compose up -d --build       # rebuild image PHP
docker compose stop                # stop container
docker compose down                # hapus container (volume DB tetap)
docker compose down -v             # HAPUS container + data database
docker compose config              # validasi konfigurasi
```

Shell di container:

```bash
docker compose exec web bash
docker compose exec db mariadb -ustokbarang -p stokbarang
```

## Import database XAMPP

```bash
mysqldump -u root stokbarang > stokbarang.sql
cp stokbarang.sql docker/initdb/     # hanya saat volume DB masih kosong
docker compose up -d --build
```

Database yang sudah berjalan:

```bash
docker compose cp stokbarang.sql db:/tmp/stokbarang.sql
docker compose exec db sh -c \
  'mariadb -uroot -p"$MARIADB_ROOT_PASSWORD" stokbarang < /tmp/stokbarang.sql'
```

## Backup & restore

```bash
docker compose exec -T db sh -c \
  'mariadb-dump -uroot -p"$MARIADB_ROOT_PASSWORD" --databases "$MARIADB_DATABASE"' \
  > "backup-$(date +%F).sql"

docker compose cp backup-2026-01-01.sql db:/tmp/restore.sql
docker compose exec db sh -c \
  'mariadb -uroot -p"$MARIADB_ROOT_PASSWORD" < /tmp/restore.sql'
```

## phpMyAdmin

```bash
docker compose --profile tools up -d
docker compose --profile tools stop phpmyadmin
```

Jangan biarkan phpMyAdmin terbuka ke internet.
