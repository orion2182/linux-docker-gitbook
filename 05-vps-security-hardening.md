# 05. VPS Security Hardening

> VPS baru = target empuk bot dalam hitungan menit. Bab ini baseline agar VPS tidak jadi zombie.

## Tujuan Pembelajaran

- Hardening SSH, sudo, firewall, fail2ban
- Memahami AppArmor dan unattended-upgrades
- Audit login dan susun security baseline

## 1. SSH Hardening

Edit `/etc/ssh/sshd_config` (backup dulu!):

```bash
$ sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
# Isi yang disarankan:
# Port 2222 (opsional, bukan pengganti firewall)
# PermitRootLogin no
# PasswordAuthentication no
# PubkeyAuthentication yes
# MaxAuthTries 3
# LoginGraceTime 30
$ sudo sshd -t && sudo systemctl reload ssh
$ sudo ss -tlnp | grep ssh
```

Uji di sesi baru sebelum tutup sesi lama. Jangan kunci diri sendiri.

## 2. SSH Keys

Di laptop:

```bash
$ ssh-keygen -t ed25519 -C "laptop-utama"
$ ssh-copy-id -i ~/.ssh/id_ed25519.pub -p 22 user@IP-VPS
$ ssh -i ~/.ssh/id_ed25519 user@IP-VPS
```

Di server, pastikan permission ketat:

```bash
$ chmod 700 ~/.ssh
$ chmod 600 ~/.ssh/authorized_keys
$ cat ~/.ssh/authorized_keys
```

## 3. Disable Root SSH

```bash
$ grep -i permitroot /etc/ssh/sshd_config
# PermitRootLogin no
$ sudo sshd -t && sudo systemctl reload ssh
$ sudo grep -i "failed\|invalid" /var/log/auth.log | tail
```

Buat user sudo dulu sebelum disable root. Kalau belum ada, jangan lanjut.

## 4. sudo

```bash
$ sudo adduser deploy
$ sudo usermod -aG sudo deploy
$ sudo visudo   # validasi sintaks, jangan nano langsung /etc/sudoers
$ sudo -l -U deploy
```

Pola aman: user harian tanpa password-less sudo kecuali untuk service account automation yang dibatasi.

## 5. UFW

Frontend gampang untuk iptables.

```bash
$ sudo ufw default deny incoming
$ sudo ufw default allow outgoing
$ sudo ufw allow 22/tcp
$ sudo ufw allow 80,443/tcp
$ sudo ufw enable
$ sudo ufw status verbose numbered
$ sudo ufw delete 3   # hapus by number, hati-hati
```

Urutan: allow SSH dulu, baru enable. Cek dari sesi kedua.

## 6. nftables / iptables concepts

UFW di belakang pakai iptables/nftables. Paham rantai dasar:

- INPUT (masuk), OUTPUT (keluar), FORWARD (lewat, untuk Docker/router)
- Policy default + rules berurutan, first-match wins.

```bash
$ sudo iptables -L -n -v --line-numbers | head -n 40
$ sudo nft list ruleset | head -n 80
```

> Jangan `iptables -F` di VPS remote + Docker — bisa putus koneksi dan rusak chain Docker.

## 7. Fail2ban

Blokir brute-force otomatis.

```bash
$ sudo apt install -y fail2ban
$ sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
$ sudo nano /etc/fail2ban/jail.local
# [sshd]
# enabled = true
# maxretry = 5
# bantime = 1h
$ sudo systemctl enable --now fail2ban
$ sudo fail2ban-client status sshd
$ sudo fail2ban-client set sshd unbanip 1.2.3.4
```

## 8. AppArmor

MAC bawaan Ubuntu. Batasi apa yang boleh dilakukan program walau sudah root.

```bash
$ sudo aa-status
$ sudo aa-enforce /etc/apparmor.d/usr.sbin.nginx
$ sudo aa-complain /etc/apparmor.d/usr.sbin.nginx
$ sudo dmesg | grep -i apparmor | tail
```

Kalau aplikasi aneh permission denied padahal `chmod` benar, curigai AppArmor.

## 9. Unattended Upgrades

Update security otomatis, reboot manual terjadwal.

```bash
$ sudo apt install -y unattended-upgrades
$ sudo dpkg-reconfigure -plow unattended-upgrades
$ cat /etc/apt/apt.conf.d/50unattended-upgrades
$ cat /var/log/unattended-upgrades/unattended-upgrades.log | tail
```

Untuk production: auto security saja, bukan semua upgrade.

## 10. Login Auditing

```bash
$ last -n 20
$ lastb -n 20
$ who; w
$ sudo grep -i "accepted\|failed" /var/log/auth.log | tail -n 30
$ sudo journalctl -u ssh --since "7 days ago" | grep -i fail | tail
```

Pasang alert kalau login dari IP asing (dibahas monitoring Bab 25).

## 11. Security Baseline

Checklist untuk setiap VPS baru:

1. User non-root + SSH key + `PermitRootLogin no` + `PasswordAuthentication no`
2. UFW deny incoming, allow hanya 22/80/443
3. Fail2ban untuk sshd
4. Unattended-upgrades security-only aktif
5. AppArmor enforcing
6. `last`, `auth.log`, dan backup config `/etc/ssh/sshd_config`, `/etc/ufw/*`
7. Snapshot awal sebelum install Docker/app

Simpan baseline sebagai skrip atau repository Git (Bab 09) agar dapat diterapkan secara konsisten.

## Fungsi Perintah

| Perintah | Fungsi |
| --- | --- |
| `ssh-keygen` | Membuat pasangan kunci SSH publik dan privat. |
| `ssh-copy-id` | Menyalin kunci publik ke `authorized_keys` pada server. |
| `sshd -t` / `sshd -T` | Memvalidasi atau menampilkan konfigurasi SSH yang efektif. |
| `systemctl reload` | Memuat ulang konfigurasi service tanpa menghentikan proses utama jika service mendukungnya. |
| `visudo` | Mengedit dan memvalidasi file sudoers dengan aman. |
| `usermod -aG` | Menambahkan user ke grup tanpa menghapus keanggotaan grup lainnya. |
| `ufw` | Mengelola firewall host dengan sintaks yang lebih sederhana. |
| `iptables` / `nft` | Memeriksa aturan firewall tingkat rendah yang digunakan sistem atau Docker. |
| `fail2ban-client` | Melihat status jail, melakukan ban, atau membuka ban alamat IP. |
| `aa-status` / `aa-enforce` | Memeriksa status AppArmor dan mengaktifkan mode enforcement pada profil. |
| `last` / `lastb` | Menampilkan login berhasil dan login gagal yang tercatat. |
| `journalctl` | Membaca log service dan aktivitas autentikasi dari journald. |
