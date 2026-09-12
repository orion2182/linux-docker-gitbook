# 15. Docker Storage

> Container itu fana, data tidak boleh fana. Bab ini kapan pakai volume vs bind vs tmpfs + backup.

## Tujuan Pembelajaran

- Menjelaskan writable layer dan kenapa hilang saat rm
- Memakai named volume, bind mount, tmpfs
- Persistensi DB, backup, dan troubleshooting storage

## 1. Container Writable Layer

Container = image read-only + layer tulis tipis (copy-on-write). `docker rm` hapus layer itu.

```bash
$ docker run -d --name ephemeral nginx:alpine
$ docker exec ephemeral sh -c "echo halo > /tmp/data.txt && cat /tmp/data.txt"
$ docker rm -f ephemeral
$ docker run --name cek nginx:alpine cat /tmp/data.txt  # gagal, data hilang
```

Kesimpulan: data penting jangan di layer tulis. Pakai volume/bind.

## 2. Volumes

Dikelola Docker di `/var/lib/docker/volumes/`. Portable, bisa di-backup, driver-able. Default untuk DB prod.

```bash
$ docker volume create webdata
$ docker volume ls
$ docker run -d --name web -v webdata:/usr/share/nginx/html nginx:alpine
$ docker inspect -f '{{json .Mounts}}' web | python3 -m json.tool
```

## 3. Bind Mounts

Map path host langsung. Enak untuk dev (live reload), config ro, tapi terikat path host.

```bash
$ mkdir -p ~/site && echo "<h1>halo</h1>" > ~/site/index.html
$ docker run -d --name web-dev -p 8080:80 -v ~/site:/usr/share/nginx/html nginx:alpine
$ curl http://127.0.0.1:8080
$ docker run -d --name web-ro -v ~/site:/usr/share/nginx/html:ro nginx:alpine
```

Config prod: bind `:ro` agar container tidak ubah config host.

## 4. tmpfs

RAM-only, hilang saat stop. Untuk secret sementara / cache / `/tmp` sensitif.

```bash
$ docker run -d --name cache --tmpfs /app/cache:rw,noexec,nosuid,size=100m myapp:1
$ docker exec cache df -h | grep cache
```

Jangan untuk data yang harus survive restart.

## 5. Named Volumes

Pola rapi: 1 volume per keperluan (`pgdata`, `redisdata`, `uploads`).

```bash
$ docker volume create pgdata
$ docker run -d --name db -e POSTGRES_PASSWORD=rahasia -v pgdata:/var/lib/postgresql/data postgres:16-alpine
$ docker volume inspect pgdata
$ docker volume ls -f dangling=true
```

Gunakan nama volume yang jelas, bukan nama acak. Dokumentasikan volume tersebut di Compose (Bab 17).

## 6. Database Persistence

Contoh Postgres benar:

```bash
$ docker volume create pgdata
$ docker run -d --name pg \
  -e POSTGRES_USER=app -e POSTGRES_PASSWORD=kuat -e POSTGRES_DB=appdb \
  -v pgdata:/var/lib/postgresql/data \
  --restart unless-stopped \
  postgres:16-alpine
$ docker logs pg --tail 20
$ docker exec -it pg psql -U app -d appdb -c "create table t(id serial primary key);"
$ docker restart pg
$ docker exec -it pg psql -U app -d appdb -c "select * from t;"
```

Test wajib: restart + recreate, data tetap ada. Kalau hilang = mount salah.

## 7. Volume Backup

Backup = container sekali pakai yang mount volume + tar ke host.

```bash
$ docker run --rm -v pgdata:/data -v $(pwd):/backup alpine tar -czf /backup/pgdata-$(date +%F).tar.gz -C /data .
$ ls -lh pgdata-*.tar.gz
# Restore (volume harus kosong / baru):
$ docker volume create pgdata-restore
$ docker run --rm -v pgdata-restore:/data -v $(pwd):/backup alpine sh -c "tar -xzf /backup/pgdata-2026-09-11.tar.gz -C /data && ls -la /data"
```

Otomatiskan via cron/systemd (Bab 08) + offsite (Bab 24). Test restore, bukan cuma backup.

## 8. Storage Troubleshooting

```bash
$ docker system df
$ docker system df -v | head -n 60
$ docker volume ls
$ docker inspect -f '{{json .Mounts}}' <nama> | python3 -m json.tool
$ ls -l /var/lib/docker/volumes/ | head
$ df -h /var/lib/docker
$ docker logs <nama> | tail -n 50
```

Kasus umum:
- Permission denied → UID di container ≠ owner file host. Fix: `--chown`, `user: "1000:1000"`, atau init chown di entrypoint.
- Data hilang setelah recreate → pakai anonymous volume / path salah. Cek `inspect Mounts`.
- `/var/lib/docker` penuh → `docker system prune` hati-hati + batasi log + pindah root dir kalau perlu.
- DB tidak start → cek log: biasanya permission `/var/lib/postgresql/data` atau password berubah.

Jangan menjalankan `docker volume prune -f` di production tanpa membuat daftar volume penting.

## Fungsi Perintah

| Perintah | Fungsi |
| --- | --- |
| `docker volume create` | Membuat named volume yang dikelola Docker. |
| `docker volume ls` / `inspect` | Menampilkan dan memeriksa detail volume. |
| `docker run -v` | Memasang named volume atau bind mount ke dalam container. Tambahkan `:ro` untuk mode baca saja. |
| `--tmpfs` | Memasang filesystem sementara di RAM yang hilang saat container berhenti. |
| `docker system df` | Menampilkan penggunaan ruang Docker oleh image, container, volume, dan cache. |
| `docker volume prune` | Menghapus volume yang tidak digunakan; bersifat destruktif. |
| `docker inspect` | Memastikan sumber dan tujuan mount yang sebenarnya. |
| `tar` | Mengarsipkan atau memulihkan isi volume. `-c` membuat arsip, `-x` mengekstrak, dan `-z` menggunakan gzip. |
| `df` | Menampilkan ruang kosong filesystem host atau mount tertentu. |
| `ls` | Memeriksa file pada direktori volume atau backup. |
| `docker logs` | Membaca log container untuk menemukan error permission atau database. |
| `cron` / `systemd` | Menjadwalkan backup volume secara berkala. |
