# 23. Docker Registry & Image Distribution

> Registry merupakan gudang image. Tanpa strategi tag dan digest, deployment menjadi tidak dapat diprediksi jika hanya mengandalkan `latest`.

## Tujuan Pembelajaran

- Docker Hub vs private registry, login/push/pull, tagging, digest, security

## 1. Docker Hub

Default publik. Cocok untuk base image + open source. Private repo gratis terbatas.

```bash
$ docker pull nginx:alpine
$ docker search --limit 5 nginx
$ docker logout; docker login
$ cat ~/.docker/config.json | head -n 20
```

Jangan push image berisi secret/env prod ke Hub publik. Cek `.dockerignore` + history sebelum push.

## 2. Private Registry

Untuk image internal. Opsi: Docker Hub private, GHCR, GitLab, ECR/GAR, atau self-host `registry:2`.

Self-host minimal:

```yaml
services:
  registry:
    image: registry:2
    ports: ["127.0.0.1:5000:5000"]
    volumes: [regdata:/var/lib/registry]
    restart: unless-stopped
volumes:
  regdata: {}
```

```bash
$ curl http://127.0.0.1:5000/v2/_catalog
```

Prod: taruh di balik TLS + auth (basic/htpasswd atau cloud IAM) + backup `regdata`.

## 3. docker login

```bash
$ docker login
$ docker login registry.example.com
$ docker login ghcr.io -u USERNAME --password-stdin < token.txt
$ cat ~/.docker/config.json
```

Di server/CI pakai token read-only + expiry, bukan password utama. Credential helper (`pass`/`secretservice`) lebih aman dari plaintext.

## 4. docker push

```bash
$ docker tag myapp/api:1.4.0 registry.example.com/team/myapp:1.4.0
$ docker push registry.example.com/team/myapp:1.4.0
$ docker push registry.example.com/team/myapp:1.4   # alias minor
```

Push setelah `build --no-cache` sesekali untuk pastikan reproducible + scan (lihat Security).

## 5. docker pull

```bash
$ docker pull registry.example.com/team/myapp:1.4.0
$ docker compose pull
$ docker pull --platform linux/amd64 myapp/api:1.4.0
```

Di prod: `pull` eksplisit sebelum `up -d` agar tahu persis digest yang jalan. Catat digest di log rilis.

## 6. Image Tagging Strategy

Gunakan skema tagging yang konsisten:

- `1.4.0` immutable per rilis (git tag sama).
- `1.4`, `1` bergerak untuk patch/minor.
- `latest` hanya untuk dev, jangan di prod.
- `sha-<git-short>` untuk trace commit.

```bash
$ docker build -t registry.example.com/team/myapp:1.4.0 -t registry.example.com/team/myapp:1.4 .
$ docker tag registry.example.com/team/myapp:1.4.0 registry.example.com/team/myapp:sha-a1b2c3d
$ docker push -a registry.example.com/team/myapp
```

Aturan: prod selalu pin `1.4.0` / digest, bukan `latest`.

## 7. Image Digests

Hash konten (`sha256:...`). Tag bisa geser, digest tidak.

```bash
$ docker images --digests | grep myapp
$ docker pull registry.example.com/team/myapp:1.4.0
$ docker inspect -f '{{.RepoDigests}}' registry.example.com/team/myapp:1.4.0
# pin di compose prod:
# image: registry.example.com/team/myapp@sha256:abc123...
```

Untuk rollback presisi + audit, simpan digest tiap deploy.

## 8. Registry Security

- TLS wajib + auth + Least privilege token (pull vs push pisah).
- Scan image: `docker scout`, Trivy, atau scanner registry cloud. Blokir HIGH/CRITICAL ke prod.
- Base image pin + update terjadwal (jangan `alpine:latest` liar).
- Prune + retention: hapus tag lama, simpan N rilis terakhir untuk rollback.
- Audit log pull/push + backup storage registry (S3/volume).

```bash
$ trivy image --severity HIGH,CRITICAL registry.example.com/team/myapp:1.4.0 | head -n 60
$ docker scout quickview registry.example.com/team/myapp:1.4.0 2>&1 | head -n 40
$ curl -u user:pass https://registry.example.com/v2/_catalog
```

## Fungsi Perintah dan Konsep

| Perintah atau konsep | Fungsi |
| --- | --- |
| `docker login` / `logout` | Menyimpan atau menghapus kredensial registry pada host atau credential helper. |
| `docker pull` | Mengunduh image atau manifest dari registry. |
| `docker push` | Mengunggah layer dan manifest image ke registry. |
| `docker tag` | Memberikan nama dan versi repository pada image lokal. |
| `docker images --digests` | Menampilkan digest yang terkait dengan tag image lokal. |
| `docker inspect` | Membaca `RepoDigests` dan metadata image. |
| `curl /v2/_catalog` | Menguji API registry dan melihat repository yang tersedia jika akses mengizinkan. |
| `trivy image` | Memindai image untuk menemukan kerentanan pada package dan library. |
| `docker scout` | Menganalisis image dan rekomendasi keamanan bila Docker Scout tersedia. |
| `registry:2` | Image registry self-hosted berbasis Distribution Registry. |
| Tag | Nama mutable yang menunjuk ke manifest image; dapat berpindah ke digest lain. |
| Digest | Hash immutable manifest image yang cocok untuk deployment presisi. |
