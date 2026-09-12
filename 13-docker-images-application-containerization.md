# 13. Docker Images & Application Containerization

> Image = cetakan, container = hasil cetakan yang hidup. Bab ini cara bungkus aplikasi jadi image yang ramping dan benar.

## Tujuan Pembelajaran

- Menjelaskan layer, tag, build context, build/tag
- Containerize app + dependency + env + .dockerignore
- Multi-stage build + optimasi ukuran

## 1. What is Docker Image?

Template read-only berlapis. Container = image + writable layer tipis.

```bash
$ docker images
$ docker image ls --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"
$ docker pull nginx:alpine
```

Image immutable by digest, tag bisa geser (lihat Bab 23).

## 2. Image Layers

Tiap instruksi Dockerfile = 1 layer (cacheable). Ubah layer atas tidak invalidate bawah kalau urutan tepat.

```bash
$ docker history nginx:alpine | head -n 20
$ docker inspect nginx:alpine | grep -i layers | head
```

Prinsip: yang jarang berubah (OS, dependency) taruh atas; code yang sering berubah taruh bawah.

## 3. Image Tags

Label versi. `latest` = bergerak, jangan andalkan di prod.

```bash
$ docker pull node:20-alpine
$ docker pull node:20.11.1-alpine
$ docker images | grep node
```

Skema disarankan: `app:1.4.0` + `app:1.4` + `app:1` (+ digest untuk pin absolut, Bab 23).

## 4. Build Context

Folder yang dikirim ke daemon saat build. Konteks besar = build lambat.

```bash
$ ls -la
$ du -sh .; du -sh node_modules .git 2>/dev/null
$ docker build -t demo:1 .
```

Jangan build dari `/` atau home penuh. Buat folder app khusus.

## 5. docker build

```bash
$ docker build -t myapp:1.0 .
$ docker build --no-cache -t myapp:1.0 .
$ DOCKER_BUILDKIT=1 docker build --progress=plain -t myapp:1.0 .
$ docker images | grep myapp
```

BuildKit = builder modern (cache paralel, secret aman, lebih cepat). Aktifkan default di Docker baru.

## 6. docker tag

Beri nama tambahan (misal versi + latest staging).

```bash
$ docker tag myapp:1.0 myapp:latest
$ docker tag myapp:1.0 registry.example.com/team/myapp:1.0
$ docker images | grep myapp
```

Tag tidak duplikat data, cuma alias.

## 7. Application Containerization

Contoh Node sederhana:

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```

```bash
$ docker build -t myapp:1.0 .
$ docker run -d --name myapp -p 3000:3000 myapp:1.0
$ curl -I http://127.0.0.1:3000
$ docker logs myapp | tail
```

Kunci: 1 container = 1 proses utama, log ke stdout, config via env (bukan hardcode).

## 8. Dependencies

Kunci versi + manfaatkan cache:

```dockerfile
COPY package.json package-lock.json ./
RUN npm ci --only=production
```

- Node: `npm ci` + lockfile. Python: `pip install --no-cache-dir -r requirements.txt`. Go: `go mod download` sebelum copy code.
- Jangan `apt upgrade` liar di tiap build — non-deterministik.

## 9. Environment Variables

```bash
$ docker run -d --name myapp -e NODE_ENV=production -e PORT=3000 myapp:1.0
$ docker exec myapp env | sort
$ docker run -d --env-file ./app.env myapp:1.0
```

`ENV` di Dockerfile = default, `-e/--env-file` = override. Secret asli jangan di ENV image (Bab 19/21).

## 10. .dockerignore

Seperti `.gitignore` tapi untuk konteks build. Wajib biar ramping + aman.

```text
node_modules
.git
.env
*.log
coverage/
dist/
.DS_Store
```

```bash
$ cat .dockerignore
```

Tanpa file ini, `node_modules` lokal dan direktori `.git` ikut dikirim sehingga build menjadi lambat dan berisiko membocorkan informasi.

## 11. Multi-stage Build

Pisah builder (besar) dan runtime (kecil). Hasil akhir hanya bawa binary.

```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine AS runtime
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY package*.json ./
RUN npm ci --only=production
CMD ["node", "dist/server.js"]
```

```bash
$ docker build -t myapp:slim .
$ docker images | grep myapp
```

Bisa potong 70–90% ukuran untuk Go/Node/Frontend.

## 12. Image Optimization

Checklist:

- Base `alpine` / `slim` / `distroless` kalau cocok.
- Gabung `RUN` + bersihkan cache: `apt-get clean && rm -rf /var/lib/apt/lists/*`, `pip --no-cache-dir`, `npm cache clean`.
- Urutan Dockerfile: dependency dulu, code belakangan.
- `.dockerignore` ketat.
- Cek hasil: `docker images`, `docker history`, `dive` (opsional).

```bash
$ docker history myapp:1.0 | head -n 20
$ docker images --format "{{.Repository}}:{{.Tag}} {{.Size}}"
```

## Fungsi Perintah dan Instruksi

| Perintah atau instruksi | Fungsi |
| --- | --- |
| `docker pull` | Mengunduh image dari registry ke host. |
| `docker images` / `docker image ls` | Menampilkan image lokal beserta tag dan ukurannya. |
| `docker history` | Menampilkan layer dan instruksi yang membentuk image. |
| `docker build` | Membuat image dari Dockerfile dan build context. `-t` memberi nama/tag, sedangkan `--no-cache` mematikan cache. |
| `docker tag` | Membuat alias repository dan tag untuk image yang sama. |
| `docker run` | Membuat container dari image. `-p` memetakan port dan `-e` menetapkan konfigurasi runtime. |
| `docker inspect` | Menampilkan metadata image atau container dalam format JSON. |
| `du` | Mengukur ukuran direktori build context atau dependency. |
| `grep` | Memilih output yang sesuai dengan pola tertentu. |
| `COPY` | Menyalin file dari build context ke image. |
| `RUN` | Menjalankan perintah ketika image sedang dibangun. |
| `WORKDIR` | Menetapkan direktori kerja untuk instruksi berikutnya dan proses runtime. |
| `ENV` / `ARG` | Menetapkan nilai runtime pada image atau nilai yang hanya tersedia saat build. |
| `EXPOSE` | Mendokumentasikan port aplikasi; tidak mem-publish port ke host. |
| `CMD` / `ENTRYPOINT` | Menetapkan proses default saat container dijalankan. |
