# Linux & Docker — Dari VPS Kosong Sampai Production

> Panduan praktis administrasi Linux (Ubuntu), Docker, dan operasional VPS untuk pemula sampai siap production.

Buku ini disusun berurutan. Jangan loncat-loncat kalau baru mulai:

1. **Fondasi Linux (Bab 01–09):** package management, storage, networking, security, logs, bash, automation, git.
2. **Core Docker (Bab 10–18):** konsep container, instalasi, CLI, image, Dockerfile, storage, networking, Compose, multi-container.
3. **Production (Bab 19–26):** security, rootless, deploy di VPS, Nginx reverse proxy + TLS, registry, backup, monitoring, troubleshooting.
4. **Studi Kasus (Bab 27):** bedah stack TokoApp dari image Docker sampai deployment multi-container.

## Cara Pakai Buku Ini

- Tiap bab punya **Tujuan Pembelajaran**, materi per sub-bab, **contoh perintah**, **latihan**, dan **rangkuman**.
- Semua contoh diuji di **Ubuntu 22.04 / 24.04 LTS** (x86_64) di VPS.
- Blok perintah dengan `$` = user biasa, `#` = root / sudo.
- Kalau stuck, langsung lompat ke **Bab 26. VPS & Docker Troubleshooting**.

## Prasyarat

- VPS / VM Ubuntu 22.04+ (1 vCPU, 1–2 GB RAM cukup untuk mulai)
- Akses SSH + user dengan `sudo`
- Domain (opsional, baru wajib di Bab 21–22)
- Docker 24+ (cara install di Bab 11)

## Konvensi

```bash
$ whoami        # user biasa
$ sudo whoami   # root via sudo
$ docker ps     # perintah docker
```

## Panduan Perintah Dasar

Tabel berikut menjelaskan perintah shell yang sering muncul di seluruh buku.

| Perintah | Fungsi |
| --- | --- |
| `pwd` | Menampilkan direktori kerja saat ini. |
| `ls` | Menampilkan isi direktori. `-l` menampilkan detail, sedangkan `-a` menampilkan file tersembunyi. |
| `cd` | Berpindah direktori. `cd ..` naik satu tingkat, sedangkan `cd -` kembali ke direktori sebelumnya. |
| `cat` | Menampilkan atau menggabungkan isi file teks. Gunakan `less` untuk file yang panjang. |
| `less` | Membaca file secara interaktif tanpa memuat seluruh isi ke layar sekaligus. Tekan `q` untuk keluar. |
| `head` | Menampilkan beberapa baris pertama file atau output. |
| `tail` | Menampilkan beberapa baris terakhir. `tail -f` mengikuti tambahan log secara langsung. |
| `grep` | Mencari pola teks. `-i` mengabaikan kapitalisasi dan `-E` mengaktifkan pola regex extended. |
| `find` | Mencari file berdasarkan nama, tipe, ukuran, waktu, atau kriteria lain. |
| `which` / `command -v` | Menampilkan lokasi executable yang dipanggil shell. |
| `man` | Membuka manual resmi suatu perintah. Contoh: `man ls`. |
| `sudo` | Menjalankan satu perintah dengan hak administrator sesuai kebijakan sudoers. |
| `echo` | Menampilkan teks atau nilai variabel. Sering dipakai bersama `>` atau `>>`. |
| `tee` | Menulis input ke file sekaligus meneruskannya ke output standar. |
| `chmod` | Mengubah permission file atau direktori. |
| `chown` | Mengubah pemilik dan grup file atau direktori. |
| `|` | Pipe: meneruskan output perintah pertama sebagai input perintah berikutnya. |
| `>` / `>>` | Mengarahkan output ke file; `>` menimpa, sedangkan `>>` menambahkan di akhir. |
| `2>&1` | Menggabungkan standard error dengan standard output. |
| `&&` | Menjalankan perintah berikutnya hanya jika perintah sebelumnya berhasil. |
| `;` | Menjalankan perintah berikutnya tanpa memperhatikan status perintah sebelumnya. |

Flag yang sering digunakan:

- `-h` atau `--human-readable`: menampilkan ukuran dalam format yang mudah dibaca.
- `-v` atau `--verbose`: menampilkan proses yang sedang dijalankan secara lebih rinci.
- `-q` atau `--quiet`: mengurangi output.
- `-f` dapat berarti `follow`, `force`, atau `file`, bergantung pada perintahnya. Periksa `man` jika ragu.

Contoh membaca konfigurasi dengan aman:

```bash
$ pwd
$ ls -la /etc/nginx
$ sudo cat /etc/nginx/nginx.conf
$ sudo nginx -t 2>&1 | tee /tmp/nginx-test.log
```

> **Catatan:** Jangan copy-paste buta perintah `rm`, `mkfs`, `iptables -F`, `docker prune` di server production. Pahami dulu di tiap bab.

## Struktur

Lihat `SUMMARY.md` untuk daftar isi lengkap (27 bab).
