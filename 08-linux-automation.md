# 08. Linux Automation

> Tugas admin yang dikerjakan 2x manual = harus diotomatisasi. Bab ini cron vs systemd timer + pola aman.

## Tujuan Pembelajaran

- Nulis cron/crontab yang benar (path, env, lock, log)
- Pakai systemd timer sebagai pengganti modern
- Pola maintenance, update, dan automation yang tidak menembak kaki sendiri

## 1. cron

Daemon jadwal klasik. Cek status dulu:

```bash
$ systemctl status cron
$ cat /etc/crontab
$ ls /etc/cron.{d,daily,hourly,weekly,monthly}/
```

Format: `menit jam dom bulan dow user perintah`.

## 2. crontab

```bash
$ crontab -l
$ crontab -e
# Contoh:
# PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
# SHELL=/bin/bash
# 0 2 * * * /opt/scripts/backup.sh >>/var/log/backup.log 2>&1
# */5 * * * * /opt/scripts/healthcheck.sh
```

Aturan: selalu set `PATH`, redirect log, jangan asumsi env interaktif. Test manual `/opt/scripts/backup.sh` sebelum pasang jadwal.

Contoh pola waktu:
- `0 2 * * *` tiap jam 2 pagi
- `*/10 * * * *` tiap 10 menit
- `0 0 * * 0` tiap Minggu tengah malam

## 3. systemd timers

Pengganti cron yang bisa dependency, jitter, persistent (kejar ketinggalan setelah mati).

```bash
# /etc/systemd/system/backup.service
[Unit]
Description=Backup harian

[Service]
Type=oneshot
ExecStart=/opt/scripts/backup.sh

# /etc/systemd/system/backup.timer
[Unit]
Description=Jalankan backup tiap jam 2

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
$ sudo systemctl daemon-reload
$ sudo systemctl enable --now backup.timer
$ systemctl list-timers --all | grep backup
$ journalctl -u backup.service --since "2 days ago"
```

Pilih timer kalau butuh log journald, retry, atau `After=docker.service`.

## 4. Scheduled Tasks

Contoh tugas umum:

- Backup config + DB (Bab 24)
- Bersih log/cache/tmp lama
- Healthcheck HTTP + restart kalau perlu
- Renew TLS (biasanya sudah otomatis via certbot timer)

```bash
$ systemctl list-timers --all
$ sudo journalctl -u certbot.timer --no-pager | tail
```

## 5. Maintenance Scripts

Satu skrip = idempotent (dijalankan 2x hasilnya sama), ada lock biar tidak tumpuk:

```bash
#!/usr/bin/env bash
set -euo pipefail
LOCK=/run/maintenance.lock
exec 9>"$LOCK"
flock -n 9 || { echo "sudah jalan, keluar"; exit 0; }
echo "[$(date -Is)] mulai maintenance"
sudo apt update
sudo journalctl --vacuum-time=14d
sudo find /tmp -type f -atime +7 -delete
echo "[$(date -Is)] selesai"
```

Taruh di `/opt/scripts/`, `chmod +x`, versioning Git.

## 6. Automated Updates

Untuk patch security otomatis (detail di Bab 05):

```bash
$ cat /etc/apt/apt.conf.d/50unattended-upgrades | grep -v "^//" | grep -v "^$"
$ cat /var/log/unattended-upgrades/unattended-upgrades.log | tail -n 20
```

Jangan auto-upgrade mayor (misal Postgres 14 → 16) tanpa snapshot + jadwal. Bedakan `security` vs `semua`.

## 7. Automation Patterns

Pola yang selamatkan kamu:

1. **Log + alert:** tiap job tulis ke log + exit code jelas. Job diam = job mati diam-diam.
2. **Lock + timeout:** `flock` + `timeout 300 skrip.sh` agar tidak gantung selamanya.
3. **Dry-run dulu:** tambah flag `--dry-run` untuk job destruktif (hapus, prune).
4. **Staging dulu:** jalankan di 1 server staging 1 minggu sebelum ke semua prod.
5. **Notifikasi gagal saja:** kirim alert hanya saat gagal biar tidak alert-fatigue.

Contoh dengan timeout di cron:

```bash
0 3 * * * /usr/bin/timeout 1800 /opt/scripts/backup.sh >>/var/log/backup.log 2>&1
```

## Latihan

1. Buat cron tiap 5 menit yang append `date` ke `/tmp/cron-test.log`. Buktikan jalan, hapus lagi.
2. Ubah job itu jadi systemd timer + service, cek `list-timers` dan `journalctl`.
3. Tambah `flock` ke 1 skrip lama kamu, simulasikan 2x jalan bersamaan.
4. Audit semua timer aktif: `systemctl list-timers`, matikan yang tidak dikenal (di lab).

## Rangkuman

- Cron simpel, systemd timer powerful + terobservasi.
- Selalu: PATH jelas, lock, log, timeout, idempotent.
- Otomatisasi tanpa monitoring = otomatisasi masalah.
