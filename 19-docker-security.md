# 19. Docker Security

> Container bukan sandbox yang sempurna. Bab ini menjelaskan cara memperketat default Docker tanpa mengganggu fungsi aplikasi.

## Tujuan Pembelajaran

- Isolasi, non-root, capabilities, seccomp, AppArmor, read-only, no-new-privileges, limit, secret, bahaya privileged + socket

## 1. Container Isolation

Isolasi container menggunakan namespace dan cgroup, serta dapat diperkuat dengan seccomp atau AppArmor. Container bukan VM; semua container berbagi kernel host sehingga kerentanan kernel dapat berdampak luas.

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

Uji perubahan ini sejak awal karena banyak aplikasi masih perlu menulis file sementara atau cache ke `/app`.

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

Tetapkan limit berdasarkan kebutuhan realistis dan pasang alert pada 80% (Bab 25), bukan menetapkan nilai terlalu kecil hingga memicu OOM.

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

Jalur container escape yang umum melibatkan container privileged, mount root host, akses tulis ke Docker socket, eksploitasi kernel, runtime `runc` yang usang, atau konfigurasi volume yang keliru.

```bash
# Contoh berbahaya. Jangan menjalankan perintah ini pada host production:
# docker run -v /:/host -it alpine chroot /host sh  # memberi akses ke root filesystem host
$ docker version; docker info | grep -i version
$ sudo apt list --upgradable | grep -i docker
```

Mitigasi: perbarui Engine secara rutin, gunakan non-root, filesystem read-only, drop capabilities, `no-new-privileges`, AppArmor atau seccomp default, dan audit image (Bab 23/25).

## Fungsi Opsi Keamanan dan Perintah

| Opsi atau perintah | Fungsi |
| --- | --- |
| `--user` / `USER` | Menjalankan proses dengan UID dan GID tertentu, bukan sebagai root. |
| `--cap-drop` / `--cap-add` | Menghapus atau menambahkan Linux capability tertentu. Mulai dari `ALL`, lalu tambahkan hanya yang diperlukan. |
| `--security-opt seccomp=...` | Memilih profil syscall seccomp. `unconfined` menonaktifkan filter dan tidak disarankan di production. |
| `--security-opt apparmor=...` | Memilih profil AppArmor untuk container. |
| `--read-only` / `read_only` | Menjadikan root filesystem container hanya-baca. |
| `--tmpfs` | Menyediakan area tulis sementara di RAM dengan opsi seperti `noexec` dan `nosuid`. |
| `no-new-privileges` | Mencegah proses memperoleh privilege tambahan melalui setuid atau mekanisme serupa. |
| `--memory` / `--cpus` / `pids_limit` | Membatasi RAM, CPU, dan jumlah proses container. |
| `--privileged` | Memberikan akses perangkat dan capability yang sangat luas; hindari untuk aplikasi biasa. |
| `docker inspect` | Memeriksa privilege, capability, security option, dan mount container. |
| `docker info` / `docker version` | Memeriksa runtime, konfigurasi keamanan, dan versi Engine. |
| `dmesg` | Membaca penolakan AppArmor, pesan kernel, dan indikasi OOM. |
| `grep` | Mencari mount socket, capability, atau pesan keamanan dalam output. |
