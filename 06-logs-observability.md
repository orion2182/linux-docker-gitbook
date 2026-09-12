# 06. Logs & Observability

> Server yang tidak diobservasi merupakan sumber risiko operasional. Bab ini membantu menjawab pertanyaan "mengapa service berhenti?" dengan cepat.

## Tujuan Pembelajaran

- Baca log via journald/journalctl dan `/var/log`
- Putar log dengan logrotate, baca kernel via dmesg
- Monitor CPU, memory, disk, network secara cepat

## 1. journald

Sistem log biner systemd. Cepat, terstruktur, ada sejak boot.

```bash
$ journalctl --disk-usage
$ sudo journalctl -b   # log sejak boot terakhir
$ cat /etc/systemd/journald.conf | grep -v "^#" | grep -v "^$"
```

## 2. journalctl

Query wajib hafal:

```bash
$ journalctl -xeu nginx --no-pager | tail -n 50
$ journalctl -u docker --since "1 hour ago"
$ journalctl -p err -b
$ journalctl -f   # follow live
$ journalctl --since "2026-09-10" --until "2026-09-11" -u ssh
```

Filter: `-u` unit, `-p` priority, `--since`, `-g` grep pola.

## 3. /var/log

File klasik (teks). Penting saat journald tidak ada / app tulis manual.

```bash
$ ls -lh /var/log/
$ ls -lh /var/log/nginx/ /var/log/mysql/ 2>&1 | head
$ sudo tail -n 100 -f /var/log/syslog
```

Jangan `chmod 777 /var/log`. Rusak permission = log berhenti.

## 4. syslog

Agregator pesan sistem (`/var/log/syslog`, `/var/log/messages` di distro lain).

```bash
$ sudo grep -i "oom\|error\|fail" /var/log/syslog | tail -n 30
$ logger "tes observability bab06"
$ sudo tail -n 5 /var/log/syslog
```

## 5. auth logs

Jejak login. Pertama dibuka saat dicurigai dibobol.

```bash
$ sudo tail -n 50 /var/log/auth.log
$ sudo grep -i "failed\|invalid\|accepted" /var/log/auth.log | tail -n 30
$ sudo lastb | head
```

## 6. logrotate

Agar disk tidak penuh oleh log. Config di `/etc/logrotate.conf` + `/etc/logrotate.d/`.

```bash
$ cat /etc/logrotate.d/nginx
$ sudo logrotate -d /etc/logrotate.d/nginx 2>&1 | head -n 60
$ sudo logrotate -f /etc/logrotate.d/nginx
$ ls -lh /var/log/*.gz | head
```

Untuk Docker: batasi log via `max-size` + `max-file` (Bab 21).

## 7. dmesg

Pesan kernel: hardware, OOM killer, filesystem error, AppArmor.

```bash
$ dmesg | tail -n 50
$ dmesg -T | grep -i "oom\|error\|fail\|ext4" | tail -n 30
$ sudo journalctl -k -b | tail -n 50
```

OOM killer di dmesg = RAM habis, proses dibunuh kernel.

## 8. CPU Monitoring

```bash
$ top -b -n1 | head -n 20
$ htop  # kalau ada
$ uptime; cat /proc/loadavg
$ vmstat 1 5
$ mpstat -P ALL 1 3
```

Load > jumlah vCPU dalam waktu lama = antrean. Lihat siapa: `%us` user, `%sy` system, `%wa` I/O wait.

## 9. Memory Monitoring

```bash
$ free -h
$ cat /proc/meminfo | grep -E "MemTotal|MemAvailable|Swap"
$ vmstat -s | head
$ ps aux --sort=-%mem | head -n 15
$ dmesg -T | grep -i oom | tail
```

`available` kecil + `swap` kepakai + OOM = tambah RAM / batasi container (Bab 21).

## 10. Disk Monitoring

```bash
$ df -h; df -i
$ iostat -xz 1 3
$ iotop -o -b -n 5 | head -n 30
$ du -sh /var/log/* | sort -rh | head
```

`%util` 100% + `await` tinggi = disk bottleneck, bukan CPU.

## 11. Network Monitoring

```bash
$ ss -s; ss -tunap | head -n 20
$ ip -s link
$ ping -c 20 1.1.1.1 | tail -n 5
$ sudo iftop  # kalau ada
$ cat /proc/net/dev
```

Paket drop atau error yang meningkat pada `ip -s link` dapat menunjukkan masalah driver, MTU, firewall, atau interface yang kelebihan beban.

## Fungsi Perintah

| Perintah | Fungsi |
| --- | --- |
| `journalctl` | Membaca log journald berdasarkan service, waktu, prioritas, atau boot. |
| `systemctl` | Memeriksa dan mengelola status service serta timer systemd. |
| `tail -f` | Mengikuti tambahan baris log secara langsung. |
| `grep` | Mencari pesan tertentu, misalnya `error`, `oom`, atau `failed`. |
| `logger` | Mengirim pesan uji ke syslog/journald. |
| `logrotate` | Memutar, mengompresi, dan menghapus log lama berdasarkan kebijakan retensi. |
| `dmesg` | Menampilkan pesan kernel, termasuk OOM killer dan error filesystem. |
| `top` / `htop` | Memantau proses dan penggunaan CPU atau RAM secara interaktif. |
| `free` | Menampilkan RAM tersedia, cache, dan penggunaan swap. |
| `vmstat` | Menampilkan statistik proses, memory, swap, dan I/O secara berkala. |
| `df` / `du` | Memeriksa kapasitas filesystem dan ukuran direktori atau file. |
| `iostat` / `iotop` | Menganalisis aktivitas dan latency I/O disk. |
| `ss` / `ip -s link` | Memeriksa socket, interface, paket, error, dan drop jaringan. |
| `ping` / `iftop` | Menguji latency dan memantau lalu lintas jaringan secara langsung. |
