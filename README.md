# Linux & Docker — Dari VPS Kosong Sampai Production

> Panduan praktis administrasi Linux (Ubuntu), Docker, dan operasional VPS untuk pemula sampai siap production.

Buku ini disusun berurutan. Jangan loncat-loncat kalau baru mulai:

1. **Fondasi Linux (Bab 01–09):** package management, storage, networking, security, logs, bash, automation, git.
2. **Core Docker (Bab 10–18):** konsep container, instalasi, CLI, image, Dockerfile, storage, networking, Compose, multi-container.
3. **Production (Bab 19–26):** security, rootless, deploy di VPS, Nginx reverse proxy + TLS, registry, backup, monitoring, troubleshooting.

## Cara Pakai Buku Ini

- Tiap bab punya **Tujuan Pembelajaran**, materi per sub-bab, **contoh perintah**, **latihan**, dan **rangkuman**.
- Semua contoh diuji di **Ubuntu 22.04 / 24.04 LTS** (x86_64) di VPS.
- Blok perintah dengan `$` = user biasa, `#` = root / sudo.
- Kalau stuck, langsung lompat ke **Bab 26. VPS & Docker Troubleshooting**.

## Prasyarat

- VPS / VM Ubuntu 22.04+ (1 vCPU, 1–2 GB RAM cukup untuk mulai)
- Akses SSH + user dengan `sudo`
- Domain (opsional, baru wajib di Bab 21–22)
- Docker 24+ (cara install di Bab 11)

## Konvensi

```bash
$ whoami        # user biasa
$ sudo whoami   # root via sudo
$ docker ps     # perintah docker
```

> **Catatan:** Jangan copy-paste buta perintah `rm`, `mkfs`, `iptables -F`, `docker prune` di server production. Pahami dulu di tiap bab.

## Struktur

Lihat `SUMMARY.md` untuk daftar isi lengkap (26 bab).
