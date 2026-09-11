# 11. Docker Installation & Architecture

> Install Docker yang benar = dari repo resmi + paham tiap komponen. Jangan dari `apt install docker` sembarangan (nama paket bisa ketuker).

## Tujuan Pembelajaran

- Install Docker Engine resmi di Ubuntu
- Menjelaskan CLI, dockerd, containerd, socket, context, info, config

## 1. Docker Engine

Engine = dockerd + containerd + runc + network + storage driver. Install resmi:

```bash
$ sudo apt update
$ sudo apt install -y ca-certificates curl gnupg
$ sudo install -m 0755 -d /etc/apt/keyrings
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
$ echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" | sudo tee /etc/apt/sources.list.d/docker.list
$ sudo apt update
$ sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
$ sudo systemctl enable --now docker
$ sudo docker run --rm hello-world
```

Verifikasi: `hello-world` jalan = Engine sehat.

## 2. Docker CLI

Client yang ngomong ke daemon via REST/socket. Bisa remote via context/SSH.

```bash
$ docker version --format '{{.Client.Version}} vs {{.Server.Version}}'
$ docker --help | head -n 30
$ docker system df
```

CLI ≠ daemon. CLI error belum tentu daemon mati — cek `systemctl status docker`.

## 3. dockerd

Daemon: terima API, jadwalkan container, kelola network/volume/image.

```bash
$ systemctl status docker --no-pager | head -n 20
$ sudo journalctl -u docker --since "1 hour ago" | tail -n 50
$ ps aux | grep -E "dockerd|containerd" | grep -v grep
```

Config: `/etc/docker/daemon.json` (jangan edit sembarang, validasi JSON).

## 4. containerd

Sudah dibahas di Bab 10, di sini fokus operasional:

```bash
$ systemctl status containerd --no-pager | head -n 15
$ containerd --version
$ sudo ctr namespaces list 2>&1 | head
```

Docker pakai namespace `moby`. Jangan hapus snapshot via `ctr` tanpa paham — pakai `docker` saja.

## 5. Docker Socket

File socket Unix default `/var/run/docker.sock`. Siapa bisa tulis = root setara (bahaya!).

```bash
$ ls -l /var/run/docker.sock
$ sudo usermod -aG docker $USER
$ newgrp docker
$ docker ps
```

> Menambah user ke grup `docker` = kasih root. Hanya untuk user tepercaya / dev. Di prod pertimbangkan rootless (Bab 20).

## 6. Docker Context

Pindah target daemon tanpa SSH manual tiap kali.

```bash
$ docker context ls
$ docker context create vps --docker "host=ssh://user@IP-VPS"
$ docker context use vps
$ docker ps
$ docker context use default
```

Bagus untuk kelola 1 laptop → banyak VPS. Butuh SSH key sudah terpasang (Bab 05).

## 7. Docker Info

Fakta sistem Docker: versi, driver, root dir, registry, warning.

```bash
$ docker info
$ docker info --format '{{.ServerVersion}} {{.StorageDriver}} {{.DockerRootDir}}'
$ docker version
```

Simpan output `docker info` saat lapor error / minta bantuan.

## 8. Docker Configuration

File kunci:

- `/etc/docker/daemon.json` — config daemon (log driver, bip, registry mirror).
- `~/.docker/config.json` — auth + preferensi CLI.

Contoh aman (batasi log!):

```json
{
  "log-driver": "json-file",
  "log-opts": { "max-size": "10m", "max-file": "3" },
  "storage-driver": "overlay2"
}
```

```bash
$ cat /etc/docker/daemon.json
$ sudo systemctl restart docker
$ docker info | grep -i "logging\|storage"
```

Selalu `cat` + validasi JSON (`python3 -m json.tool`) sebelum restart.

## Latihan

1. Install resmi di VM lab, jalankan `hello-world` + `nginx:alpine`.
2. `docker info`, catat storage driver dan root dir.
3. Buat context ke VPS lab via SSH, `ps` dari laptop.
4. Set `max-size/max-file`, restart, buktikan log terbatasi.

## Rangkuman

- Install dari repo Docker resmi, bukan paket `docker` random.
- Paham CLI vs dockerd vs containerd vs socket.
- Socket = privilegi root, context = remote aman, info/config = alat diagnosa.
