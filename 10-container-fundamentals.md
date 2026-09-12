# 10. Container Fundamentals

> Sebelum hafal `docker run`, paham dulu kenapa container beda dari VM. Ini fondasi semua bab Docker.

## Tujuan Pembelajaran

- Menjelaskan beda container vs VM dengan tepat
- Menjelaskan namespaces, cgroups, OCI, containerd, runc
- Menggambarkan arsitektur Docker dan lifecycle container

## 1. Containers vs VM

VM = virtualisasi hardware (tiap VM bawa kernel + OS sendiri, berat, boot menit). Container = virtualisasi OS (bagi kernel host, isolasi proses, ringan, start detik).

| Aspek | VM | Container |
|---|---|---|
| Isolasi | Kuat (hypervisor) | Cukup (namespace+cgroup) |
| Ukuran | GB | MB |
| Start | Menit | Detik |
| Density | Puluhan | Ratusan |

Pakai VM untuk isolasi kernel / OS beda. Pakai container untuk app portable yang skalabel.

## 2. Linux Namespaces

Isolasi "apa yang bisa dilihat" proses: PID, NET, MNT, UTS, IPC, USER.

```bash
$ lsns -t pid,net,mnt,uts
$ sudo lsns -p 1
$ unshare --help | head -n 20
```

Container = sekumpulan namespace yang digabung. Beda namespace network = beda IP/interface walau 1 host.

## 3. cgroups

Batasi "berapa banyak" resource bisa dipakai: CPU, memory, I/O, pids.

```bash
$ cat /proc/self/cgroup
$ systemd-cgls --no-pager | head -n 40
$ cat /sys/fs/cgroup/system.slice/cpu.max 2>/dev/null || cat /sys/fs/cgroup/cpu/cpu.shares
```

Tanpa batas cgroup, satu container yang mengalami kebocoran resource dapat menghabiskan seluruh RAM host (dibahas pada Bab 19/21).

## 4. OCI

Open Container Initiative = standar agar image/runtime bisa tukar (Docker, Podman, containerd cocok).

- **image-spec:** format image + layer + manifest.
- **runtime-spec:** cara jalankan bundle (config.json + rootfs).
- **distribution-spec:** cara push/pull registry.

Praktisnya: image yang kamu build bisa jalan di runtime lain yang OCI-compliant.

## 5. containerd

Runtime high-level: pull image, kelola snapshot, panggil runc, kelola lifecycle. Dipakai Docker dan Kubernetes.

```bash
$ sudo ctr version 2>&1 | head
$ sudo ctr images list 2>&1 | head
```

Kamu jarang panggil langsung, tapi wajib tahu dia ada di bawah Docker.

## 6. runc

Runtime low-level OCI: terima bundle → buat namespaces/cgroups → `exec` proses init container. 1 proses runc per container (pendek umur).

```bash
$ runc --version
$ sudo runc list 2>&1 | head
```

## 7. Docker Architecture

Alur: `docker CLI` → REST ke `dockerd` via socket → `containerd` → `runc` → container jalan.

```bash
$ docker version
$ docker info | head -n 40
$ ls -l /var/run/docker.sock
```

Komponen: Engine (dockerd), CLI, containerd, runc, buildkit (build), network/storage driver.

## 8. Container Lifecycle

`create → start → running → stop → start lagi → rm`. Hapus ≠ stop. Image ≠ container.

```bash
$ docker run -d --name demo nginx:alpine
$ docker ps
$ docker stop demo
$ docker start demo
$ docker rm -f demo
$ docker ps -a
```

Data pada writable layer akan hilang saat container dihapus dengan `rm`, kecuali data tersebut disimpan pada volume (Bab 15).

## Fungsi Perintah

| Perintah | Fungsi |
| --- | --- |
| `lsns` | Menampilkan namespace Linux yang sedang digunakan oleh proses. |
| `unshare` | Menjalankan proses dengan namespace baru untuk eksperimen isolasi. |
| `cat /proc/...` | Membaca informasi kernel, termasuk pemetaan cgroup atau UID. |
| `systemd-cgls` | Menampilkan hierarki control group systemd. |
| `ctr` | CLI tingkat rendah untuk containerd; gunakan hanya jika memahami namespace dan objek yang dikelola. |
| `runc` | Menjalankan dan memeriksa container sesuai OCI pada level runtime rendah. |
| `docker version` | Menampilkan versi client dan server Docker. |
| `docker info` | Menampilkan konfigurasi daemon, storage driver, runtime, dan peringatan. |
| `docker run` | Membuat dan menjalankan container baru dari image. |
| `docker ps` | Menampilkan container yang sedang berjalan; `-a` juga menampilkan container yang berhenti. |
| `docker stop` / `start` | Menghentikan atau menjalankan kembali container yang sudah ada. |
| `docker rm` | Menghapus container. Opsi `-f` juga menghentikan container terlebih dahulu. |
