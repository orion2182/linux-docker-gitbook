# 21. Production Docker on VPS

> Dari `docker run` coba-coba ke susunan prod yang bisa diupdate dan rollback tanpa keringat dingin.

## Tujuan Pembelajaran

- Struktur direktori, env, secret, proxy, TLS, healthcheck, restart, limit, logging, update, rollback

Asumsi: 1 VPS Ubuntu + Compose + Nginx (detail Nginx di Bab 22).

## 1. Production Directory Structure

Satu app = satu folder, versioning Git, data di volume (bukan di folder app).

```text
/opt/myapp/
  compose.yaml
  .env                 # 600, gitignore, backup terenkripsi
  .env.example         # template tanpa secret
  nginx/app.conf
  secrets/db_pass.txt  # 600, gitignore
  scripts/backup.sh
  README.md            # cara deploy/rollback
```

```bash
$ ls -l /opt/myapp/
$ cat /opt/myapp/.env.example
$ sudo chown -R root:docker /opt/myapp
$ sudo chmod 600 /opt/myapp/.env
```

Jangan deploy dari `/tmp` atau home random yang tidak dibackup.

## 2. Environment Management

Bedakan `dev/staging/prod` via file terpisah + substitusi tag.

```bash
$ cp .env.example .env.prod
$ cat compose.yaml | grep -E "image:|env_file"
```

```yaml
services:
  api:
    image: "myapp/api:${API_TAG:?isi tag}"
    env_file: [.env.prod]
```

Deploy: `API_TAG=1.4.0 docker compose --env-file .env.prod up -d`. Jangan edit `.env` langsung di prod tanpa commit/template.

## 3. Secrets

File 600 + gitignore + backup terenkripsi. Jangan di image/Git/log.

```bash
$ mkdir -p secrets && openssl rand -hex 24 > secrets/db_pass.txt
$ chmod 600 secrets/db_pass.txt .env.prod
$ cat .gitignore
# .env*
# !.env.example
# secrets/
```

```yaml
services:
  db:
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_pass
    secrets: [db_pass]
secrets:
  db_pass: { file: ./secrets/db_pass.txt }
```

Rotasi: ganti file → recreate service → test → hapus backup lama.

## 4. Reverse Proxy

Hanya proxy yang publish 80/443. App lain di network internal.

```yaml
services:
  proxy:
    image: nginx:alpine
    ports: ["80:80", "443:443"]
    volumes: ["./nginx/app.conf:/etc/nginx/conf.d/default.conf:ro", "certs:/etc/nginx/certs:ro"]
  web:
    expose: ["3000"]
    networks: [frontend, backend]
```

Detail config + TLS di Bab 22. Prinsip di sini: 1 pintu masuk.

## 5. TLS

Wajib HTTPS di prod (Let's Encrypt gratis, auto-renew via timer).

```bash
$ sudo certbot --nginx -d contoh.com -d www.contoh.com
$ sudo systemctl status certbot.timer
$ curl -vI https://contoh.com 2>&1 | head -n 20
```

Jangan terminasi TLS di tiap container. Sentral di proxy, internal HTTP biasa (kecuali compliance minta mTLS).

## 6. Healthchecks

Semua service penting punya healthcheck + `depends_on healthy`.

```yaml
services:
  db:
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
  api:
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://127.0.0.1:3000/health"]
    depends_on:
      db: { condition: service_healthy }
```

```bash
$ docker compose ps --format "table {{.Name}}\t{{.Status}}"
```

Healthcheck = syarat deploy lanjut / rollback (lihat Update/Rollback).

## 7. Restart Policies

```yaml
services:
  proxy: { restart: unless-stopped }
  web: { restart: unless-stopped }
  worker: { restart: unless-stopped }
  db: { restart: unless-stopped }
```

`unless-stopped` = survive reboot tapi hormati stop manual. Test: `sudo reboot` di staging, pastikan semua naik + healthy.

## 8. Resource Limits

Biar 1 service bocor tidak bunuh host.

```yaml
services:
  api:
    deploy:
      resources:
        limits: { cpus: "1.0", memory: 512M }
    pids_limit: 200
```

```bash
$ docker stats --no-stream
```

Set dari observasi staging + margin 30%, bukan tebak. Alert 80% (Bab 25).

## 9. Logging

Log ke stdout + batasi + rotasi. Jangan log ke file di writable layer.

```yaml
services:
  api:
    logging:
      driver: json-file
      options: { max-size: "10m", max-file: "3" }
```

```bash
$ docker compose logs -f --tail 100 api
$ docker system df -v | grep -i log | head
$ cat /etc/docker/daemon.json | grep -A4 log-opts
```

Untuk agregasi: forward ke Loki/syslog/ELK (Bab 25). Retensi jelas (misal 14 hari lokal).

## 10. Updates

Alur aman (staging dulu!):

```bash
$ cd /opt/myapp
$ cp compose.yaml compose.yaml.bak-$(date +%F)
$ docker compose pull
$ docker compose up -d
$ docker compose ps
$ curl -f http://127.0.0.1/health || echo "GAGAL, rollback!"
$ docker compose logs --since 5m | tail -n 50
```

Untuk build sendiri: push tag baru ke registry (Bab 23) → ubah `API_TAG` → `up -d`. Jangan `build` di prod dari branch random.

## 11. Rollback

Rollback = ganti tag ke sebelumnya + `up -d` + verifikasi. Harus <5 menit.

```bash
$ API_TAG=1.3.9 docker compose up -d api
$ docker compose ps
$ curl -f http://127.0.0.1/health
# kalau DB migrasi: siapkan down-migration / restore backup (Bab 24), jangan asal rollback code saja
$ git log --oneline -n 5
$ git diff HEAD~1 compose.yaml | head -n 60
```

Latih rollback di staging tiap rilis besar. Rollback yang belum pernah dites = bukan rollback.

## Latihan

1. Susun `/opt/myapp` sesuai struktur + `.env.example` + gitignore.
2. Terapkan limit + logging + healthcheck + restart, `config` lalu `up`.
3. Reboot staging, catat waktu sampai semua healthy.
4. Update tag dummy + rollback, ukur MTTR kamu.

## Rangkuman

- Prod = struktur + env/secret disiplin + 1 pintu proxy + TLS + health + limit + log + update/rollback terlatih.
- Semua di Git kecuali secret. Semua secret dibackup terenkripsi.
- Kalau belum pernah rollback, belum siap prod.
