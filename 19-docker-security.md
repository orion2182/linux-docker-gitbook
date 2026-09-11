# 19. Docker Security

> Container bukan sandbox ajaib. Bab ini bikin default Docker yang longgar jadi ketat tanpa bikin app mati.

## Tujuan Pembelajaran

- Isolasi, non-root, capabilities, seccomp, AppArmor, read-only, no-new-privileges, limit, secret, bahaya privileged + socket

## 1. Container Isolation

Isolasi = namespace + cgroup + (opsional) seccomp/AppArmor. Bukan VM. Kernel shared = kernel bocor = semua container kena.

```bash
$ docker info | grep -i "security\|apparmor\|seccomp"
$ docker inspect -f '{{.HostConfig.Privileged}}' web
```

Prinsip: least privilege. Nyalakan hanya yang app butuh, matikan sisanya.

## 2. Root vs Non-root

Default container jalan sebagai root (UID 0 di dalam, map ke host kecuali userns). Bocor = root host berisiko.

Dockerfile:

```dockerfile
RUN addgroup -S app && adduser -S app -G app
USER app
```

Compose:

```yaml
services:
  web:
    user: "10000:10000"
```

```bash
$ docker exec web whoami
$ docker top web
```

Uji tulis ke volume + bind port >1024 sebagai non-root sebelum prod.

## 3. Linux Capabilities

Root dipecah jadi capability kecil (`NET_BIND_SERVICE`, `CHOWN`, `SYS_ADMIN`, ...). Buang semua, tambah seperlunya.

```bash
$ docker inspect -f '{{.HostConfig.CapAdd}} {{.HostConfig.CapDrop}}' web
$ docker run -d --cap-drop ALL --cap-add NET_BIND_SERVICE --name tight nginx:alpine
$ capsh --print | head
```

`SYS_ADMIN` ~ root. Kalau app minta `--privileged` / `SYS_ADMIN`, curigai dulu, cari cara tanpa itu.

## 4. Seccomp

Filter syscall. Default Docker sudah blokir ~40 syscall berbahaya. Jangan disable tanpa alasan.

```bash
$ docker info | grep -i seccomp
$ docker run --security-opt seccomp=unconfined --rm -it alpine sh  # HINDARI di prod
```

Custom profile hanya kalau app butuh syscall langka + sudah diuji.

## 5. AppArmor

Profil MAC per container (Ubuntu sudah ada `docker-default`).

```bash
$ sudo aa-status | head -n 30
$ docker inspect -f '{{.HostConfig.SecurityOpt}}' web
$ sudo dmesg | grep -i "apparmor.*denied" | tail
```

Kalau `permission denied` misterius padahal chmod benar, cek dmesg AppArmor dulu sebelum `chmod 777`.

## 6. Read-only Filesystem

Rootfs read-only, tulis hanya ke volume/tmpfs yang jelas. Serangan yang butuh tulis biner gagal.

```yaml
services:
  web:
    read_only: true
    tmpfs: [/tmp:rw,noexec,nosuid,size=100m]
    volumes: [webcache:/var/cache/nginx]
```

```bash
$ docker exec web touch /evil || echo "benar: ditolak"
```

Test dari awal — banyak app kaget karena mau tulis log ke `/app`.

## 7. no-new-privileges

Cegah proses naik privilege via setuid binary.

```yaml
services:
  web:
    security_opt: [no-new-privileges:true]
```

```bash
$ docker inspect -f '{{.HostConfig.SecurityOpt}}' web
```

Murah, hampir tanpa efek samping. Nyalakan default di prod.

## 8. Resource Limits

Tanpa limit, 1 container bisa OOM-kan host.

```yaml
services:
  api:
    deploy:
      resources:
        limits: { cpus: "1.0", memory: 512M }
        reservations: { memory: 256M }
    pids_limit: 200
```

atau `docker run --memory 512m --cpus 1 --pids-limit 200`.

```bash
$ docker stats --no-stream
$ dmesg -T | grep -i oom | tail
```

Set limit = pagu realistis + alert 80% (Bab 25), bukan asal kecil bikin OOM sendiri.

## 9. Secrets

Jangan di ENV/image layer/Git. Pakai file + mount + gitignore + permission 600.

```yaml
services:
  db:
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_pass
    secrets: [db_pass]
secrets:
  db_pass: { file: ./secrets/db_pass.txt }
```

```bash
$ echo "kuat-$(openssl rand -hex 12)" > secrets/db_pass.txt
$ chmod 600 secrets/db_pass.txt
$ grep -r "PASSWORD" . --exclude-dir=.git | head
```

Cek bocor: `docker inspect`, `docker history`, `git log -p | grep -i pass`.

## 10. Privileged Containers

`--privileged` = matikan hampir semua proteksi (akses device host). Hampir tidak pernah dibutuhkan untuk web/db.

```bash
$ docker inspect -f '{{.HostConfig.Privileged}}' web
# harus false di prod. Kalau true, wajib ada alasan tertulis + mitigasi.
```

Alternatif: `--device`, `--cap-add` spesifik, atau pisah job privileged ke host khusus.

## 11. Docker Socket Security

Mount `/var/run/docker.sock` = root host via API. Jangan mount ke container yang terekspos publik / multi-tenant.

```bash
$ docker inspect -f '{{json .Mounts}}' ci-agent | python3 -m json.tool | grep -i sock
```

Kalau butuh (CI, Portainer): isolasi di host khusus, user terbatas, image terpercaya, audit log, pertimbangkan socket-proxy read-only.

## 12. Container Escape Concepts

Jalur kabur klasik: privileged + mount `/` host, socket write, kernel exploit, breakout via `runc` basi / misconfig volume.

```bash
# CONTOH BERBAHAYA - JANGAN di prod, pahami polanya saja:
# docker run -v /:/host -it alpine chroot /host sh  # mount root host = game over
$ docker version; docker info | grep -i version
$ sudo apt list --upgradable | grep -i docker
```

Mitigasi: update Engine rutin, non-root + ro + drop caps + no-new-priv + AppArmor/seccomp default + audit image (Bab 23/25).

## Latihan

1. Ubah 1 service jadi non-root + ro + no-new-priv + cap-drop ALL. Catat apa yang rusak, perbaiki minimal.
2. `inspect` semua container prod, pastikan `Privileged:false` dan tidak mount sock sembarang.
3. Grep repo kamu: ada secret di ENV/Git/history? Rotasi + pindah ke file.
4. Set limit mem/CPU, load test, lihat `stats` + OOM atau throttling.

## Rangkuman

- Default longgar, prod harus ketat: non-root, ro, drop caps, no-new-priv, limit, secret file.
- Privileged + socket = dua dosa besar. Hindari atau isolasi keras.
- Security tanpa ganggu fungsi = uji tiap pengerasan satu per satu.
