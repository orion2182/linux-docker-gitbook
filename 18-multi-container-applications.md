# 18. Multi-Container Applications

> App nyata tidak pernah 1 container. Bab ini pola web + db + redis + worker + queue yang terbukti di prod kecil-menengah.

## Tujuan Pembelajaran

- Merancang web, database, redis, worker, queue
- Memisahkan internal vs public network + dependensi antar service

Stack contoh: Nginx (public) → Web/API → Postgres + Redis → Worker.

## 1. Web Application

Stateless, bisa direplika, config via env, log ke stdout.

```yaml
services:
  web:
    image: myapp/web:1.4.0
    ports: ["8080:3000"]
    env_file: [.env]
    networks: [frontend, backend]
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://127.0.0.1:3000/health"]
```

Stateless = bisa `up --scale web=3` (di balik LB) tanpa pusing session. Session taruh di Redis, bukan memori lokal.

## 2. Database

Stateful, 1 volume khusus, password via secret/env, healthcheck `pg_isready`.

```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: ${DB_PASS:?}
      POSTGRES_DB: appdb
    volumes: [pgdata:/var/lib/postgresql/data]
    networks: [backend]
    restart: unless-stopped
volumes:
  pgdata: {}
```

Aturan: DB tidak publish port ke publik. Hanya ke `backend`. Backup otomatis (Bab 24).

## 3. Redis

Cache + session + pub/sub ringan. Persistensi tergantung butuh: cache murni boleh tanpa volume, queue/session penting wajib volume + AOF.

```yaml
services:
  redis:
    image: redis:7-alpine
    command: ["redis-server", "--appendonly", "yes"]
    volumes: [redisdata:/data]
    networks: [backend]
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
volumes:
  redisdata: {}
```

```bash
$ docker compose exec redis redis-cli ping
$ docker compose exec redis redis-cli info memory | head
```

## 4. Worker

Proses background (kirim email, resize gambar, laporan). Image sama dengan web tapi command beda.

```yaml
services:
  worker:
    image: myapp/web:1.4.0
    command: ["node", "worker.js"]
    env_file: [.env]
    networks: [backend]
    depends_on:
      db: { condition: service_healthy }
      redis: { condition: service_healthy }
    restart: unless-stopped
```

Worker harus idempotent (dijalankan 2x aman) + graceful shutdown (`stop_grace_period: 30s`).

## 5. Queue

Antrian = Redis list/stream atau RabbitMQ untuk butuh durability + routing.

Pola Redis queue:

```yaml
services:
  api:
    environment:
      REDIS_URL: redis://redis:6379/0
      QUEUE: "jobs"
```

Pola RabbitMQ (lebih berat tapi andal):

```yaml
services:
  rabbit:
    image: rabbitmq:3-management-alpine
    volumes: [rabbitdata:/var/lib/rabbitmq]
    networks: [backend]
```

Pilih Redis kalau job sederhana + boleh retry. RabbitMQ kalau butuh ack, routing, DLQ serius.

## 6. Internal Networks

```yaml
networks:
  frontend: {}
  backend: {}
  # web: frontend+backend, api/worker: backend, db/redis/rabbit: backend only
```

Hasil: DB/Redis tidak bisa dijangkau langsung dari Nginx/public. Serangan permukaan kecil. Verifikasi:

```bash
$ docker exec myapp-web-1 ping -c1 db || echo "benar: web (frontend) tidak lihat db langsung kalau tidak se-network"
$ docker exec myapp-api-1 ping -c1 db
```

## 7. Public Networks

Hanya 1 pintu: reverse proxy (Nginx, Bab 22). Lainnya `expose` internal saja.

```yaml
services:
  proxy:
    image: nginx:alpine
    ports: ["80:80", "443:443"]
    networks: [frontend]
  web:
    expose: ["3000"]
    networks: [frontend, backend]
```

Jangan `ports: ["5432:5432"]` untuk DB di VPS publik. Kalau butuh admin, pakai SSH tunnel / profile tools sesaat.

## 8. Service Dependencies

Alur dependensi yang sehat adalah proxy → web/API → database atau Redis → worker yang membaca queue.

```yaml
services:
  api:
    depends_on:
      db: { condition: service_healthy }
      redis: { condition: service_healthy }
  worker:
    depends_on:
      db: { condition: service_healthy }
      redis: { condition: service_healthy }
```

Plus: app retry dengan backoff (DB start 10–20 detik pertama). `depends_on` tanpa healthcheck = start bareng tapi belum siap.

Contoh compose penuh (ringkas):

```yaml
services:
  proxy:
    image: nginx:alpine
    ports: ["80:80"]
    volumes: ["./proxy.conf:/etc/nginx/conf.d/default.conf:ro"]
    networks: [frontend]
  web:
    image: myapp/web:1.4.0
    env_file: [.env]
    networks: [frontend, backend]
  worker:
    image: myapp/web:1.4.0
    command: ["node", "worker.js"]
    env_file: [.env]
    networks: [backend]
  db:
    image: postgres:16-alpine
    volumes: [pgdata:/var/lib/postgresql/data]
    networks: [backend]
  redis:
    image: redis:7-alpine
    volumes: [redisdata:/data]
    networks: [backend]
```

Operasi:

```bash
$ docker compose config
$ docker compose up -d --build
$ docker compose ps
$ docker compose logs -f web worker
```

## Fungsi Perintah dan Field

| Perintah atau field | Fungsi |
| --- | --- |
| `docker compose config` | Memvalidasi dan merender konfigurasi multi-container sebelum dijalankan. |
| `docker compose up` | Membuat dan menjalankan seluruh service beserta network dan volume yang diperlukan. |
| `docker compose ps` | Memeriksa status, port, dan kondisi health service. |
| `docker compose logs` | Membaca log dari satu atau beberapa service. |
| `docker compose exec` | Menjalankan perintah di dalam service yang aktif, misalnya `redis-cli` atau `psql`. |
| `image` | Menentukan image yang digunakan service. |
| `build` | Menentukan konteks dan Dockerfile untuk membangun image service. |
| `command` | Mengganti perintah default image, misalnya menjalankan worker. |
| `ports` | Mem-publish port service ke host atau publik. Hindari pada database dan queue internal. |
| `expose` | Mendokumentasikan port yang tersedia untuk komunikasi internal tanpa mem-publish ke host. |
| `networks` | Menentukan zona komunikasi service, misalnya `frontend` dan `backend`. |
| `volumes` | Menyimpan data stateful di luar writable layer container. |
| `depends_on` | Menentukan service yang harus tersedia sebelum service lain dijalankan. |
| `healthcheck` | Menguji kesiapan aplikasi, bukan hanya keberadaan prosesnya. |
