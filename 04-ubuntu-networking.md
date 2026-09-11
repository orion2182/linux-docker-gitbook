# 04. Ubuntu Networking

> Di Ubuntu modern, network dikendalikan Netplan. Salah spasi YAML bisa bikin SSH putus. Bab ini cara amannya.

## Tujuan Pembelajaran

- Baca/tulis Netplan YAML dengan aman
- Set static IP, DHCP, DNS, route, hosts, hostname
- Recovery saat network mati total

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

> Selalu pakai `netplan try` saat remote. Jangan `apply` buta.

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

1. Pakai console VNC dari panel provider (jangan panik).
2. Cek file YAML: `cat /etc/netplan/*.yaml`, perhatikan indent 2 spasi.
3. Validasi: `sudo netplan generate`, `sudo netplan --debug apply`.
4. Rollback: `sudo cp ~/netplan.bak/* /etc/netplan/ && sudo netplan apply`.
5. Cek `ip addr`, `ip route`, `resolvectl status`, `ping 8.8.8.8`.
6. Kalau cloud-init menimpa, cek `/etc/cloud/cloud.cfg.d/` dan `sudo cloud-init status`.

Tips: sebelum edit remote, pasang `at` job rollback otomatis atau buka 2 sesi (satu `sleep 120 && reboot` sebagai parasut — hanya di lab!).

## Latihan

1. Backup Netplan, tampilkan `netplan status`, pahami tiap field.
2. Di VM lab: ubah DHCP → static → kembali ke DHCP pakai `netplan try`.
3. Tambah entry `/etc/hosts` palsu, buktikan dengan `getent hosts`, hapus lagi.
4. Ganti hostname, reboot, pastikan prompt + `hostnamectl` konsisten.

## Rangkuman

- Netplan = satu sumber kebenaran network Ubuntu.
- Remote wajib `try` sebelum `apply`, backup selalu.
- DNS via Netplan/systemd-resolved, bukan edit manual.
- Kuasai console provider sebelum butuh darurat.
