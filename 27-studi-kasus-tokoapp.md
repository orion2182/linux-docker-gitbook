# 27. Studi Kasus TokoApp: Dari Source sampai Docker Hub

> Bab ini membangun ulang stack TokoApp dari direktori source dan file konfigurasi, membungkus setiap komponen menjadi image Docker, menguji stack dengan Compose, lalu melakukan tag dan push ke Docker Hub.

Dockerfile asli dari image publik tidak tersedia. Karena itu, contoh pada bab ini adalah implementasi ulang yang mengikuti kontrak, port, endpoint, dan konfigurasi yang ditemukan pada image `azeshion21/toko-*`.

## Tujuan Pembelajaran

Setelah menyelesaikan bab ini, kamu dapat:

- Memulai project dari direktori kosong.
- Memahami fungsi setiap file sebelum file tersebut masuk ke image.
- Membuat Dockerfile untuk API, frontend, database, proxy, dan monitoring.
- Menghubungkan semua komponen dengan Docker Compose.
- Memvalidasi konfigurasi sebelum menjalankan container.
- Membuat image dengan tag versi dan mendorongnya ke Docker Hub.
- Menarik kembali image tersebut dan menjalankannya di VPS.

## 1. Model Mental: Source sampai Container

Jangan menganggap image sebagai aplikasi yang muncul secara otomatis. Image adalah hasil dari proses build.

```text
Source code dan konfigurasi
        |
        | Dockerfile + build context
        v
docker build
        |
        v
Image lokal
        |
        | docker tag + docker push
        v
Docker Hub Registry
        |
        | docker pull
        v
Container yang berjalan
```

Perbedaan penting:

| Objek | Isi |
| --- | --- |
| Source code | File aplikasi seperti Go, JavaScript, Python, dan SQL. |
| Konfigurasi | File YAML, Nginx, Grafana, Prometheus, dan environment variable. |
| Dockerfile | Instruksi untuk mengubah source menjadi image. |
| Image | Paket immutable yang berisi filesystem, runtime, binary, dan konfigurasi default. |
| Container | Instance image yang sedang berjalan. |
| Registry | Tempat menyimpan dan mendistribusikan image, misalnya Docker Hub. |

## 2. Arsitektur TokoApp

```text
Client
  |
  v
toko-nginx:80/443
  |----------------------> toko-web:3000
  |                        Next.js
  `----------------------> toko-api:8080
                              |
                              v
                           toko-pg:5432
                           PostgreSQL

Prometheus
  |-- node-exporter
  |-- cadvisor
  |-- postgres-exporter
  `-- toko-api /metrics
       |
       v
  Alertmanager -> telegram-notify -> Telegram Bot API

Grafana -> Prometheus
```

Alur request:

1. Client mengakses Nginx.
2. Nginx meneruskan `/api/` ke API.
3. Nginx meneruskan path lain ke frontend Next.js.
4. API membaca dan mengubah data PostgreSQL.
5. Prometheus mengambil metrics dari exporter dan API.
6. Alertmanager mengirim alert ke notifier.

## 3. Struktur Direktori dari Awal

Buat direktori project baru:

```bash
$ mkdir -p tokoapp/{api,web,postgres,telegram-notify}
$ mkdir -p tokoapp/infra/{nginx,prometheus/rules,alertmanager,grafana/datasources,grafana/dashboards}
$ cd tokoapp
```

Fungsi perintah:

- `mkdir -p` membuat direktori beserta parent directory yang belum ada.
- `{api,web,...}` adalah brace expansion Bash untuk membuat beberapa direktori sekaligus.
- `cd` berpindah ke direktori project.

Struktur akhirnya:

```text
tokoapp/
  api/
    Dockerfile
    go.mod
    go.sum
    cmd/
    internal/
  web/
    Dockerfile
    package.json
    package-lock.json
    next.config.mjs
    src/
  postgres/
    Dockerfile
    init.sql
  telegram-notify/
    Dockerfile
    notifier.py
  infra/
    nginx/
      Dockerfile
      default.conf
    prometheus/
      Dockerfile
      prometheus.yml
      rules/alerts.yml
    alertmanager/
      Dockerfile
      alertmanager.yml
    grafana/
      Dockerfile
      datasources/prometheus.yaml
      dashboards/dashboards.yaml
      dashboards/toko-overview.json
  compose.yaml
  .env.example
  .gitignore
```

## 4. File Environment dan Git Ignore

Buat `.env.example` sebagai dokumentasi variable yang diperlukan. File ini tidak berisi secret asli.

```text
DB_PASSWORD=ganti-dengan-password-kuat
DATABASE_URL=postgresql://appuser:ganti-dengan-password-kuat@pg:5432/mydb?sslmode=disable
DATA_SOURCE_NAME=postgresql://appuser:ganti-dengan-password-kuat@pg:5432/mydb?sslmode=disable
TELEGRAM_BOT_TOKEN=ganti-dengan-token-bot
TELEGRAM_CHAT_ID=ganti-dengan-chat-id
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=ganti-dengan-password-grafana
```


```bash
$ cp .env.example .env
$ chmod 600 .env
$ nano .env
```


```text
.env
.env.*
!.env.example
secrets/
*.pem
*.key
node_modules/
.next/
```



## 5. Source API Go

Image `toko-api` yang dianalisis berisi binary Go pada `/app/api`, port `8080`, dan endpoint berikut:

```text
POST   /api/auth/login
POST   /api/auth/logout
GET    /api/auth/me
GET    /api/health
GET    /api/metrics
GET    /api/products
POST   /api/products
DELETE /api/products/{id}
GET    /api/stats
```

Source API berada di luar image dan tidak dapat dipulihkan utuh dari binary yang sudah stripped. Untuk project baru, source minimal perlu memiliki:

- package HTTP handler untuk route API;
- package database untuk PostgreSQL;
- session store atau mekanisme token;
- middleware metrics;
- healthcheck database;
- `go.mod` dan `go.sum`.

Contoh kontrak `DATABASE_URL`:

```text
postgresql://appuser:${DB_PASSWORD}@pg:5432/mydb?sslmode=disable
```



## 6. Dockerfile API Go

Buat `api/Dockerfile`:

```dockerfile
FROM golang:1.24-alpine AS builder

WORKDIR /src

# Dependency dipisahkan agar layer cache tetap dapat digunakan.
COPY go.mod go.sum ./
RUN go mod download

# Source aplikasi baru disalin setelah dependency.
COPY . .

# Binary statis cocok untuk runtime Alpine yang kecil.
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -trimpath -ldflags="-s -w" -o /out/api .

FROM alpine:3.22

WORKDIR /app
RUN addgroup -S app && adduser -S -G app app
COPY --from=builder /out/api ./api
RUN chown app:app /app/api
USER app
EXPOSE 8080
CMD ["./api"]
```

Penjelasan instruksi:

- `FROM ... AS builder` membuat build stage yang berisi compiler Go.
- `WORKDIR` menetapkan direktori kerja.
- `COPY go.mod go.sum` menyalin daftar dependency terlebih dahulu agar cache efektif.
- `RUN go mod download` mengunduh dependency Go.
- `CGO_ENABLED=0` membuat binary tidak bergantung pada library C runtime.
- `GOOS=linux` dan `GOARCH=amd64` menetapkan target deployment.
- `-trimpath` menghapus path build dari binary.
- `-ldflags="-s -w"` mengurangi simbol debug dan ukuran binary.
- `FROM alpine` memulai runtime image yang lebih kecil.
- `USER app` mencegah aplikasi berjalan sebagai root.
- `EXPOSE 8080` mendokumentasikan port internal.
- `CMD` menetapkan proses utama container.


## 7. Source Frontend Next.js

Image `toko-web` menggunakan Node.js 22.23.2, Next.js 15.3.4, dan React 19.1.0. Buat `web/package.json`:

```json
{
  "name": "toko-app-web",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start"
  },
  "dependencies": {
    "next": "15.3.4",
    "react": "19.1.0",
    "react-dom": "19.1.0"
  }
}
```

Frontend TokoApp menggunakan route `/login` dan `/`. Request dari browser diarahkan ke:

```text
/api/auth/login
/api/auth/logout
/api/auth/me
/api/products
/api/stats
/api/health
```

Frontend yang dianalisis menyimpan bearer token pada `localStorage` dengan key `toko_token`. Pola ini dapat bekerja, tetapi token dapat dibaca JavaScript pada origin yang sama. Untuk production, pertimbangkan session cookie `HttpOnly`, `Secure`, dan `SameSite` setelah memastikan alur login kompatibel.


## 8. Dockerfile Frontend Next.js

Buat `web/Dockerfile`:

```dockerfile
FROM node:22-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

FROM node:22-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ENV NEXT_TELEMETRY_DISABLED=1
RUN npm run build

FROM node:22-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
EXPOSE 3000
CMD ["node", "server.js"]
```

`next.config.mjs` harus mengaktifkan standalone output agar direktori `.next/standalone` tersedia:

```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  output: "standalone",
};

export default nextConfig;
```



## 9. Database PostgreSQL dan init.sql

Buat `postgres/init.sql`. File ini dijalankan oleh entrypoint resmi PostgreSQL hanya ketika volume database masih kosong.

```sql
CREATE TABLE IF NOT EXISTS products (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    category TEXT NOT NULL,
    price BIGINT NOT NULL CHECK (price > 0),
    stock INT NOT NULL DEFAULT 0 CHECK (stock >= 0),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX IF NOT EXISTS idx_products_name
    ON products (lower(name));

CREATE INDEX IF NOT EXISTS idx_products_category
    ON products (category);

INSERT INTO products (name, category, price, stock)
VALUES
  ('Laptop Demo', 'Elektronik', 6900000, 14),
  ('Keyboard Demo', 'Elektronik', 385000, 32),
  ('Kopi Demo', 'Makanan & Minuman', 68000, 65)
ON CONFLICT DO NOTHING;

CREATE TABLE IF NOT EXISTS users (
    id SERIAL PRIMARY KEY,
    username TEXT NOT NULL UNIQUE,
    password_hash TEXT NOT NULL,
    full_name TEXT NOT NULL,
    role TEXT NOT NULL DEFAULT 'staff',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Fungsi bagian penting:

- `CREATE TABLE` membuat schema aplikasi.
- `PRIMARY KEY` memberi identifier unik.
- `NOT NULL` menolak nilai kosong.
- `CHECK` membatasi nilai yang tidak valid.
- `CREATE INDEX` mempercepat pencarian nama dan kategori.
- `ON CONFLICT DO NOTHING` mencegah seed gagal karena data sudah ada.
- Password harus berupa hash yang dibuat oleh aplikasi atau migration aman. Jangan menaruh password plaintext atau hash user demo pada image publik.


```dockerfile
FROM postgres:18.6-alpine
COPY init.sql /docker-entrypoint-initdb.d/init.sql
```



## 10. Nginx Reverse Proxy

Buat `infra/nginx/default.conf`:

```nginx
server {
    listen 80;
    server_name _;

    access_log /var/log/nginx/app_access.log;
    error_log /var/log/nginx/app_error.log;

    location /api/ {
        proxy_pass http://api:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 2s;
        proxy_read_timeout 30s;
    }

    location / {
        proxy_pass http://web:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 2s;
        proxy_read_timeout 30s;
    }
}
```

Penjelasan:

- `listen 80` menerima koneksi HTTP.
- `server_name _` menjadi catch-all untuk contoh lokal.
- `location /api/` memilih request API.
- `proxy_pass` meneruskan request ke nama service Compose, bukan IP hardcode.
- `proxy_set_header` meneruskan informasi host dan IP asli.
- `proxy_read_timeout` membatasi waktu tunggu response backend.
- Header `Upgrade` dan `Connection` membantu WebSocket pada frontend.


```dockerfile
FROM nginx:1.30.4-alpine
COPY default.conf /etc/nginx/conf.d/default.conf
EXPOSE 80 443
```



## 11. Telegram Notifier

Buat `telegram-notify/notifier.py`:

```python
import json
import os
import urllib.request
from http.server import BaseHTTPRequestHandler, HTTPServer

TOKEN = os.environ.get("TELEGRAM_BOT_TOKEN", "")
CHAT_ID = os.environ.get("TELEGRAM_CHAT_ID", "")
PORT = int(os.environ.get("PORT", "8081"))
API = f"https://api.telegram.org/bot{TOKEN}/sendMessage"


def send_telegram(text):
    if not TOKEN or not CHAT_ID:
        print("Telegram credentials belum diisi")
        return

    body = json.dumps({
        "chat_id": CHAT_ID,
        "text": text,
        "disable_web_page_preview": True,
    }).encode()
    request = urllib.request.Request(
        API,
        data=body,
        headers={"Content-Type": "application/json"},
    )
    with urllib.request.urlopen(request, timeout=10) as response:
        print("telegram status:", response.status)


class Handler(BaseHTTPRequestHandler):
    def do_POST(self):
        length = int(self.headers.get("Content-Length", 0))
        payload = json.loads(self.rfile.read(length))

        for alert in payload.get("alerts", []):
            labels = alert.get("labels", {})
            annotations = alert.get("annotations", {})
            title = annotations.get("summary", labels.get("alertname", "alert"))
            status = alert.get("status", "unknown").upper()
            send_telegram(f"[{status}] {title}")

        self.send_response(200)
        self.end_headers()


if __name__ == "__main__":
    HTTPServer(("0.0.0.0", PORT), Handler).serve_forever()
```

Penjelasan:

- `os.environ.get` membaca konfigurasi dari environment.
- `BaseHTTPRequestHandler` menangani POST webhook.
- `json.loads` mengubah payload JSON menjadi object Python.
- `urllib.request` mengirim request ke Telegram Bot API.
- `HTTPServer` membuka listener internal pada port 8081.



```dockerfile
FROM python:3.12-alpine
WORKDIR /app
COPY notifier.py .
ENV PYTHONUNBUFFERED=1
EXPOSE 8081
CMD ["python", "notifier.py"]
```


## 12. Prometheus Configuration

Buat `infra/prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: node-host-metrics
    static_configs:
      - targets: [node-exporter:9100]

  - job_name: docker-containers-metrics
    static_configs:
      - targets: [cadvisor:8080]

  - job_name: postgresql-metrics
    static_configs:
      - targets: [postgres-exporter:9187]

  - job_name: app-api-metrics
    metrics_path: /metrics
    static_configs:
      - targets: [api:8080]

rule_files:
  - /etc/prometheus/rules/*.yml

alerting:
  alertmanagers:
    - static_configs:
        - targets: [alertmanager:9093]
```

Penjelasan:

- `scrape_interval` menentukan interval pengambilan metrics.
- `scrape_configs` berisi daftar target.
- `job_name` menjadi label `job` pada metrics.
- `metrics_path` menetapkan endpoint metrics API.
- `rule_files` memuat file alert rule.
- `alerting` memberi tahu Prometheus alamat Alertmanager.


```yaml
groups:
  - name: toko-app-alerts
    rules:
      - alert: DiskHostHigh
        expr: (1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Disk host melebihi 85%"

      - alert: TokoDBDown
        expr: pg_up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "PostgreSQL tidak merespons"

      - alert: APIDown
        expr: up{job="app-api-metrics"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "API tidak merespons"
```



```dockerfile
FROM prom/prometheus:v2.54.0
COPY prometheus.yml /etc/prometheus/prometheus.yml
COPY rules/ /etc/prometheus/rules/
```


```bash
$ docker run --rm \
    -v "$PWD/infra/prometheus:/etc/prometheus:ro" \
    --entrypoint /bin/promtool \
    prom/prometheus:v2.54.0 \
    check config /etc/prometheus/prometheus.yml
```



## 13. Alertmanager Configuration

Buat `infra/alertmanager/alertmanager.yml`:

```yaml
route:
  receiver: telegram
  group_by: [alertname]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

receivers:
  - name: telegram
    webhook_configs:
      - url: http://telegram-notify:8081/hook
        send_resolved: true
```

Penjelasan setiap bagian:

- `route` adalah aturan routing utama alert.
- `receiver` memilih tujuan default.
- `group_by` menggabungkan alert dengan label yang sama.
- `group_wait` menunggu sebelum mengirim grup pertama.
- `group_interval` menentukan jeda antar grup baru.
- `repeat_interval` menentukan interval pengiriman ulang alert aktif.
- `receivers` mendefinisikan tujuan notifikasi.
- `webhook_configs` mengirim HTTP POST ke notifier.
- `send_resolved: true` mengirim notifikasi saat masalah sudah pulih.


```dockerfile
FROM prom/alertmanager:v0.27.0
COPY alertmanager.yml /etc/alertmanager/alertmanager.yml
```


```bash
$ docker run --rm \
    -v "$PWD/infra/alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro" \
    --entrypoint /bin/amtool \
    prom/alertmanager:v0.27.0 \
    check-config /etc/alertmanager/alertmanager.yml
```


## 14. Grafana Provisioning

Buat `infra/grafana/datasources/prometheus.yaml`:

```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    uid: prometheus
    url: http://prometheus:9090
    isDefault: true
    editable: true
```


Buat `infra/grafana/dashboards/dashboards.yaml`:

```yaml
apiVersion: 1
providers:
  - name: toko
    orgId: 1
    folder: TokoApp
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    options:
      path: /etc/grafana/provisioning/dashboards
```


Contoh panel dashboard:

```json
{
  "title": "CPU per Container",
  "type": "timeseries",
  "datasource": {
    "type": "prometheus",
    "uid": "prometheus"
  },
  "targets": [
    {
      "expr": "sum by (name) (rate(container_cpu_usage_seconds_total{name=~\".+\"}[5m]) * 100)",
      "legendFormat": "{{name}}"
    }
  ]
}
```


```dockerfile
FROM grafana/grafana:11.1.4
COPY datasources/ /etc/grafana/provisioning/datasources/
COPY dashboards/ /etc/grafana/provisioning/dashboards/
```



## 15. Docker Compose Stack

`compose.yaml` menyatukan image yang sudah dibuat. Service tidak menggunakan IP hardcode; Compose menyediakan DNS berdasarkan nama service.

```yaml
services:
  nginx:
    build: ./infra/nginx
    image: azeshion21/toko-nginx:1.0.0
    ports:
      - "80:80"
    depends_on:
      api:
        condition: service_healthy
      web:
        condition: service_started
    networks: [frontend]
    restart: unless-stopped

  web:
    build: ./web
    image: azeshion21/toko-web:1.0.0
    expose: ["3000"]
    networks: [frontend]
    restart: unless-stopped

  api:
    build: ./api
    image: azeshion21/toko-api:1.0.0
    environment:
      DATABASE_URL: ${DATABASE_URL:?isi DATABASE_URL}
    expose: ["8080"]
    networks: [frontend, backend, monitoring]
    depends_on:
      pg:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://127.0.0.1:8080/api/health || exit 1"]
      interval: 15s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  pg:
    build: ./postgres
    image: azeshion21/toko-pg:1.0.0
    environment:
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: ${DB_PASSWORD:?isi DB_PASSWORD}
      POSTGRES_DB: mydb
    volumes:
      - pgdata:/var/lib/postgresql
    networks: [backend]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d mydb"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  prometheus:
    build: ./infra/prometheus
    image: azeshion21/toko-prometheus:1.0.0
    volumes:
      - promdata:/prometheus
    ports:
      - "127.0.0.1:9090:9090"
    networks: [monitoring]
    depends_on: [api, alertmanager]
    restart: unless-stopped

  alertmanager:
    build: ./infra/alertmanager
    image: azeshion21/toko-alertmanager:1.0.0
    volumes:
      - alertdata:/alertmanager
    ports:
      - "127.0.0.1:9093:9093"
    networks: [monitoring]
    depends_on: [telegram-notify]
    restart: unless-stopped

  telegram-notify:
    build: ./telegram-notify
    image: azeshion21/telegram-notify:1.0.0
    environment:
      TELEGRAM_BOT_TOKEN: ${TELEGRAM_BOT_TOKEN:?isi token Telegram}
      TELEGRAM_CHAT_ID: ${TELEGRAM_CHAT_ID:?isi chat ID Telegram}
      PORT: "8081"
    expose: ["8081"]
    networks: [monitoring]
    restart: unless-stopped

  grafana:
    build: ./infra/grafana
    image: azeshion21/toko-grafana:1.0.0
    ports:
      - "127.0.0.1:3001:3000"
    volumes:
      - grafanadata:/var/lib/grafana
    networks: [monitoring]
    depends_on: [prometheus]
    restart: unless-stopped

  node-exporter:
    image: prom/node-exporter:<pinned-version>
    expose: ["9100"]
    networks: [monitoring]
    restart: unless-stopped

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:<pinned-version>
    expose: ["8080"]
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker:/var/lib/docker:ro
    networks: [monitoring]
    restart: unless-stopped

  postgres-exporter:
    image: prometheuscommunity/postgres-exporter:<pinned-version>
    environment:
      DATA_SOURCE_NAME: ${DATA_SOURCE_NAME:?isi datasource exporter}
    expose: ["9187"]
    networks: [backend, monitoring]
    depends_on:
      pg:
        condition: service_healthy
    restart: unless-stopped

networks:
  frontend: {}
  backend: {}
  monitoring: {}

volumes:
  pgdata: {}
  promdata: {}
  alertdata: {}
  grafanadata: {}
```

Penjelasan desain:

- `frontend` menghubungkan Nginx, web, dan API.
- `backend` hanya digunakan API, PostgreSQL, dan postgres-exporter.
- `monitoring` menghubungkan metrics target, Prometheus, Grafana, Alertmanager, dan notifier.
- `expose` hanya menyediakan port untuk network Docker; port tidak dibuka ke host.
- `ports` digunakan hanya untuk Nginx dan akses admin lokal ke Prometheus atau Grafana.
- `depends_on.condition` menunggu healthcheck, tetapi aplikasi tetap sebaiknya memiliki retry database.
- `restart: unless-stopped` membuat service kembali setelah daemon atau VPS reboot.


## 16. Build Lokal

Pastikan file dan Dockerfile sudah ada, lalu validasi Compose:

```bash
$ docker compose config
```

`docker compose config` membaca YAML, menggabungkan `.env`, memeriksa struktur, dan menampilkan konfigurasi final. Jika variable wajib kosong, command ini gagal.

Build seluruh image:

```bash
$ docker compose build --no-cache
```

Gunakan `--no-cache` untuk build bersih saat debugging. Pada build harian, hilangkan flag tersebut agar layer dependency dapat dipakai ulang.

Lihat image yang terbentuk:

```bash
$ docker image ls | grep azeshion21
$ docker image inspect azeshion21/toko-api:1.0.0
```


## 17. Validasi Sebelum Menjalankan

Validasi konfigurasi service:

```bash
$ docker compose config
$ docker run --rm --entrypoint nginx azeshion21/toko-nginx:1.0.0 -t
$ docker run --rm --entrypoint python azeshion21/telegram-notify:1.0.0 -m py_compile /app/notifier.py
$ docker run --rm --entrypoint /bin/promtool azeshion21/toko-prometheus:1.0.0 check config /etc/prometheus/prometheus.yml
$ docker run --rm --entrypoint /bin/amtool azeshion21/toko-alertmanager:1.0.0 check-config /etc/alertmanager/alertmanager.yml
```



## 18. Menjalankan Stack Lokal

```bash
$ docker compose up -d
$ docker compose ps
$ docker compose logs --tail 100 api
$ docker compose logs --tail 100 nginx
```

Uji endpoint:

```bash
$ curl -f http://127.0.0.1/api/health
$ curl -f 'http://127.0.0.1/api/products?limit=10'
$ curl -f http://127.0.0.1:9090/-/ready
$ curl -f http://127.0.0.1:3001/api/health
```

Uji DNS internal:

```bash
$ docker compose exec nginx getent hosts api web
$ docker compose exec pg pg_isready -U appuser -d mydb
$ docker compose exec prometheus wget -qO- http://api:8080/metrics | head
```



## 19. Tagging Image

Gunakan tag versi yang immutable secara operasional. `latest` boleh dipakai sebagai alias, tetapi jangan menjadi satu-satunya referensi production.

```bash
$ VERSION=1.0.0
$ docker tag tokoapp-api azeshion21/toko-api:$VERSION
$ docker tag tokoapp-web azeshion21/toko-web:$VERSION
```

Jika Compose sudah memiliki `image: azeshion21/toko-api:1.0.0`, tag akan dibuat sesuai konfigurasi tersebut.

Verifikasi digest:

```bash
$ docker image ls --digests | grep azeshion21
$ docker inspect -f '{{.RepoDigests}}' azeshion21/toko-api:1.0.0
```

Digest adalah hash manifest image. Tag dapat dipindahkan, sedangkan digest menunjuk ke konten tertentu.


## 20. Push ke Docker Hub

Login menggunakan credential Docker Hub. Jangan menulis password atau access token langsung pada command history.

```bash
$ docker login
```

Push seluruh image:

```bash
$ docker push azeshion21/toko-api:1.0.0
$ docker push azeshion21/toko-web:1.0.0
$ docker push azeshion21/toko-pg:1.0.0
$ docker push azeshion21/toko-nginx:1.0.0
$ docker push azeshion21/telegram-notify:1.0.0
$ docker push azeshion21/toko-prometheus:1.0.0
$ docker push azeshion21/toko-alertmanager:1.0.0
$ docker push azeshion21/toko-grafana:1.0.0
```



## 21. Pull dan Deploy di VPS

Di VPS, siapkan `.env` yang sesuai environment production. Jangan menyalin `.env` development secara buta.

```bash
$ git clone git@github.com:organisasi/tokoapp.git /opt/tokoapp
$ cd /opt/tokoapp
$ chmod 600 .env
$ docker login
$ docker compose pull
$ docker compose up -d
$ docker compose ps
```


```bash
$ curl -f https://contoh.com/api/health
$ docker compose logs --since 5m nginx web api
$ docker compose ps
```

Untuk rollback:

```bash
$ sed -i 's/:1.0.0/:0.9.0/g' compose.yaml
$ docker compose pull
$ docker compose up -d
```



## 22. Validasi Image Hasil Pull

Image publik TokoApp yang dianalisis menunjukkan hasil berikut:

| Image | Temuan dari image yang sudah jadi |
| --- | --- |
| `toko-grafana` | Grafana 11.1.4 dengan datasource dan dashboard yang diprovision otomatis. |
| `toko-alertmanager` | Alertmanager 0.27.0 dengan webhook ke `telegram-notify:8081/hook`. |
| `toko-prometheus` | Prometheus 2.54.0 dengan target exporter dan API. |
| `telegram-notify` | Python server pada port 8081. |
| `toko-pg` | PostgreSQL 18.6 dengan `init.sql`. |
| `toko-nginx` | Nginx 1.30.4 dengan route API dan web. |
| `toko-web` | Next.js 15.3.4 dan React 19.1.0. |
| `toko-api` | Binary Go static pada `/app/api`, port 8080. |

Perbandingan ini berguna untuk memastikan build dari source menghasilkan command, port, volume, dan endpoint yang sama seperti image yang dipublikasikan.


## 23. Security Review

Periksa hal berikut sebelum image dipublikasikan:

1. Jangan masukkan password PostgreSQL ke binary, source code, Dockerfile, atau image layer.
2. Hapus user demo atau gunakan migration khusus development.
3. Jangan commit `.env`, token Telegram, private key, atau file sertifikat.
4. Jangan membuka port PostgreSQL, Redis, Prometheus, Grafana, atau webhook notifier ke internet.
5. Tambahkan autentikasi webhook notifier atau batasi aksesnya pada network internal.
6. Jalankan API sebagai non-root.
7. Gunakan tag versi dan digest, bukan hanya `latest`.
8. Pin versi base image dan jadwalkan pembaruan dependency.
9. Scan image dengan Trivy atau scanner registry sebelum push.
10. Buat build multi-platform jika VPS ARM64 juga menjadi target.


```bash
$ grep -RInE 'password|secret|token|BEGIN .* PRIVATE KEY|postgresql://' . \
    --exclude-dir=.git \
    --exclude='.env.example'
```



## 24. Troubleshooting

```bash
# Status seluruh service
$ docker compose ps

# Log jalur request
$ docker compose logs --tail 100 nginx web api

# API tidak sehat
$ docker compose logs api
$ docker compose exec pg pg_isready -U appuser -d mydb
$ docker compose exec nginx getent hosts api

# PostgreSQL tidak memiliki tabel
$ docker compose exec pg psql -U appuser -d mydb -c '\dt'

# Prometheus target down
$ docker compose exec prometheus wget -qO- http://api:8080/metrics | head
$ curl -s http://127.0.0.1:9090/api/v1/targets | python3 -m json.tool | head -n 80

# Nginx 502
$ docker compose exec nginx getent hosts api web
$ docker compose logs --tail 50 api web nginx
```

Pola diagnosis:

- Nginx `502`: periksa DNS service, port internal, dan status API atau web.
- API gagal start: periksa `DATABASE_URL`, DNS `pg`, dan health PostgreSQL.
- PostgreSQL tidak memiliki tabel: volume mungkin sudah pernah diinisialisasi sehingga `init.sql` tidak dijalankan ulang.
- Grafana kosong: periksa datasource `http://prometheus:9090` dan status scrape target.
- Alert tidak masuk Telegram: periksa Alertmanager, notifier, token, chat ID, dan koneksi outbound.
- Prometheus target `DOWN`: periksa network monitoring dan endpoint `/metrics`.


## Fungsi Perintah

| Perintah atau konsep | Fungsi |
| --- | --- |
| `mkdir -p` | Membuat struktur direktori project beserta parent directory. |
| `cp` | Menyalin file, misalnya `.env.example` menjadi `.env`. |
| `chmod` | Mengatur permission file, misalnya membatasi `.env` menjadi mode 600. |
| `docker build` | Mengubah Dockerfile dan build context menjadi image. |
| `docker build --no-cache` | Memaksa build tanpa memakai layer cache. |
| `docker compose config` | Memvalidasi dan merender konfigurasi Compose final. |
| `docker compose up` / `down` | Membuat, menjalankan, atau menghentikan stack. |
| `docker compose build` | Membuild image yang didefinisikan pada field `build`. |
| `docker compose pull` | Mengunduh image dari registry. |
| `docker compose ps` / `logs` | Memeriksa status dan log service. |
| `docker compose exec` | Menjalankan command di container aktif. |
| `docker image inspect` | Membaca metadata image, command, port, user, dan environment. |
| `docker history` | Melihat layer pembentuk image. |
| `docker tag` | Membuat tag repository dan versi pada image. |
| `docker login` | Mengautentikasi Docker CLI ke registry. |
| `docker push` / `pull` | Mengunggah atau mengunduh image. |
| `curl` | Menguji endpoint HTTP, healthcheck, dan API registry. |
| `wget` | Menguji endpoint HTTP dari dalam container. |
| `promtool` | Memvalidasi configuration dan alert rule Prometheus. |
| `amtool` | Memvalidasi konfigurasi Alertmanager. |
| `nginx -t` | Memvalidasi sintaks konfigurasi Nginx. |
| `pg_isready` | Memeriksa kesiapan PostgreSQL menerima koneksi. |
| `psql` | Menjalankan query PostgreSQL. |
| `grep` | Mencari kemungkinan secret pada source dan konfigurasi. |
