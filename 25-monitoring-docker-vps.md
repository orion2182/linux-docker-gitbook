# 25. Monitoring Docker & VPS

> Service yang tidak dimonitor dapat berhenti tanpa diketahui. Bab ini membahas observabilitas mulai dari `docker stats` hingga Prometheus, Grafana, dan alerting.

## Tujuan Pembelajaran

- stats/events/logs/healthcheck + Node Exporter/cAdvisor/Prometheus/Grafana/alerting

## 1. docker stats

Snapshot cepat CPU/mem/net/IO per container.

```bash
$ docker stats --no-stream
$ docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemPerc}}\t{{.NetIO}}"
$ watch -n 2 docker stats --no-stream
```

Untuk investigasi sesaat, bukan tren jangka panjang. Tren butuh Prometheus (lihat bawah).

## 2. docker events

Kejadian: die, oom, kill, destroy, health_status.

```bash
$ docker events --since 30m | head -n 50
$ docker events -f 'event=die' --since 24h
$ docker events -f 'container=api' &
```

Pasangan wajib `docker logs` + `inspect` saat container restart misterius.

## 3. Container Logs

stdout = sumber kebenaran. Batasi + sentralisasi.

```bash
$ docker compose logs -f --tail 100 api
$ docker logs --since 15m web 2>&1 | grep -i "error\|exception" | tail -n 30
$ docker inspect -f '{{.HostConfig.LogConfig}}' api
```

Compose prod wajib `logging: max-size/max-file` (Bab 21). Log JSON besar tanpa rotasi = disk penuh (Bab 02/06).

## 4. Healthchecks

Health = sinyal untuk manusia + orchestrator + LB.

```bash
$ docker compose ps --format "table {{.Name}}\t{{.Status}}"
$ docker inspect -f '{{.State.Health.Status}} {{.State.Health.FailingStreak}}' api
$ docker inspect -f '{{json .State.Health.Log}}' api | python3 -m json.tool | tail -n 40
```

Endpoint `/health` harus cek DB/Redis beneran, bukan cuma `return ok`. Bedakan `/live` vs `/ready` untuk start lambat.

## 5. Node Exporter

Eksportir metrik host (CPU/mem/disk/net/systemd) untuk Prometheus.

```yaml
services:
  node-exporter:
    image: prom/node-exporter:latest
    ports: ["127.0.0.1:9100:9100"]
    volumes: ["/proc:/host/proc:ro", "/sys:/host/sys:ro", "/:/rootfs:ro"]
    command: ["--path.procfs=/host/proc", "--path.sysfs=/host/sys", "--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($|/)"]
    restart: unless-stopped
    networks: [monitoring]
```

```bash
$ curl -s http://127.0.0.1:9100/metrics | grep -E "^node_load1|^node_memory_MemAvailable" | head
```

Bind ke `127.0.0.1` + scrape via Prometheus internal, jangan ekspos publik.

## 6. cAdvisor

Metrik per container (CPU/mem/FS/net + label Docker).

```yaml
services:
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    ports: ["127.0.0.1:8082:8080"]
    volumes: ["/:/rootfs:ro", "/var/run:/var/run:ro", "/sys:/sys:ro", "/var/lib/docker/:/var/lib/docker:ro"]
    restart: unless-stopped
    networks: [monitoring]
```

```bash
$ curl -s http://127.0.0.1:8082/metrics | grep -i "container_memory" | head
```

Alternatif ringan: `docker stats` + exporter kecil kalau VPS 1 GB keberatan.

## 7. Prometheus

TSDB + scrape + PromQL + alert rule. Retensi sesuaikan disk (misal 15 hari di VPS kecil).

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    ports: ["127.0.0.1:9090:9090"]
    volumes: ["./prometheus.yml:/etc/prometheus/prometheus.yml:ro", "promdata:/prometheus"]
    networks: [monitoring]
    restart: unless-stopped
volumes:
  promdata: {}
```

```yaml
# prometheus.yml
global: { scrape_interval: 15s }
scrape_configs:
  - job_name: node
    static_configs: [{ targets: ["node-exporter:9100"] }]
  - job_name: cadvisor
    static_configs: [{ targets: ["cadvisor:8080"] }]
```

```bash
$ curl -s "http://127.0.0.1:9090/api/v1/query?query=up" | python3 -m json.tool | head -n 40
```

Query awal: `up`, `node_load1`, `container_memory_usage_bytes{name="api"}`.

## 8. Grafana

Dashboard visual di atas Prometheus.

```yaml
services:
  grafana:
    image: grafana/grafana:latest
    ports: ["127.0.0.1:3001:3000"]
    volumes: [grafanadata:/var/lib/grafana]
    networks: [monitoring]
    restart: unless-stopped
volumes:
  grafanadata: {}
```

Langkah: login admin → Add Prometheus (`http://prometheus:9090`) → Import dashboard Node Exporter (1860) + Docker (193) → simpan. Jangan ekspos Grafana publik tanpa auth + TLS (lewat proxy Bab 22).

## 9. Alerting

Alert = aksi, bukan spam. Mulai dari 5: down, disk, mem, CPU, health gagal.

Contoh `alerts.yml` Prometheus:

```yaml
groups:
  - name: vps
    rules:
      - alert: InstanceDown
        expr: up == 0
        for: 2m
      - alert: DiskPenuh
        expr: (1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) > 0.85
        for: 5m
      - alert: MemKritis
        expr: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) > 0.9
        for: 5m
```

Kirim ke Telegram/Email via Alertmanager/webhook. Tiap alert wajib ada runbook: "kalau bunyi, buka Bab 26 bagian X".

```bash
$ docker compose logs prometheus --tail 30
$ curl -s http://127.0.0.1:9090/api/v1/rules | python3 -m json.tool | head -n 60
```

## Fungsi Perintah dan Komponen

| Perintah atau komponen | Fungsi |
| --- | --- |
| `docker stats` | Menampilkan penggunaan resource container secara realtime atau satu kali dengan `--no-stream`. |
| `docker events` | Menampilkan event lifecycle seperti start, stop, die, dan OOM. |
| `docker logs` / `docker compose logs` | Membaca log aplikasi atau service untuk korelasi dengan metrik. |
| `healthcheck` | Menentukan apakah aplikasi siap melayani request. |
| `node-exporter` | Mengekspos metrik host Linux untuk Prometheus. |
| `cAdvisor` | Mengekspos metrik penggunaan resource per container. |
| `Prometheus` | Menyimpan time series, melakukan scrape endpoint metrics, dan mengevaluasi PromQL. |
| `Grafana` | Menampilkan dashboard dan visualisasi dari Prometheus atau data source lain. |
| `Alertmanager` | Mengelompokkan, merutekan, dan mengirim alert ke Telegram, email, atau webhook. |
| `curl` | Menguji endpoint metrics dan API Prometheus. |
| `python3 -m json.tool` | Memformat respons JSON agar mudah dibaca. |
| `up` | Metric Prometheus yang menunjukkan apakah target scrape dapat dijangkau. |
| `node_load1` | Metric load average host untuk interval satu menit. |
| `docker compose logs --tail` | Mengambil sejumlah baris log terakhir untuk diagnosis cepat. |
