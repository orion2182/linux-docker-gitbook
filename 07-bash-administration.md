# 07. Bash Administration

> Admin tanpa bash = klik manual selamanya. Bab ini bash secukupnya tapi tajam untuk admin.

## Tujuan Pembelajaran

- Variabel, quotes, exit code, kondisional, loop, fungsi
- Command substitution, parameter, error handling, ShellCheck
- Nulis skrip admin yang aman (set -euo pipefail)

## 1. Variables

```bash
APP="myapp"
ENV="prod"
PORT=8080
echo "$APP $ENV $PORT"
BACKUP_DIR="/backup/$(date +%F)"
echo "$BACKUP_DIR"
```

Tanpa spasi di sekitar `=`. Selalu quote `"$VAR"` kecuali memang mau word-splitting.

## 2. Quotes

- `'...'` literal, `$VAR` tidak expand.
- `"..."` expand `$VAR` dan `$(cmd)`.
- `` `...` `` usang, pakai `$(...)`.

```bash
NAME="budi"
echo 'halo $NAME'   # halo $NAME
echo "halo $NAME"   # halo budi
echo "home: $HOME, tanggal: $(date +%F)"
```

## 3. Exit Codes

`0` sukses, selain itu gagal. `$?` = exit code perintah terakhir.

```bash
$ ls /ada; echo $?
$ ls /tidak-ada; echo $?
$ ping -c1 8.8.8.8 >/dev/null 2>&1; echo "ping exit: $?"
```

Cek exit code langsung, bukan parsing teks `success`.

## 4. Conditionals

```bash
if systemctl is-active --quiet nginx; then
  echo "nginx jalan"
else
  echo "nginx mati"
fi

[ -f /etc/nginx/nginx.conf ] && echo "config ada"
[ "$ENV" = "prod" ] && echo "PROD: hati-hati!"
```

Gunakan `[[ ]]` untuk test modern, `-f` file, `-d` dir, `-z` string kosong.

## 5. Loops

```bash
for svc in nginx docker ssh; do
  echo "== $svc =="
  systemctl is-active "$svc"
done

for f in /var/log/*.log; do
  echo "$f: $(du -h "$f" | cut -f1)"
done
```

Untuk daftar host: `while read -r h; do ssh "$h" uptime; done < hosts.txt`.

## 6. Functions

```bash
log() { echo "[$(date '+%F %T')] $*"; }
die() { echo "ERROR: $*" >&2; exit 1; }

backup_etc() {
  local dest="/backup/etc-$(date +%F).tar.gz"
  tar -czf "$dest" /etc/nginx /etc/docker 2>/dev/null || die "backup gagal"
  log "backup ok: $dest"
}
```

`local` di fungsi, return code jelas, 1 fungsi = 1 tugas.

## 7. Command Substitution

```bash
TODAY=$(date +%F)
CONTAINERS=$(docker ps -q | wc -l)
echo "hari $TODAY, container jalan: $CONTAINERS"
DISK_USE=$(df / | awk 'NR==2{print $5}')
echo "disk root: $DISK_USE"
```

## 8. Parameters

```bash
#!/usr/bin/env bash
# usage: ./deploy.sh <env> <versi>
ENV=${1:?isi env: prod/staging}
VERSI=${2:-latest}
echo "deploy $VERSI ke $ENV"
echo "argumen: $#, semua: $@, nama skrip: $0"
```

`${VAR:-default}` dan `${VAR:?pesan}` mencegah variabel kosong lolos ke `rm -rf /$KOSONG`.

## 9. Error Handling

Template wajib tiap skrip admin:

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'
log() { echo "[$(date '+%F %T')] $*"; }
trap 'echo "gagal di baris $LINENO"' ERR
```

- `-e` menghentikan skrip saat terjadi error, `-u` menolak variabel yang belum didefinisikan, dan `-o pipefail` meneruskan kegagalan dari pipeline.
- `trap` untuk cleanup: hapus tmp, turunkan lock.

## 10. ShellCheck

Linter bash. Wajib sebelum skrip naik ke server.

```bash
$ sudo apt install -y shellcheck
$ shellcheck ./backup.sh
$ shellcheck -S warning ./deploy.sh
```

Perbaiki SC2086 (unquoted var) dulu — sumber bug paling sering.

## 11. Administrative Scripts

Contoh skrip sehat: cek disk + alert.

```bash
#!/usr/bin/env bash
set -euo pipefail
THRESHOLD=85
USE=$(df / | awk 'NR==2{gsub(/%/,"",$5); print $5}')
if [ "$USE" -ge "$THRESHOLD" ]; then
  echo "WARN disk / ${USE}% >= ${THRESHOLD}% pada $(hostname) $(date -Is)"
  df -h; du -sh /var/log/* 2>/dev/null | sort -rh | head
  exit 2
fi
echo "OK disk / ${USE}%"
```

Simpan di `~/bin/`, berikan permission eksekusi dengan `chmod +x`, versioning menggunakan Git (Bab 09), lalu jadwalkan melalui cron atau systemd (Bab 08).

## Fungsi Perintah dan Sintaks

| Perintah atau sintaks | Fungsi |
| --- | --- |
| `bash` | Menjalankan skrip atau shell Bash. Shebang `#!/usr/bin/env bash` memilih Bash dari `PATH`. |
| `echo` | Menampilkan teks, nilai variabel, atau pesan log. |
| `date` | Menghasilkan tanggal dan waktu, termasuk format ISO dengan `-Is`. |
| `$(...)` | Command substitution; mengganti ekspresi dengan output perintah di dalamnya. |
| `$?` | Menyimpan exit code perintah terakhir. Nilai `0` berarti berhasil. |
| `if` / `case` | Mengevaluasi kondisi dan memilih blok perintah yang sesuai. |
| `for` / `while` | Mengulang perintah untuk daftar item atau selama kondisi terpenuhi. |
| `local` | Membatasi variabel agar hanya berlaku di dalam fungsi. |
| `trap` | Menjalankan tindakan ketika event tertentu terjadi, misalnya error atau exit. |
| `set -euo pipefail` | Mengaktifkan berhenti saat error, menolak variabel yang belum didefinisikan, dan meneruskan error dari pipeline. |
| `chmod +x` | Menambahkan permission eksekusi pada skrip. |
| `df` / `du` | Membaca kapasitas filesystem dan ukuran direktori atau file. |
| `awk` | Memproses teks berbentuk kolom atau record. |
| `sort` / `head` | Mengurutkan output dan mengambil sejumlah baris awal. |
| `shellcheck` | Menganalisis skrip Bash untuk menemukan bug dan pola yang tidak aman. |
