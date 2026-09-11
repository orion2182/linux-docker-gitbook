# 24. Backup & Disaster Recovery

> Backup yang belum pernah direstore = bukan backup. Bab ini 3-2-1 versi VPS + Docker.

## Tujuan Pembelajaran

- Snapshot, volume, DB, config, offsite, restore, verifikasi

Prinsip 3-2-1: 3 salinan, 2 media beda, 1 offsite. RPO (data hilang maks) + RTO (lama pulih) tentukan frekuensi.

## 1. VPS Snapshot

Snapshot provider = foto disk instan. Cepat untuk rollback sebelum upgrade besar.

```bash
# via panel/CLI provider (contoh generik):
# - buat snapshot "pre-upgrade-2026-09-11"
# - verifikasi status ready
# - baru apt upgrade / docker update
$ uptime; df -h; docker compose ps  # catat state sebelum snapshot
```

Snapshot ≠ backup offsite. Jangan andalkan 1 provider saja untuk data kritis. Test boot dari snapshot di staging.

## 2. Docker Volumes

Backup per volume via container sekali pakai (pola Bab 15):

```bash
$ docker volume ls
$ docker run --rm -v pgdata:/data -v $(pwd):/backup alpine tar -czf /backup/pgdata-$(date +%F).tar.gz -C /data .
$ ls -lh *.tar.gz
$ docker run --rm -v uploads:/data -v $(pwd):/backup alpine tar -czf /backup/uploads-$(date +%F).tar.gz -C /data .
```

Otomatiskan 1 skrip untuk semua volume penting + retensi (hapus >14 hari). Simpan daftar volume di `README.md`.

## 3. Database Backup

Dump logis (portable) + backup volume (fisik). Keduanya, bukan salah satu.

Postgres:

```bash
$ docker compose exec db pg_dump -U app appdb | gzip > db-$(date +%F).sql.gz
$ ls -lh db-*.sql.gz
$ zcat db-2026-09-11.sql.gz | head -n 20
# restore ke DB baru:
$ zcat db-2026-09-11.sql.gz | docker compose exec -T db psql -U app -d appdb_restore
$ docker compose exec db psql -U app -d appdb_restore -c "\dt"
```

MySQL:

```bash
$ docker compose exec db sh -c 'mysqldump -u root -p"$MYSQL_ROOT_PASSWORD" --all-databases' | gzip > mysql-$(date +%F).sql.gz
```

Jadwal: harian untuk kecil, per jam + WAL untuk kritis. Enkripsi sebelum offsite (`age`/`gpg`).

## 4. Configuration Backup

Config + compose + env template + secret terenkripsi (bukan secret plaintext ke Git publik!).

```bash
$ tar -czf config-$(date +%F).tar.gz -C /opt myapp --exclude='*.log'
$ cp /etc/nginx/sites-enabled/app.conf ./nginx-backup/
$ git -C /opt/myapp status; git -C /opt/myapp log --oneline -n 5
$ ls -lh config-*.tar.gz
```

Repo Git privat untuk config + skrip. Secret hanya sebagai file terenkripsi (`sops`/`age`) atau vault.

## 5. Offsite Backup

Salin keluar VPS: S3-compatible / rsync ke NAS / provider kedua.

```bash
$ rclone copy ./backups/ remote:bucket/myapp/ --progress
$ aws s3 sync ./backups/ s3://bucket-myapp/backups/ --delete
$ rsync -avz ./backups/ user@backup-server:/srv/backups/myapp/
$ rclone ls remote:bucket/myapp/ | tail
```

Aturan: enkripsi dulu, versioning di bucket, lifecycle hapus >90 hari, test download. Jangan 1 kunci untuk semua.

Contoh skrip harian (`/opt/scripts/backup.sh` + cron/systemd Bab 08):

```bash
#!/usr/bin/env bash
set -euo pipefail
D=/backup/$(date +%F); mkdir -p "$D"
docker run --rm -v pgdata:/data -v "$D":/b alpine tar -czf /b/pgdata.tar.gz -C /data .
docker compose -f /opt/myapp/compose.yaml exec -T db pg_dump -U app appdb | gzip > "$D/db.sql.gz"
tar -czf "$D/config.tar.gz" -C /opt myapp --exclude='*.log'
rclone copy "$D" remote:bucket/myapp/$(date +%F)/
find /backup -maxdepth 1 -type d -mtime +14 -exec rm -rf {} +
```

## 6. Restore

Urut: infra kosong → install Docker → restore config → restore volume/DB → up → verifikasi.

```bash
$ mkdir -p /opt && tar -xzf config-2026-09-11.tar.gz -C /opt
$ docker volume create pgdata
$ docker run --rm -v pgdata:/data -v $(pwd):/b alpine sh -c "tar -xzf /b/pgdata.tar.gz -C /data && ls -la /data"
$ zcat db-2026-09-11.sql.gz | docker compose -f /opt/myapp/compose.yaml exec -T db psql -U app -d appdb
$ docker compose -f /opt/myapp/compose.yaml up -d
$ docker compose ps; curl -f http://127.0.0.1/health
```

Dokumentasikan 1 halaman runbook + waktu tiap langkah (RTO nyata).

## 7. Backup Verification

Tanpa verifikasi otomatis, backup basi tidak ketahuan.

```bash
$ ls -lh /backup/*/ | tail -n 20
$ gzip -t db-*.sql.gz && echo "gzip ok"
$ tar -tzf pgdata-*.tar.gz | head
$ zcat db-*.sql.gz | head -n 5 | grep -i "PostgreSQL dump"
# restore drill bulanan ke staging:
$ ./scripts/restore-drill.sh staging && curl -f https://staging.contoh.com/health
```

Alert kalau: ukuran 0 / tidak ada file <25 jam / `gzip -t` gagal / drill gagal. Catat hasil drill di log.

## Latihan

1. Tentukan RPO/RTO untuk 1 app dummy (misal RPO 24 jam, RTO 2 jam).
2. Backup volume + DB + config, kirim offsite (rclone ke 1 remote).
3. Hapus volume di lab, restore dari backup, verifikasi data kembali.
4. Jadwalkan drill bulanan di kalender + skrip verifikasi otomatis.

## Rangkuman

- Snapshot untuk cepat, dump+volume untuk portable, offsite untuk selamat.
- Backup = 50%, restore teruji = 100%.
- Otomatis + alert + drill = tidur nyenyak.
