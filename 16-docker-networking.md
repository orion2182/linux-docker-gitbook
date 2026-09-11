# 16. Docker Networking

> Networking Docker membingungkan di awal, simpel setelah paham bridge vs host vs none + DNS bawaan.

## Tujuan Pembelajaran

- Menjelaskan docker0, bridge, user-defined bridge, host, none
- Publish port, DNS, komunikasi antar container, IPv4/IPv6
- Troubleshooting network container

## 1. docker0

Bridge default (`172.17.0.0/16`) yang dibuat saat Docker install. Container tanpa `--network` nempel sini.

```bash
$ ip addr show docker0
$ docker network ls
$ docker network inspect bridge | head -n 60
```

Jangan pakai `bridge` default untuk prod — tidak ada DNS nama, harus pakai IP.

## 2. Bridge

Default driver NAT: container dapat IP privat, keluar via MASQUERADE host.

```bash
$ docker run -d --name a nginx:alpine
$ docker inspect -f '{{.NetworkSettings.IPAddress}}' a
$ docker exec a ping -c 2 8.8.8.8
$ sudo iptables -t nat -L -n | grep -i docker | head
```

Cocok untuk single-host sederhana, tapi pindah ke user-defined bridge untuk fitur DNS.

## 3. User-defined Bridge

Wajib untuk multi-container: auto DNS by name + isolasi per app.

```bash
$ docker network create app-net
$ docker run -d --name web --network app-net nginx:alpine
$ docker run -d --name api --network app-net myapp:1
$ docker exec web ping -c 2 api
$ docker exec api wget -qO- http://web | head
$ docker network inspect app-net | grep -A5 -B5 web
```

Pola: 1 app = 1 network (`frontend`, `backend`). DB hanya di `backend`, tidak terekspos ke `frontend`.

## 4. Host

Container pakai network host langsung (tanpa NAT, port = port host). Cepat, tapi tanpa isolasi + konflik port.

```bash
$ docker run -d --name fast --network host nginx:alpine
$ curl -I http://127.0.0.1:80
$ docker inspect -f '{{.HostConfig.NetworkMode}}' fast
```

Pakai hanya kalau butuh performa / port range aneh / IPv6 tricky. Tidak jalan di Docker Desktop Mac/Win sama persis.

## 5. None

Tanpa network sama sekali. Untuk job batch sensitif / hardening.

```bash
$ docker run -d --name isolated --network none alpine sleep 3600
$ docker exec isolated ip addr
$ docker exec isolated ping -c1 8.8.8.8 || echo "memang tidak bisa"
```

## 6. Port Publishing

Map host:container. Format `-p [IP:]HOST:CONTAINER[/tcp|/udp]`.

```bash
$ docker run -d --name web -p 8080:80 nginx:alpine
$ docker run -d --name web2 -p 127.0.0.1:8081:80 nginx:alpine
$ docker ps --format "{{.Names}} {{.Ports}}"
$ ss -tlnp | grep 8080
$ curl -I http://127.0.0.1:8080
```

Prod di balik Nginx (Bab 22): bind ke `127.0.0.1` saja, jangan `0.0.0.0` untuk DB/internal.

## 7. Container DNS

Di user-defined bridge, Docker sediakan resolver `127.0.0.11`. Nama container = hostname.

```bash
$ docker exec web cat /etc/resolv.conf
$ docker exec web nslookup api
$ docker exec web getent hosts api
$ docker run --rm --network app-net alpine nslookup web
```

Custom DNS:

```bash
$ docker run -d --dns 1.1.1.1 --dns-search example.com --name test nginx:alpine
```

atau di `daemon.json` untuk global.

## 8. Container-to-Container Communication

Pola benar: via nama service di 1 network, bukan IP hardcode, bukan link legacy.

```bash
# web butuh api:
$ docker exec web wget -qO- http://api:3000/health | head
# dari host via publish:
$ curl http://127.0.0.1:8080/
```

Di Compose (Bab 17) ini otomatis: `http://db:5432`, `http://redis:6379`.

## 9. IPv4 / IPv6

Default IPv4 NAT. IPv6 butuh enable di daemon + subnet.

```bash
$ docker network inspect bridge | grep -i subnet
$ cat /etc/docker/daemon.json
# contoh enable IPv6 (butuh planning subnet!):
# { "ipv6": true, "fixed-cidr-v6": "fd00:dead:beef::/48" }
```

Jangan enable IPv6 asal di prod tanpa firewall + routing jelas.

## 10. Network Troubleshooting

Urutan:

```bash
$ docker network ls
$ docker network inspect app-net | head -n 80
$ docker inspect -f '{{json .NetworkSettings.Networks}}' web | python3 -m json.tool
$ docker exec web ip addr
$ docker exec web ping -c 2 api
$ docker exec web wget -qO- http://api:3000/health || echo "app layer gagal"
$ ss -tlnp | grep -E '8080|80'
$ sudo iptables -L DOCKER-USER -n -v | head
```

Kasus umum:
- `ping IP ok, ping nama gagal` → bukan di user-defined bridge / typo nama.
- `curl host ok, dari luar gagal` → publish salah / UFW / bind `127.0.0.1` vs `0.0.0.0`.
- Tiba-tiba putus setelah `iptables -F` → chain Docker rusak, restart Docker (hati-hati container ikut restart).
- Konflik subnet dengan VPC/kantor → ganti subnet via `docker network create --subnet`.

## Latihan

1. Buat `app-net`, 2 container saling ping by name. Buktikan di `bridge` default gagal by name.
2. Publish 1 service ke `127.0.0.1` saja, buktikan dari luar tidak bisa langsung.
3. `inspect` network, gambar IP tiap container.
4. Rusakkan 1 network (disconnect), sambungkan lagi via `network connect/disconnect`.

## Rangkuman

- Default bridge untuk coba, user-defined bridge untuk kerja serius.
- Komunikasi via nama, publish minimal, isolasi per app.
- `network inspect` + `exec ping/nslookup` = diagnosa 2 menit.
