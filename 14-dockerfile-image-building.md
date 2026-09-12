# 14. Dockerfile & Image Building

> Dockerfile = resep. Bab ini bedah tiap instruksi + cara debug build yang gagal.

## Tujuan Pembelajaran

- Memakai FROM, WORKDIR, COPY/ADD, RUN, ENV/ARG, USER, EXPOSE, CMD/ENTRYPOINT, HEALTHCHECK
- Build + debug layer per layer

Contoh base yang dipakai di bab ini: `node:20-alpine`.

## 1. FROM

Image dasar. Pilih kecil + LTS + arch sesuai.

```dockerfile
FROM node:20-alpine
FROM python:3.12-slim
FROM golang:1.22-alpine AS build
```

Cek arch: `docker pull --platform linux/amd64 ...` kalau VPS ARM vs laptop beda.

## 2. WORKDIR

Folder kerja + auto-create. Selalu pakai absolut.

```dockerfile
WORKDIR /app
```

Hindari `RUN cd /app && ...` berulang — rapuh.

## 3. COPY

Salin dari konteks build. Bentuk exec (JSON) lebih aman untuk spasi.

```dockerfile
COPY package*.json ./
COPY --chown=node:node . .
```

`--chown` penting kalau nanti pakai USER non-root (Bab 19).

## 4. ADD

Bisa ekstrak tar + fetch URL, tapi jarang dibutuhkan. Prefer `COPY` kecuali butuh auto-extract.

```dockerfile
# ADD app.tar.gz /app/   # auto-extract
COPY app.tar.gz /app/
```

Aturan: default `COPY`, `ADD` hanya kalau sadar kenapa.

## 5. RUN

Eksekusi saat build. Gabungkan + bersihkan dalam 1 layer.

```dockerfile
RUN apk add --no-cache curl tzdata \
 && npm ci --only=production \
 && rm -rf /tmp/*
```

Beda `RUN` vs `CMD`: RUN saat build, CMD saat container start.

## 6. ENV

Default env di image.

```dockerfile
ENV NODE_ENV=production \
    PORT=3000
```

Override saat run: `docker run -e PORT=4000 ...`. Jangan taruh secret di ENV (kebaca `inspect/history`).

## 7. ARG

Variabel saat build saja (versi dependency, target).

```dockerfile
ARG NODE_VERSION=20
FROM node:${NODE_VERSION}-alpine
ARG BUILD_DATE
RUN echo "built $BUILD_DATE" > /build-info.txt
```

```bash
$ docker build --build-arg BUILD_DATE=$(date -u +%FT%TZ) -t myapp:1 .
```

ARG tidak persist ke runtime kecuali di-copy ke ENV eksplisit.

## 8. USER

Jangan jalan sebagai root kalau tidak perlu (wajib di prod).

```dockerfile
RUN addgroup -S app && adduser -S app -G app
USER app
```

```bash
$ docker exec myapp whoami
$ docker top myapp
```

Test: app masih bisa baca file + bind port >1024 sebagai non-root?

## 9. EXPOSE

Dokumentasi port (tidak otomatis publish).

```dockerfile
EXPOSE 3000
```

```bash
$ docker run -d -p 3000:3000 myapp:1
$ docker run -d -P myapp:1   # port host acak untuk semua port EXPOSE
$ docker ps
```

Publish tetap via `-p` / Compose `ports`.

## 10. CMD

Perintah default, bisa dioverride argumen `docker run`.

```dockerfile
CMD ["node", "server.js"]
```

## Fungsi Instruksi dan Perintah

| Instruksi atau perintah | Fungsi |
| --- | --- |
| `FROM` | Memilih image dasar atau stage build. |
| `WORKDIR` | Menetapkan direktori kerja untuk instruksi dan proses container. |
| `COPY` | Menyalin file dari build context ke image. Opsi `--chown` mengatur pemilik file. |
| `ADD` | Menyalin file dan memiliki kemampuan tambahan seperti ekstraksi arsip; gunakan hanya saat diperlukan. |
| `RUN` | Menjalankan perintah saat proses build dan menyimpan hasilnya pada layer image. |
| `ENV` | Menetapkan environment variable yang tersedia saat runtime. |
| `ARG` | Menetapkan variable yang tersedia selama proses build. |
| `USER` | Menetapkan user atau group untuk instruksi berikutnya dan proses runtime. |
| `EXPOSE` | Mendokumentasikan port aplikasi; tidak membuka port host secara otomatis. |
| `CMD` | Menetapkan perintah default yang dapat dioverride saat `docker run`. |
| `ENTRYPOINT` | Menetapkan executable utama yang tetap digunakan oleh container. |
| `HEALTHCHECK` | Menetapkan pemeriksaan kesehatan proses atau endpoint aplikasi. |
| `docker build` | Membuat image dari Dockerfile. `--progress=plain` menampilkan log build rinci, sedangkan `--target` memilih stage tertentu. |
| `docker history` | Memeriksa layer yang terbentuk dan membantu menemukan layer berukuran besar. |
| `docker inspect` | Membaca metadata container, termasuk status healthcheck. |

```bash
$ docker run --rm myapp:1 node --version
```

Hanya 1 CMD efektif (terakhir menang). Bentuk shell `CMD node server.js` rapuh sinyal — pakai exec JSON.

## 11. ENTRYPOINT

Wrapper tetap, CMD jadi parameternya. Cocok untuk CLI image.

```dockerfile
ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["node", "server.js"]
```

```bash
$ docker run --rm myapp:1 sh -c "echo override CMD"
```

Pola: entrypoint siapkan config/permission, lalu `exec "$@"`.

## 12. HEALTHCHECK

Cek kesehatan dari dalam container (selain cek luar di Compose/K8s).

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD wget -qO- http://127.0.0.1:3000/health || exit 1
```

```bash
$ docker ps --format "{{.Names}} {{.Status}}"
$ docker inspect -f '{{.State.Health.Status}}' myapp
```

Healthcheck gagal ≠ auto-restart (itu tugas restart policy/orchestrator, Bab 17/21).

## 13. Build & Debug

Debug layer per layer:

```bash
$ docker build --progress=plain -t myapp:debug . 2>&1 | tail -n 80
$ docker build --no-cache --progress=plain -t myapp:1 .
$ docker history myapp:1 | head -n 20
$ docker run --rm -it myapp:1 sh
$ docker build --target build -t myapp:build .   # stop di stage tertentu
```

Kasus umum:
- `COPY failed: no such file` → konteks salah / `.dockerignore` terlalu agresif.
- `apt 404` → indeks pada image dasar sudah usang; jalankan `apt update` pada instruksi RUN yang sama.
- permission denied → `USER` terlalu awal sebelum `COPY/chown` atau `npm install`.
- cache menipu → `--no-cache` sekali untuk pastikan.

Contoh Dockerfile final rapi:

```dockerfile
FROM node:20-alpine AS base
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY --chown=node:node . .
USER node
EXPOSE 3000
ENV NODE_ENV=production
HEALTHCHECK --interval=30s --timeout=3s CMD wget -qO- http://127.0.0.1:3000/health || exit 1
CMD ["node", "server.js"]
```
