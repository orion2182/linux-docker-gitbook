# 04. Ubuntu Networking

> Pada Ubuntu modern, network dikelola oleh Netplan. Kesalahan indentasi YAML dapat memutus koneksi SSH. Bab ini menjelaskan cara mengubahnya dengan aman.

## Tujuan Pembelajaran

- Baca/tulis Netplan YAML dengan aman
- Set static IP, DHCP, DNS, route, hosts, hostname
- Memulihkan network saat koneksi terputus total

## 1. Netplan

File di `/etc/netplan/*.yaml`. Backend: `networkd` (server) atau `NetworkManager` (desktop).

```bash
$ ls /etc/netplan/
$ cat /etc/netplan/50-cloud-init.yaml
$ sudo netplan status
```

Alur: edit YAML → `netplan generate` → `netplan try` → `netplan apply`.

## 2. Static IP

Contoh server (`eth0` sesuaikan dengan `ip link`):

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: no
      addresses: [192.168.1.50/24]
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [1.1.1.1, 8.8.8.8]
```

Terapkan aman:

```bash
$ sudo cp /etc/netplan/*.yaml ~/netplan.bak/
$ sudo nano /etc/netplan/01-static.yaml
$ sudo netplan generate
$ sudo netplan try   # ada countdown, aman untuk remote
$ sudo netplan apply
$ ip addr; ip route show default
```

> Selalu gunakan `netplan try` saat mengubah konfigurasi melalui koneksi remote. Jangan menjalankan `apply` tanpa validasi.

## 3. DHCP

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: true
```

```bash
$ sudo netplan apply
$ ip addr show eth0
```

Di VPS cloud, IP biasanya dari DHCP + cloud-init. Jangan lawan cloud-init tanpa disable dulu.

## 4. DNS Resolver

Ubuntu pakai `systemd-resolved`. Stub di `127.0.0.53`.

```bash
$ resolvectl status
$ cat /etc/resolv.conf
$ resolvectl query github.com
```

Set DNS via Netplan (`nameservers.addresses`), bukan edit `resolv.conf` langsung (bakal ketimpa).

## 5. /etc/hosts

Override lokal, prioritas tertinggi. Bagus untuk staging / bypass DNS.

```bash
$ cat /etc/hosts
# 127.0.0.1 localhost
# 192.168.1.10  app.local  api.local
$ getent hosts app.local
```

Jangan pakai hosts sebagai pengganti DNS permanen di production.

## 6. hostnamectl

```bash
$ hostnamectl
$ sudo hostnamectl set-hostname vps-prod-01
$ cat /etc/hostname
$ cat /etc/hosts   # pastikan hostname resolve ke 127.0.0.1
```

Hostname bagus =mudah dikenali di prompt, log, dan monitoring.

## 7. Routes

Tambah route custom (misal ke VPC / VPN):

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: no
      addresses: [10.0.1.10/24]
      routes:
        - to: default
          via: 10.0.1.1
        - to: 10.99.0.0/16
          via: 10.0.1.254
```

```bash
$ ip route show
$ ip route get 10.99.5.5
```

## 8. Network Recovery

Kalau SSH putus setelah utak-atik:

1. Gunakan console VNC dari panel provider.
2. Cek file YAML: `cat /etc/netplan/*.yaml`, perhatikan indent 2 spasi.
3. Validasi: `sudo netplan generate`, `sudo netplan --debug apply`.
4. Rollback: `sudo cp ~/netplan.bak/* /etc/netplan/ && sudo netplan apply`.
5. Cek `ip addr`, `ip route`, `resolvectl status`, `ping 8.8.8.8`.
6. Jika cloud-init menimpa konfigurasi, periksa `/etc/cloud/cloud.cfg.d/` dan jalankan `sudo cloud-init status`.

Tips: sebelum mengedit server remote, siapkan job rollback menggunakan `at` atau gunakan dua sesi SSH. Uji pola ini di lab terlebih dahulu.

## Fungsi Perintah

| Perintah | Fungsi |
| --- | --- |
| `netplan status` | Menampilkan status konfigurasi jaringan yang dikelola Netplan. |
| `netplan generate` | Memvalidasi YAML dan menghasilkan konfigurasi backend tanpa menerapkannya. |
| `netplan try` | Menerapkan konfigurasi sementara dan menyediakan rollback otomatis jika tidak dikonfirmasi. |
| `netplan apply` | Menerapkan konfigurasi jaringan secara langsung. |
| `ip addr` / `ip route` | Memeriksa alamat interface dan tabel routing setelah perubahan. |
| `resolvectl` | Memeriksa resolver DNS yang sedang digunakan. |
| `cat` | Membaca file Netplan, hosts, atau resolv.conf. |
| `cp` | Menyalin file konfigurasi untuk backup atau rollback. |
| `ping` | Menguji konektivitas ke gateway atau alamat internet. |
| `hostnamectl` | Membaca atau mengubah hostname sistem. |
| `cloud-init status` | Memeriksa status inisialisasi dan kemungkinan perubahan konfigurasi oleh cloud-init. |
