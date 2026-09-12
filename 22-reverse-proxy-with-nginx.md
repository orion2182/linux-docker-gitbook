# 22. Reverse Proxy with Nginx

> Nginx = pintu depan: terima 80/443, teruskan ke container, urus TLS. Salah header = app bingung IP / redirect loop.

## Tujuan Pembelajaran

- HTTP/HTTPS, arsitektur Nginx, server block, reverse proxy, header, domain→VPS, Docker→Nginx, SSL/TLS, LE, 502/504

## 1. HTTP / HTTPS

HTTP port 80 polos, HTTPS 443 terenkripsi TLS. Prod wajib HTTPS (SEO, auth, cookie aman).

```bash
$ curl -I http://contoh.com
$ curl -I https://contoh.com
$ curl -v https://contoh.com 2>&1 | grep -i "subject\|issuer\|expire" | head
```

Redirect permanen 80→443 di Nginx, bukan di app.

## 2. Nginx Architecture

Master (baca config, manage) + worker (handle koneksi, event-driven, hemat RAM).

```bash
$ nginx -v
$ ps aux | grep nginx | grep -v grep
$ nginx -T 2>&1 | head -n 80
$ cat /etc/nginx/nginx.conf | grep -E "worker_|events|http" | head -n 20
```

1 Nginx di host / container proxy cukup untuk VPS kecil-menengah. Jangan 2 layer proxy tanpa alasan.

## 3. Server Block

Virtual host per domain.

```nginx
server {
  listen 80;
  server_name contoh.com www.contoh.com;
  return 301 https://$host$request_uri;
}
server {
  listen 443 ssl;
  server_name contoh.com;
  # ssl_* diisi certbot / manual (lihat TLS)
  location / {
    proxy_pass http://127.0.0.1:3000;
  }
}
```

```bash
$ sudo nginx -t
$ sudo systemctl reload nginx
```

1 file per domain di `sites-enabled/`, test tiap ubah.

## 4. Reverse Proxy

Teruskan + jaga timeout + buffer wajar.

```nginx
location / {
  proxy_pass http://127.0.0.1:3000;
  proxy_http_version 1.1;
  proxy_set_header Host $host;
  proxy_set_header X-Real-IP $remote_addr;
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_set_header X-Forwarded-Proto $scheme;
  proxy_connect_timeout 5s;
  proxy_read_timeout 60s;
  proxy_send_timeout 60s;
}
```

```bash
$ curl -I http://127.0.0.1:3000/health
$ curl -I https://contoh.com/
```

`proxy_pass` tanpa `/` akhir vs dengan `/` beda perilaku path — uji eksplisit.

## 5. Proxy Headers

App butuh IP asli + skema untuk log/rate-limit/redirect benar.

- `Host` = domain asli
- `X-Real-IP` / `X-Forwarded-For` = IP client
- `X-Forwarded-Proto` = http/https

Di app (Express contoh): `app.set('trust proxy', 1)`. Tanpa ini, app kira semua dari `127.0.0.1` + redirect loop http↔https.

## 6. Domain → VPS

1. DNS A/AAAA ke IP VPS (tanpa proxy CDN dulu untuk debug).
2. Tunggu propagasi, verifikasi dari luar.
3. Baru pasang TLS.

```bash
$ dig +short contoh.com
$ dig +short www.contoh.com
$ ping -c2 contoh.com
$ curl -I http://contoh.com
```

TTL kecil (300) saat migrasi, besarkan setelah stabil.

## 7. Docker → Nginx

Dua pola:

**A. Nginx di host, app di Docker (simple, disarankan mulai):**

```nginx
upstream app { server 127.0.0.1:3000; }
server {
  listen 80; server_name contoh.com;
  location / { proxy_pass http://app; }
}
```

Compose publish `127.0.0.1:3000:3000` saja.

**B. Nginx juga di Docker (1 stack Compose):**

```yaml
services:
  proxy:
    image: nginx:alpine
    ports: ["80:80", "443:443"]
    volumes: ["./proxy.conf:/etc/nginx/conf.d/default.conf:ro"]
  web:
    expose: ["3000"]
```

`proxy_pass http://web:3000;` (nama service, bukan IP). Pilih 1 pola, jangan campur 2 proxy berlapis tanpa alasan.

## 8. SSL/TLS

TLS menyediakan enkripsi dan autentikasi server melalui sertifikat. Gunakan TLS 1.2 atau lebih baru; TLS 1.0 dan 1.1 sebaiknya dinonaktifkan.

```bash
$ openssl s_client -connect contoh.com:443 -servername contoh.com </dev/null 2>/dev/null | openssl x509 -noout -dates -subject -issuer
$ curl --tlsv1.2 -I https://contoh.com
```

Cipher default Ubuntu + certbot sudah cukup untuk umum. HSTS aktifkan setelah yakin HTTPS stabil.

## 9. Let's Encrypt

Gratis + auto-renew. Pakai plugin nginx (host) atau webroot/dns (docker).

Host:

```bash
$ sudo apt install -y certbot python3-certbot-nginx
$ sudo certbot --nginx -d contoh.com -d www.contoh.com
$ sudo certbot renew --dry-run
$ systemctl status certbot.timer
```

Docker (webroot):

```bash
$ docker compose exec proxy nginx -T | head
# pastikan /.well-known/acme-challenge/ dilayani dari volume webroot
$ sudo certbot certonly --webroot -w /var/www/certbot -d contoh.com
```

Jangan lupa mount `certs:/etc/nginx/certs:ro` + reload setelah renew (hook `--deploy-hook "docker exec proxy nginx -s reload"`).

## 10. Troubleshooting 502/504

- **502 Bad Gateway** = Nginx tidak dapat terhubung ke upstream (aplikasi mati, port salah, network berbeda, atau proses crash saat start).
- **504 Gateway Timeout** = upstream merespons terlalu lama (query berat, timeout terlalu kecil, atau worker habis).

```bash
$ sudo tail -n 50 /var/log/nginx/error.log
$ sudo nginx -T | grep -A10 "server_name contoh"
$ curl -v http://127.0.0.1:3000/health
$ docker compose ps; docker compose logs --tail 50 web
$ ss -tlnp | grep 3000
$ curl -w "code:%{http_code} time:%{time_total}s\n" -o /dev/null -s https://contoh.com/lambat
```

Naikkan `proxy_read_timeout` hanya setelah memastikan masalahnya bukan query N+1 atau deadlock. Timeout besar tanpa perbaikan akar masalah dapat memperpanjang antrean.

## Fungsi Perintah dan Directive

| Perintah atau directive | Fungsi |
| --- | --- |
| `nginx -t` | Memvalidasi sintaks konfigurasi sebelum reload atau restart. |
| `nginx -T` | Menampilkan konfigurasi gabungan yang sedang digunakan Nginx. |
| `systemctl reload nginx` | Memuat ulang konfigurasi tanpa memutus koneksi aktif jika memungkinkan. |
| `server` / `server_name` | Mendefinisikan virtual host dan domain yang dilayani. |
| `listen` | Menentukan alamat dan port tempat Nginx menerima koneksi. |
| `location` | Menentukan aturan berdasarkan path request. |
| `proxy_pass` | Meneruskan request ke upstream atau aplikasi backend. |
| `proxy_set_header` | Mengirim header seperti Host, IP client, dan skema HTTP ke backend. |
| `curl -I` / `curl -v` | Menguji header atau menampilkan detail koneksi HTTP/TLS. |
| `dig` | Memeriksa record DNS domain. |
| `openssl s_client` | Memeriksa sertifikat dan negosiasi TLS secara langsung. |
| `certbot` | Meminta, memeriksa, dan memperbarui sertifikat Let's Encrypt. |
| `ss` | Memastikan port upstream sedang listen. |
| `docker compose logs` | Membaca log aplikasi di belakang Nginx. |
