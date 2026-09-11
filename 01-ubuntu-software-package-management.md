# 01. Ubuntu Software & Package Management

> Fondasi wajib sebelum pegang VPS: paham dari mana software datang, cara install/update yang aman, dan cara benerin kalau dependency rusak.

## Tujuan Pembelajaran

Setelah bab ini kamu bisa:
- Menjelaskan alur APT → repository → GPG → dpkg
- Update, upgrade, install, remove, purge dengan aman
- Cari info paket, kelola repository, PPA, dan pinning
- Mendiganosa dependency problem tanpa `rm -rf` ngawur

## 1. APT Architecture

Alur sederhana: `apt` (frontend) → `sources.list` + `/etc/apt/sources.list.d/` (daftar repo) → download `.deb` → `dpkg` (installer level rendah) → database di `/var/lib/dpkg/`.

```bash
$ ls /etc/apt/sources.list.d/
$ cat /etc/apt/sources.list
$ ls /var/lib/apt/lists/ | head
```

## 2. apt update

Sinkronisasi indeks paket, **tidak** install apapun. Wajib sebelum `install/upgrade`.

```bash
$ sudo apt update
$ sudo apt update -o Acquire::AllowInsecureRepositories=false
```

> Selalu `update` dulu setelah tambah repo baru.

## 3. apt upgrade

`upgrade` = naikkan versi tanpa hapus paket. `full-upgrade` boleh hapus untuk selesaikan dependency.

```bash
$ sudo apt upgrade -y
$ sudo apt full-upgrade -y
$ sudo apt autoremove --purge -y
```

Cek apa yang mau diubah dulu dengan `--dry-run` di server penting.

## 4. apt install

```bash
$ sudo apt install -y nginx
$ sudo apt install -y nginx=1.24.0-2ubuntu1
$ sudo apt install --no-install-recommends -y htop
```

Kunci: tentukan versi kalau di production, pakai `--no-install-recommends` biar ramping.

## 5. apt remove

Hapus program tapi sisakan config di `/etc`.

```bash
$ sudo apt remove -y nginx
```

## 6. apt purge

Hapus program + config. Pakai ini kalau mau bersih total.

```bash
$ sudo apt purge -y nginx
$ sudo apt autoremove --purge -y
```

## 7. apt search

```bash
$ apt search "web server" | head -n 40
$ apt search ^nginx$
```

## 8. apt show

Lihat versi, dependensi, maintainer, deskripsi.

```bash
$ apt show nginx
$ apt show -a docker-ce
```

## 9. apt-cache

Tools lama tapi berguna untuk policy dan dependensi.

```bash
$ apt-cache policy nginx
$ apt-cache depends nginx
$ apt-cache rdepends --installed nginx
```

## 10. dpkg

Level rendah: install `.deb` manual, audit, debug.

```bash
$ sudo dpkg -i ./paket.deb
$ dpkg -l | grep nginx
$ dpkg -L nginx
$ dpkg -S /usr/sbin/nginx
$ sudo dpkg --configure -a   # benerin install kepotong
```

## 11. Repositories

```bash
$ cat /etc/apt/sources.list
$ ls /etc/apt/sources.list.d/
$ sudo add-apt-repository universe
$ sudo apt update
```

Format deb822 baru (Ubuntu 24.04): `/etc/apt/sources.list.d/ubuntu.sources`.

## 12. GPG Signing

Tiap repo ditandatangani. Tanpa key yang benar, apt akan tolak.

```bash
$ sudo apt update -o Debug::Acquire::gpgv=true
$ ls /etc/apt/trusted.gpg.d/
$ ls /usr/share/keyrings/
```

Pola modern (jangan `apt-key` lagi, sudah deprecated):

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker.gpg
```

## 13. PPAs

Personal Package Archive dari Launchpad. Praktis tapi risiko: tidak resmi, bisa basi.

```bash
$ sudo add-apt-repository ppa:deadsnakes/ppa
$ sudo apt update
$ sudo add-apt-repository --remove ppa:deadsnakes/ppa
```

> Aturan: 1 PPA = 1 kebutuhan jelas. Catat kenapa ditambah.

## 14. Package Pinning

Kunci versi biar tidak ke-upgrade liar (misal Docker, kernel).

```bash
# /etc/apt/preferences.d/docker-pin
Package: docker-ce
Pin: version 5:24.*
Pin-Priority: 1000
```

```bash
$ sudo apt-mark hold nginx
$ apt-mark showhold
$ sudo apt-mark unhold nginx
```

## 15. Dependency Problems

Gejala: `unmet dependencies`, `broken packages`, `dpkg interrupted`.

```bash
$ sudo apt --fix-broken install
$ sudo dpkg --configure -a
$ sudo apt update && sudo apt full-upgrade
$ apt-cache policy <paket-bermasalah>
```

Jangan langsung `dist-upgrade -f` brutal tanpa baca output.

## 16. Software Lifecycle

Siklus sehat di VPS:

```bash
$ sudo apt update && sudo apt upgrade -y
$ sudo apt autoremove --purge -y
$ sudo apt autoclean
$ apt list --upgradable
```

Di production: jadwalkan maintenance window, snapshot dulu, baru upgrade. Otomatisasi penuh dibahas di Bab 08 (unattended-upgrades di Bab 05).

## Latihan

1. Cari paket `htop`, lihat info, install, purge, install lagi.
2. Tambah 1 repo, `update`, lalu hapus lagi. Perhatikan error GPG kalau key belum dipasang.
3. Hold 1 paket, coba `upgrade`, lalu unhold.
4. Simulasikan `dpkg --configure -a` setelah install `.deb` gagal.

## Rangkuman

- `update` = refresh indeks, `upgrade` = naikkan versi.
- `remove` sisakan config, `purge` bersihkan total.
- `apt` untuk harian, `dpkg` + `apt-cache` untuk debug.
- Repo + GPG + pinning = kunci stabilitas production.
