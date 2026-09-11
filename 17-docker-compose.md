# 17. Docker Compose

> Compose = 1 file YAML untuk seluruh app. Kalau Bab 12 adalah kata, Compose adalah kalimat.

## Tujuan Pembelajaran

- Nulis compose.yaml: services, networks, volumes, env, secrets, healthcheck, depends_on, restart, profiles, build
- Menjalankan lifecycle: up/down/ps/logs/pull/build

Contoh acuan di bab ini: web (nginx) + api (node) + db (postgres).

## 1. compose.yaml

Nama modern `compose.yaml` (bukan `docker-compose.yml` lama, tapi tetap didukung). Plugin: `docker compose` (spasi, bukan strip).

```bash
$ docker compose version
$ ls -la compose.yaml .env
$ docker compose config   # validasi + render final (wajib sebelum up!)
```

Selalu `config` dulu untuk tangkap typo env/substitusi.

## 2. Services

Tiap container = 1 service.

```yaml
services:
  web:
    image: nginx:alpine
    ports: ["8080:80"]
  api:
    build: ./api
    expose: ["3000"]
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: ${DB_PASS:?wajib isi}
```

```bash
$ docker compose up -d
$ docker compose ps
```

1 service = 1 tanggung jawab. Jangan gabung nginx+api+db dalam 1 image.

## 3. Networks

```yaml
services:
  web:
    networks: [frontend]
  api:
    networks: [frontend, backend]
  db:
    networks: [backend]
networks:
  frontend: {}
  backend: {}
```

Hasil: web ↔ api bisa, api ↔ db bisa, web ↛ db langsung. Isolasi gratis (lihat Bab 16/18).

## 4. Volumes

```yaml
services:
  db:
    volumes: ["pgdata:/var/lib/postgresql/data"]
  web:
    volumes: ["./nginx.conf:/etc/nginx/conf.d/default.conf:ro"]
volumes:
  pgdata: {}
```

Named untuk data, bind `:ro` untuk config. Cek dengan `docker compose config` + `inspect`.

## 5. Environment Variables

Urutan prioritas: `environment:` > `--env-file` > `env_file:` > `.env` + shell.

```yaml
services:
  api:
    env_file: [.env]
    environment:
      NODE_ENV: production
      PORT: "3000"
```

```bash
$ cat .env
DB_PASS=kuat-sekali
$ docker compose config | grep -A5 api
```

Jangan commit `.env` prod (Bab 21). Bedakan `.env.example` untuk template.

## 6. .env

File `key=value` di sebelah compose.yaml, otomatis dibaca untuk substitusi `${VAR}`.

```text
COMPOSE_PROJECT_NAME=myapp
DB_PASS=kuat-sekali
API_TAG=1.4.0
```

```yaml
services:
  api:
    image: "myapp/api:${API_TAG:-latest}"
```

Test substitusi: `docker compose config | head -n 60`.

## 7. Secrets

Jangan taruh password di `environment` plaintext kalau bisa file (terutama swarm / prod).

```yaml
services:
  db:
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_pass
    secrets: [db_pass]
secrets:
  db_pass:
    file: ./secrets/db_pass.txt
```

```bash
$ ls -l secrets/
$ cat secrets/db_pass.txt  # permission 600, gitignore!
```

Untuk single-host Compose biasa, `env_file` + permission ketat + backup terenkripsi sudah cukup (detail Bab 21).

## 8. Healthchecks

Beda dari Dockerfile HEALTHCHECK: ini level Compose/orchestrator untuk gating.

```yaml
services:
  api:
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://127.0.0.1:3000/health"]
      interval: 15s
      timeout: 3s
      retries: 3
      start_period: 20s
```

```bash
$ docker compose ps --format "table {{.Name}}\t{{.Status}}"
$ docker inspect -f '{{.State.Health.Status}}' myapp-api-1
```

## 9. depends_on

Urutan start, bukan kesiapan! Tanpa healthcheck, DB belum tentu siap saat api start.

```yaml
services:
  api:
    depends_on:
      db:
        condition: service_healthy
  db:
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
```

Api yang bagus tetap retry koneksi DB (backoff), jangan andalkan `depends_on` saja.

## 10. Restart Policies

```yaml
services:
  web:
    restart: unless-stopped
  api:
    restart: unless-stopped
  job-sekali:
    restart: "no"
```

- `no` = job/manual.
- `unless-stopped` = server/app standar (survive reboot, hormati stop manual).
- `always` = kiosk/edge yang harus nyala walau sempat di-stop.
- `on-failure` = worker yang boleh mati sukses.

## 11. Profiles

Aktifkan service opsional (adminer, seed, debug) tanpa ganggu default.

```yaml
services:
  adminer:
    image: adminer
    profiles: [tools]
    ports: ["8081:8080"]
```

```bash
$ docker compose up -d
$ docker compose --profile tools up -d
$ docker compose --profile tools ps
```

## 12. Build

Build dari Compose (dev/staging). Prod lebih baik pull image jadi dari registry (Bab 23).

```yaml
services:
  api:
    build:
      context: ./api
      dockerfile: Dockerfile
      target: runtime
      args:
        BUILD_DATE: ${BUILD_DATE:-dev}
    image: myapp/api:1.4.0
```

```bash
$ docker compose build
$ docker compose build --no-cache api
$ docker compose up -d --build
```

## 13. Compose Lifecycle

Hafalkan:

```bash
$ docker compose config
$ docker compose pull
$ docker compose up -d
$ docker compose ps
$ docker compose logs -f --tail 100
$ docker compose exec api sh
$ docker compose stop
$ docker compose start
$ docker compose down
$ docker compose down -v   # HATI-HATI: hapus volume!
```

- `stop` = mati bisa nyala lagi. `down` = hapus container+network (volume aman kecuali `-v`).
- Update: `pull`/`build` → `up -d` lagi. Rollback: ganti tag → `up -d`.

Contoh lengkap:

```yaml
services:
  web:
    image: nginx:alpine
    ports: ["8080:80"]
    volumes: ["./nginx.conf:/etc/nginx/conf.d/default.conf:ro"]
    networks: [frontend]
    restart: unless-stopped
  api:
    image: myapp/api:1.4.0
    env_file: [.env]
    networks: [frontend, backend]
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: ${DB_PASS:?wajib isi}
    volumes: ["pgdata:/var/lib/postgresql/data"]
    networks: [backend]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      retries: 5
    restart: unless-stopped
networks:
  frontend: {}
  backend: {}
volumes:
  pgdata: {}
```

## Latihan

1. Tulis compose 3 service di atas dari nol, `config`, `up`, `ps`, `logs`.
2. Matikan 1 service, buktikan `depends_on` + healthcheck menahan start.
3. Tambah profile `tools` berisi adminer, up dengan dan tanpa profile.
4. Simulasikan update tag + rollback via `up -d`.

## Rangkuman

- `config → pull/build → up -d → ps/logs` = siklus Compose.
- Network/volume/env/secrets/healthcheck/depends_on/restart = fondasi prod (Bab 18/21).
- `down -v` adalah perintah paling berbahaya di bab ini. Pahami sebelum tekan enter.
