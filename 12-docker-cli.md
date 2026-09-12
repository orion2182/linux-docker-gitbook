# 12. Docker CLI

> Bab ini berisi perintah yang paling sering digunakan. Kuasai 13 perintah berikut untuk menangani sebagian besar operasi Docker sehari-hari.

## Tujuan Pembelajaran

- create/run/start/stop/restart/rm, ps, exec, inspect, logs, cp, stats, events

Konvensi contoh: image `nginx:alpine`, nama `web`.

## 1. docker run

Buat + jalankan sekaligus. Flag wajib hafal: `-d`, `--name`, `-p`, `-v`, `-e`, `--rm`, `--restart`.

```bash
$ docker run -d --name web -p 8080:80 nginx:alpine
$ docker run --rm -it ubuntu:22.04 bash
$ docker run -d --name web2 -e NGINX_HOST=example.com --restart unless-stopped nginx:alpine
$ curl -I http://127.0.0.1:8080
```

`run` = `create + start`. Untuk sekali pakai/debug, tambah `--rm`.

## 2. docker create

Buat saja tanpa start (untuk disiapkan dulu).

```bash
$ docker create --name web-siap nginx:alpine
$ docker ps -a | grep web-siap
$ docker start web-siap
```

Jarang dipakai harian, berguna di skrip provisioning.

## 3. docker start

Nyalakan container yang sudah ada (stop sebelumnya).

```bash
$ docker start web
$ docker start -a web   # attach, lihat output langsung (Ctrl+C bisa stop)
```

## 4. docker stop

Berhenti sopan (SIGTERM → tunggu → SIGKILL). Default 10 detik.

```bash
$ docker stop web
$ docker stop -t 30 web   # kasih waktu 30 dtk untuk shutdown rapi (DB!)
```

Untuk DB jangan `kill` brutal kecuali macet.

## 5. docker restart

Stop + start. Untuk terapkan env/config baru yang sudah diupdate? Tidak — env butuh recreate. Restart hanya untuk proses ulang.

```bash
$ docker restart web
$ docker restart -t 5 web
```

## 6. docker rm

Hapus container (harus stop dulu kecuali `-f`).

```bash
$ docker stop web && docker rm web
$ docker rm -f web2
$ docker container prune -f   # HATI-HATI: hapus semua yang stop
```

`prune` di prod wajib konfirmasi + snapshot.

## 7. docker ps

Lihat status. Format biar enak dibaca:

```bash
$ docker ps
$ docker ps -a
$ docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}\t{{.Image}}"
$ docker ps -f "status=running" -f "name=web"
```

`ps` tanpa `-a` hanya yang jalan. Container hilang? Cek `ps -a` dulu.

## 8. docker exec

Masuk / jalankan perintah di container hidup tanpa restart.

```bash
$ docker exec -it web sh
# di dalam: cat /etc/nginx/conf.d/default.conf
$ docker exec web nginx -t
$ docker exec -u nginx web whoami
```

`exec` untuk debug, bukan untuk install manual permanen (tidak reproducible).

## 9. docker inspect

Fakta lengkap JSON: IP, mount, env, network, entrypoint.

```bash
$ docker inspect web | head -n 80
$ docker inspect -f '{{.NetworkSettings.IPAddress}}' web
$ docker inspect -f '{{json .Mounts}}' web | python3 -m json.tool
$ docker inspect -f '{{.Config.Env}}' web
```

Hafalkan `-f` Go template untuk ambil 1 field cepat.

## 10. docker logs

Baca stdout/stderr container.

```bash
$ docker logs web
$ docker logs -f --tail 100 web
$ docker logs --since 10m web 2>&1 | tail -n 50
```

Kalau kosong = app log ke file, bukan stdout (perbaiki di image, Bab 14/21).

## 11. docker cp

Copy file host ↔ container (darurat / ambil config).

```bash
$ docker cp web:/etc/nginx/nginx.conf ./nginx.conf
$ docker cp ./index.html web:/usr/share/nginx/html/
```

Untuk data permanen pakai volume (Bab 15), bukan `cp` berulang.

## 12. docker stats

Top untuk container: CPU, mem, net, I/O live.

```bash
$ docker stats --no-stream
$ docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
```

`--no-stream` untuk snapshot sekali (buat skrip/monitoring).

## 13. docker events

Lihat kejadian daemon realtime: create, start, die, oom, destroy.

```bash
$ docker events --since 10m
$ docker events -f 'container=web' &
$ docker restart web
```

Perintah ini berguna untuk mencari penyebab container berhenti bersama `docker logs` dan `docker inspect`.

## Fungsi Perintah dan Flag

| Perintah atau flag | Fungsi |
| --- | --- |
| `docker run` | Membuat dan menjalankan container. `-d` berjalan di background, `--name` memberi nama, `-p` mem-publish port, `-v` memasang volume, `-e` menetapkan environment variable, dan `--rm` menghapus container setelah berhenti. |
| `docker create` | Membuat container tanpa menjalankannya. |
| `docker start` / `stop` / `restart` | Menjalankan, menghentikan dengan sinyal graceful, atau menjalankan ulang container. |
| `docker rm` | Menghapus container. `-f` memaksa penghentian dan penghapusan. |
| `docker ps` | Menampilkan container aktif; `-a` menyertakan container yang berhenti. |
| `docker exec` | Menjalankan perintah tambahan di dalam container yang sedang berjalan. `-it` menyediakan terminal interaktif. |
| `docker inspect` | Menampilkan metadata JSON lengkap tentang container, image, network, atau volume. |
| `docker logs` | Membaca stdout dan stderr container. `-f` mengikuti log, sedangkan `--tail` membatasi jumlah baris. |
| `docker cp` | Menyalin file antara host dan container. |
| `docker stats` | Menampilkan penggunaan CPU, memory, jaringan, dan I/O per container. |
| `docker events` | Menampilkan event daemon secara realtime, misalnya `start`, `die`, dan `oom`. |
| `--format` | Mengubah bentuk output menggunakan template Go. |
| `--restart` | Menetapkan kebijakan restart container, misalnya `unless-stopped`. |
