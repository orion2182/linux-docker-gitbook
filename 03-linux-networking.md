# 03. Linux Networking

> Tanpa pemahaman networking, gangguan jaringan sulit dianalisis. Bab ini membahas fondasi TCP/IP dan tools diagnosis harian.

## Tujuan Pembelajaran

- Menjelaskan IP, CIDR, gateway, routing, DNS, port, socket
- Pakai `ip`, `ss`, `ping`, `traceroute`, `curl`, `wget`
- Membedakan masalah DNS vs routing vs firewall vs aplikasi

## 1. TCP/IP

Model ringkas: Link → Internet (IP) → Transport (TCP/UDP) → Application (HTTP/DNS/SSH).

- TCP = andal, ada handshake, dipakai HTTP/SSH/DB.
- UDP = cepat tanpa jaminan, dipakai DNS, QUIC, streaming.

## 2. IPv4

32-bit, format desimal `192.168.1.10`. Makin langka publik, makanya ada NAT.

```bash
$ ip -4 addr show
$ ip -4 route show
```

## 3. IPv6

128-bit, format heksa `2001:db8::1`. Tanpa NAT, wajib paham dasarnya untuk VPS modern.

```bash
$ ip -6 addr show
$ ip -6 route show
$ ping -6 google.com
```

## 4. CIDR

`IP/prefix`. Prefix = jumlah bit network.

- `/24` = 256 IP (255.255.255.0)
- `/16` = 65.536 IP
- `/32` = 1 IP (host route)

```bash
$ ipcalc 192.168.1.10/24
$ python3 -c "import ipaddress; print(list(ipaddress.ip_network('10.0.0.0/29')))"
```

## 5. Gateway

Gateway merupakan pintu keluar menuju network lain. Default gateway harus benar; jika tidak, koneksi internet tidak akan berfungsi.

```bash
$ ip route show default
$ ip route get 8.8.8.8
```

## 6. Routing

Tabel routing = peta jalan paket.

```bash
$ ip route show
$ ip route get 1.1.1.1
$ sudo ip route add 10.99.0.0/16 via 10.0.0.1 dev eth0
$ sudo ip route del 10.99.0.0/16
```

> Route manual hilang saat reboot. Permanen via Netplan (Bab 04).

## 7. DNS

Nama → IP. Urutan: `/etc/hosts` → resolver (`/etc/resolv.conf` / systemd-resolved) → upstream.

```bash
$ cat /etc/resolv.conf
$ resolvectl status
$ getent hosts github.com
$ dig github.com +short
$ nslookup github.com
```

Jika DNS bermasalah, `ping 8.8.8.8` dapat berhasil, tetapi `ping google.com` gagal.

## 8. Ports

16-bit, 0–65535. Well-known: 22 SSH, 80 HTTP, 443 HTTPS, 53 DNS, 3306 MySQL, 5432 Postgres, 6379 Redis.

```bash
$ cat /etc/services | grep -E '^(http|https|ssh)'
```

## 9. Sockets

Kombinasi IP:port + state (LISTEN, ESTABLISHED, TIME_WAIT).

```bash
$ ss -tulpn
$ ss -s
$ ss -tunap | head -n 30
```

## 10. ip

Pengganti `ifconfig`/`route` yang deprecated.

```bash
$ ip addr
$ ip link
$ ip route
$ ip neigh   # ARP table
```

## 11. ss

Pengganti `netstat`.

```bash
$ ss -tulpn              # siapa listen di mana
$ ss -tunap | grep :80
$ ss -plant | grep nginx
```

## 12. ping

Cek reachability + latency, bukan cek aplikasi.

```bash
$ ping -c 4 8.8.8.8
$ ping -c 4 google.com
```

Gagal IP tapi nama gagal = DNS. Keduanya gagal = routing/firewall.

## 13. traceroute

Lihat hop per hop ke mana paket nyangkut.

```bash
$ traceroute 8.8.8.8
$ traceroute google.com
$ mtr --report google.com   # kalau ada mtr, lebih bagus
```

Bintang `* * *` di tengah belum tentu rusak — banyak router blokir ICMP.

## 14. curl

Swiss-army knife HTTP + debug header/timing.

```bash
$ curl -I https://example.com
$ curl -v https://example.com 2>&1 | head -n 40
$ curl -w "code:%{http_code} time:%{time_total}s\n" -o /dev/null -s https://example.com
$ curl --resolve example.com:443:127.0.0.1 https://example.com -k -I
```

## 15. wget

Fokus download + mirror.

```bash
$ wget https://example.com/file.tar.gz
$ wget -c https://example.com/file.iso   # resume
$ wget -r -np -nH --cut-dirs=2 https://example.com/docs/
```

## 16. Network Troubleshooting

Gunakan urutan pemeriksaan berikut agar lapisan masalah tidak tertukar:

```bash
$ ip addr; ip route show default
$ ping -c 3 8.8.8.8
$ ping -c 3 google.com
$ resolvectl status; cat /etc/resolv.conf
$ ss -tulpn | grep -E ':80|:443|:22'
$ curl -v http://127.0.0.1/
$ sudo iptables -L -n -v | head; sudo ufw status verbose
```

Pola cepat:
- Tidak ada IP → kabel/Netplan/cloud-init.
- Ping IP gagal → gateway/route/firewall provider.
- Ping IP ok, nama gagal → DNS.
- curl localhost ok, dari luar gagal → firewall / listen `127.0.0.1` bukan `0.0.0.0`.

## Fungsi Perintah

| Perintah | Fungsi |
| --- | --- |
| `ip addr` / `ip link` | Menampilkan alamat IP dan status interface jaringan. |
| `ip route` | Menampilkan atau mengelola tabel routing. `ip route get` menunjukkan jalur yang dipilih untuk tujuan tertentu. |
| `ip neigh` | Menampilkan tabel neighbor, yaitu pemetaan IP dan alamat MAC pada jaringan lokal. |
| `ss` | Menampilkan socket dan port. `-t` TCP, `-u` UDP, `-l` listening, `-p` proses, dan `-n` alamat numerik. |
| `ping` | Menguji keterjangkauan host dan mengukur latency menggunakan ICMP. |
| `traceroute` / `mtr` | Menampilkan hop jaringan menuju tujuan; `mtr` menggabungkan traceroute dan monitoring berkala. |
| `dig` / `nslookup` | Menguji resolusi DNS dan melihat jawaban DNS secara langsung. |
| `getent hosts` | Menguji resolusi nama menggunakan resolver sistem, termasuk `/etc/hosts`. |
| `resolvectl` | Memeriksa status dan melakukan query melalui `systemd-resolved`. |
| `curl` | Mengirim request HTTP atau HTTPS dan memeriksa header, status, isi, serta waktu respons. |
| `wget` | Mengunduh file atau melakukan mirror HTTP. |
| `iptables` / `ufw` | Memeriksa atau mengelola aturan firewall; UFW merupakan frontend yang lebih sederhana. |
| `cat` | Membaca file konfigurasi, misalnya `/etc/resolv.conf`. |
| `grep` | Menyaring output berdasarkan pola, misalnya port atau alamat tertentu. |
