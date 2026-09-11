# 20. Rootless Docker

> Docker tanpa daemon root = bocor container tidak langsung root host. Harga: setup user-namespace + limitasi network/port.

## Tujuan Pembelajaran

- Arsitektur rootless, user namespaces, subuid/subgid, RootlessKit, systemd user, network/storage, troubleshooting

## 1. Rootless Architecture

Daemon jalan sebagai user biasa (bukan root). Container root (UID 0 di dalam) = map ke UID user di host (misal 1000), bukan 0.

```bash
$ dockerd --version
$ rootlesskit --version 2>&1 | head
$ ps aux | grep -E "rootless|dockerd" | grep -v grep
```

Cocok untuk shared VPS / CI multi-user / compliance ketat. Tidak cocok kalau butuh banyak low-port + overlay complex tanpa tuning.

## 2. User Namespaces

Map UID/GID dalam container → luar. `root:0` di dalam bisa = `100000` di luar.

```bash
$ cat /proc/self/uid_map
$ cat /proc/self/gid_map
```

Ini yang bikin escape tidak langsung dapat root host.

## 3. subuid / subgid

Range ID yang boleh dipakai user untuk mapping. Wajib ada sebelum install rootless.

```bash
$ grep $USER /etc/subuid /etc/subgid
# contoh: ubuntu:100000:65536
$ sudo usermod --add-subuids 100000-165535 --add-subgids 100000-165535 $USER
$ grep $USER /etc/subuid /etc/subgid
```

Tanpa ini, rootless gagal start dengan error userns.

## 4. RootlessKit

Helper network + process agar container bisa jalan tanpa root (slirp, port driver).

```bash
$ rootlesskit --help 2>&1 | head -n 20
$ dockerd-rootless.sh --help 2>&1 | head -n 20
```

Kamu jarang panggil langsung — dipanggil oleh skrip rootless Docker.

Install cepat (user biasa, bukan root):

```bash
$ curl -fsSL https://get.docker.com/rootless | sh
$ echo 'export PATH=$HOME/bin:$PATH' >> ~/.bashrc
$ echo 'export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock' >> ~/.bashrc
$ source ~/.bashrc
$ docker version
```

## 5. systemd User Services

Agar daemon nyala otomatis + survive logout (lingering).

```bash
$ loginctl enable-linger $USER
$ systemctl --user enable --now docker
$ systemctl --user status docker --no-pager | head -n 20
$ journalctl --user -u docker --since "1 hour ago" | tail -n 30
$ echo $XDG_RUNTIME_DIR
$ ls -l $XDG_RUNTIME_DIR/docker.sock
```

Tanpa lingering, Docker mati saat SSH logout. Ini jebakan paling sering.

## 6. Networking

Default rootless pakai slirp (userspace NAT). Lambat dibanding bridge rootful + ada batasan ping/ICMP dan port <1024.

```bash
$ docker run --rm alpine ping -c1 8.8.8.8 || echo "slirp bisa blokir ping, coba wget"
$ docker run --rm alpine wget -qO- http://example.com | head
$ docker run -d -p 8080:80 nginx:alpine
$ curl -I http://127.0.0.1:8080
```

Low port (<1024) butuh `net.ipv4.ip_unprivileged_port_start=0` atau pakai port tinggi + reverse proxy (Bab 22). Untuk performa, ada opsi `bypass4netns` (advanced, uji dulu).

## 7. Storage

Root dir pindah ke home: `~/.local/share/docker`. Driver `overlay2` (atau `fuse-overlayfs` di kernel tua). Volume tetap jalan, bind perlu perhatikan UID map.

```bash
$ docker info --format '{{.DockerRootDir}} {{.StorageDriver}}'
$ df -h ~/.local/share/docker
$ docker volume create testvol
$ docker run --rm -v testvol:/data alpine sh -c "echo hi > /data/f && cat /data/f"
```

Backup: backup home + volume (path beda dari rootful!). Jangan samakan skrip backup rootful mentah-mentah.

## 8. Troubleshooting

```bash
$ docker version
$ echo $DOCKER_HOST
$ ls -l $XDG_RUNTIME_DIR/docker.sock
$ systemctl --user status docker
$ journalctl --user -u docker -xe | tail -n 50
$ cat /etc/subuid; cat /etc/subgid
$ loginctl show-user $USER | grep -i linger
$ sysctl net.ipv4.ip_unprivileged_port_start
```

Kasus umum:
- `cannot connect to daemon` → `DOCKER_HOST` salah / service user mati / logout tanpa linger.
- `no subuid` → tambah subuid/subgid lalu reinstall user setup.
- Port 80 gagal bind → pakai 8080 + Nginx di depan, atau turunkan `ip_unprivileged_port_start`.
- `ping` gagal tapi `wget` jalan → limitasi slirp, bukan internet mati.
- Migrasi rootful → rootless: image harus pull/build ulang, volume pindah manual via backup/restore (Bab 15).

## Latihan

1. Install rootless di VM lab (user baru), enable linger, reboot, buktikan tetap jalan.
2. Jalankan nginx di 8080, curl dari host. Coba port 80, catat error.
3. Bandingkan `docker info` rootful vs rootless (root dir, driver).
4. Migrasi 1 volume kecil rootful → rootless via tar backup/restore.

## Rangkuman

- Rootless = kurangi blast radius dengan harga network/storage beda.
- Kunci: subuid, linger, DOCKER_HOST, port tinggi + proxy.
- Pilih rootless untuk multi-user/ketat, rootful untuk performa/simplicity single-tenant + hardening Bab 19.
