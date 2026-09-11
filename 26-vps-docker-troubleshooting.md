# 26. VPS & Docker Troubleshooting

> Bab paling sering dibuka jam 2 pagi. Ikuti alur, jangan tebak. Tiap bagian: gejala → perintah → penyebab umum → fix.

## Cara Pakai

1. Identifikasi lapisan: SSH → DNS → port/firewall → disk/CPU/mem → daemon → container → app → Nginx/TLS.
2. Kumpulkan fakta (`ps`, `ss`, `df`, `logs`, `inspect`) sebelum restart.
3. Ubah 1 hal, test, catat. Jangan `reboot + prune + reload` sekaligus.

## 1. SSH Failure

```bash
$ ssh -v user@IP
$ sudo systemctl status ssh
$ sudo sshd -T | grep -i "port\|permitroot\|passwordauth"
$ sudo tail -n 30 /var/log/auth.log
$ sudo ufw status verbose; sudo iptables -L INPUT -n --line-numbers | head
```

Umum: port salah, key permission 600/700 bukan, `PasswordAuthentication no` tapi paksa password, UFW blokir, fail2ban ban IP sendiri. Fix via console provider + `fail2ban-client set sshd unbanip IP`.

## 2. DNS Failure

```bash
$ ping -c2 8.8.8.8
$ ping -c2 google.com
$ resolvectl status; cat /etc/resolv.conf
$ getent hosts google.com; dig google.com +short
```

IP ok + nama gagal = DNS. Cek Netplan `nameservers`, `systemd-resolved`, bukan kabel. Jangan edit `resolv.conf` manual permanen (ketimpa, Bab 04).

## 3. Port Closed

```bash
$ ss -tlnp | grep -E ':80|:443|:3000'
$ curl -v http://127.0.0.1:3000/ 2>&1 | head -n 20
$ nmap -p 80,443 IP-VPS  # dari luar (lab sendiri)
```

Listen `127.0.0.1:3000` = hanya lokal (benar untuk di balik proxy). Butuh publik = `0.0.0.0` / `ports` benar. Cek `docker ps --format "{{.Ports}}"`.

## 4. Firewall Problem

```bash
$ sudo ufw status verbose numbered
$ sudo iptables -L -n -v --line-numbers | head -n 40
$ sudo iptables -t nat -L DOCKER -n -v | head -n 20
$ sudo nft list ruleset | head -n 40
```

Umum: UFW deny tapi lupa allow, `iptables -F` hancurkan chain Docker, urutan rule salah. Fix Docker chain: restart Docker (siap container ikut restart) + jangan flush manual di prod.

## 5. Disk Full

```bash
$ df -h; df -i
$ du -sh /var/* | sort -rh | head
$ du -sh /var/lib/docker/* | sort -rh | head
$ sudo lsof | grep deleted | head
$ docker system df -v | head -n 40
```

Fix: `journalctl --vacuum-time=7d`, `logrotate -f`, `docker system prune` (konfirmasi!), hapus backup lama, tambah volume/LVM (Bab 02). Cari deleted-handle: restart service yang pegang file.

## 6. High CPU

```bash
$ top -b -n1 | head -n 20
$ ps aux --sort=-%cpu | head -n 10
$ docker stats --no-stream
$ cat /proc/loadavg; nproc
$ vmstat 1 5
```

Bedakan `%us` (app), `%sy` (kernel), `%wa` (disk). Container biang = `stats` + `logs` + limit (Bab 19/21). Jangan reboot tanpa tahu siapa.

## 7. High Memory

```bash
$ free -h
$ ps aux --sort=-%mem | head -n 10
$ dmesg -T | grep -i oom | tail
$ docker stats --no-stream --format "table {{.Name}}\t{{.MemUsage}}\t{{.MemPerc}}"
```

OOM killer di dmesg = korban sudah dibunuh. Fix: limit container, tambah swap sementara, naikkan RAM, perbaiki leak. Bukan tambah swap selamanya.

## 8. Docker Daemon Failure

```bash
$ systemctl status docker containerd --no-pager
$ sudo journalctl -u docker --since "30 min ago" | tail -n 80
$ cat /etc/docker/daemon.json | python3 -m json.tool
$ df -h /var/lib/docker
$ dockerd --version; docker info 2>&1 | head -n 20
```

Umum: JSON invalid setelah edit daemon, disk penuh, socket permission, iptables rusak. Rollback `daemon.json` dari backup + `systemctl restart docker`.

## 9. Container Crash

```bash
$ docker ps -a --format "table {{.Names}}\t{{.Status}}"
$ docker logs --tail 100 <nama>
$ docker inspect -f '{{.State.ExitCode}} {{.State.Error}} {{.State.OOMKilled}}' <nama>
$ docker events --since 30m | grep <nama>
```

Exit 0 = selesai wajar (command salah untuk long-run). 137 = OOM/kill. 1 = app error (baca logs). Restart loop = healthcheck gagal / env hilang / volume permission.

## 10. Image Pull Failure

```bash
$ docker pull <image> 2>&1 | tail -n 20
$ ping -c2 registry-1.docker.io; curl -I https://registry-1.docker.io/v2/
$ docker login registry.example.com
$ cat ~/.docker/config.json
$ df -h /var/lib/docker
```

Umum: typo tag, rate limit Hub, auth expired, DNS, disk penuh, platform ARM vs AMD64. Pin digest + mirror registry untuk prod (Bab 23).

## 11. Permission Error

```bash
$ ls -l <path>; namei -l /opt/myapp/data/file.db
$ docker exec <nama> whoami; docker exec <nama> id
$ docker inspect -f '{{.Config.User}} {{json .Mounts}}' <nama> | python3 -m json.tool
$ sudo dmesg | grep -i "apparmor.*denied" | tail
$ getenforce 2>&1; sestatus 2>&1 | head
```

Fix: `chown UID:GID` sesuai USER container, `--chown` saat COPY, volume init chown, jangan `777`. Cek AppArmor sebelum chmod brutal.

## 12. Volume Problem

```bash
$ docker volume ls
$ docker inspect -f '{{json .Mounts}}' <nama> | python3 -m json.tool
$ ls -la /var/lib/docker/volumes/<vol>/_data | head
$ mount | grep <vol>
```

Anonymous volume nyasar = data di tempat salah. `down -v` hapus volume = data hilang. Backup dulu sebelum prune/migrasi (Bab 15/24).

## 13. Network Problem

Lihat Bab 16 ringkas:

```bash
$ docker network ls; docker network inspect <net> | head -n 60
$ docker exec a ping -c2 b
$ docker exec a nslookup b; cat /etc/resolv.conf
$ ss -tlnp | grep <port>
```

Nama gagal = beda network / typo. IP ok + nama gagal = DNS bridge. Host ok + luar gagal = publish/firewall.

## 14. Nginx 502

App tidak bisa dihubungi (lihat Bab 22):

```bash
$ sudo tail -n 30 /var/log/nginx/error.log
$ curl -v http://127.0.0.1:3000/health
$ docker compose ps; docker compose logs --tail 30 web
$ ss -tlnp | grep 3000
```

Fix: nyalakan app, betulkan `proxy_pass` (nama service vs IP), samakan network, tunggu healthy (jangan proxy ke container starting).

## 15. Nginx 504

Upstream kelamaan:

```bash
$ curl -w "code:%{http_code} time:%{time_total}s\n" -o /dev/null -s https://contoh.com/lambat
$ sudo tail -n 30 /var/log/nginx/error.log | grep -i timeout
$ docker compose logs --tail 50 api | grep -i "slow\|timeout\|deadlock"
```

Naikkan `proxy_read_timeout` sementara + perbaiki akar (query, worker, limit CPU). Timeout besar tanpa fix = antrean meledak.

## 16. TLS Problems

```bash
$ echo | openssl s_client -connect contoh.com:443 -servername contoh.com 2>/dev/null | openssl x509 -noout -dates -subject -issuer
$ sudo certbot certificates
$ sudo nginx -t; sudo tail -n 20 /var/log/nginx/error.log
$ curl -v https://contoh.com 2>&1 | head -n 30
```

Expired = renew gagal (port 80 tertutup / webroot salah / DNS pindah). Mixed content = app masih硬code `http://`. Redirect loop = `X-Forwarded-Proto` hilang + app paksa https.

## Template Laporan Insiden (copy-paste)

```text
Waktu:
Gejala:
Perintah + output (ps/ss/df/logs/inspect):
Perubahan terakhir (deploy/config/cron?):
Dugaan lapisan:
Fix 1 baris:
Verifikasi (curl/health/monitor):
Pencegahan (alert/backup/runbook):
```

## Rangkuman

- Fakta dulu, restart belakangan. Satu perubahan satu test.
- Hafalkan segitiga: `logs + inspect + events` (Docker), `ss + ip + curl` (network), `df + free + top` (resource).
- Tiap insiden ditutup dengan alert + runbook, bukan cuma "sudah nyala lagi".
