# 02. Storage Administration

> Disk penuh pada tengah malam merupakan salah satu insiden paling merepotkan bagi administrator. Bab ini menjelaskan `lsblk`, mount, fstab, LVM, dan swap.

## Tujuan Pembelajaran

- Memetakan block device → filesystem → mountpoint
- Mount manual dan permanen via `/etc/fstab` + UUID
- Cek disk usage dan tambah ruang dengan LVM / swap
- Troubleshooting disk penuh, read-only, gagal mount

## 1. Block Devices

```bash
$ lsblk -f
$ lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,FSTYPE
$ sudo fdisk -l
$ sudo blkid
```

Pola nama: `sda` (SATA), `vda` (KVM/VPS), `nvme0n1` (NVMe). Partisi = `sda1`, `vda1`.

## 2. lsblk

Perintah utama untuk visualisasi cepat.

```bash
$ lsblk
$ lsblk -f
$ lsblk -d -o NAME,SIZE,MODEL
```

## 3. Filesystems

Umum di Ubuntu: `ext4` (default), `xfs`, `btrfs`, `vfat` (EFI), `swap`.

```bash
$ df -T
$ sudo mkfs.ext4 /dev/vdb1   # HATI-HATI: hapus data!
$ sudo e2label /dev/vda1
```

> Jangan `mkfs` di disk yang sudah ada datanya. Cek `lsblk + blkid` 2x.

## 4. mount / umount

```bash
$ sudo mkdir -p /data
$ sudo mount /dev/vdb1 /data
$ mount | grep /data
$ sudo umount /data
$ sudo umount -l /data   # lazy kalau device busy
```

Lihat yang sibuk dengan `lsof +D /data` atau `fuser -m /data`.

## 5. /etc/fstab

Biar mount permanen survive reboot. Selalu backup dulu.

```bash
$ sudo cp /etc/fstab /etc/fstab.bak
$ cat /etc/fstab
# Contoh:
# UUID=xxxx-xxxx  /data  ext4  defaults,nofail  0  2
$ sudo mount -a   # test tanpa reboot
$ sudo findmnt --verify
```

Opsi penting: `nofail` (VPS tetap boot walau disk tambahan hilang), `noatime`.

## 6. UUID

Nama `/dev/sdX` bisa berubah. UUID tidak.

```bash
$ sudo blkid
$ ls -l /dev/disk/by-uuid/
```

Selalu pakai `UUID=` di fstab untuk production.

## 7. Disk Usage

```bash
$ df -h
$ df -i               # cek inode penuh!
$ du -sh /* | sort -rh | head
$ du -sh /var/* | sort -rh | head
$ sudo ncdu /var      # kalau ada, lebih enak
```

Kasus klasik: disk masih penuh setelah hapus log karena file masih dipegang proses. Cek `sudo lsof | grep deleted`.

## 8. LVM

Layer fleksibel: PV → VG → LV. Enak untuk resize tanpa repartisi.

```bash
$ sudo pvs; sudo vgs; sudo lvs
$ sudo lvdisplay
$ sudo lvextend -r -L +10G /dev/ubuntu-vg/ubuntu-lv
$ df -h /
```

Alur tambah disk baru ke LVM: `pvcreate → vgextend → lvextend -r → resize2fs/xfs_growfs`.

## 9. Swap

RAM habis? Swap jadi penyelamat sementara (bukan pengganti RAM).

```bash
$ free -h
$ swapon --show
$ sudo fallocate -l 2G /swapfile
$ sudo chmod 600 /swapfile
$ sudo mkswap /swapfile
$ sudo swapon /swapfile
# permanen di fstab:
# /swapfile none swap sw 0 0
$ cat /proc/sys/vm/swappiness
```

Untuk VPS kecil 1 GB, swap 1–2 GB + monitoring (Bab 06) cukup.

## 10. Storage Troubleshooting

Checklist:

```bash
$ df -h; df -i
$ lsblk -f
$ sudo dmesg | tail -n 50
$ sudo findmnt --verify
$ sudo mount -a
$ sudo e2fsck -n /dev/vdb1   # cek tanpa ubah (unmount dulu untuk fix)
```

Gejala → aksi:
- `No space left` tapi `df` masih ada → cek `df -i` (inode habis, banyak file kecil).
- `read-only filesystem` → cek `dmesg` (error disk), remount `mount -o remount,rw /`.
- Gagal boot karena fstab → pakai `nofail`, boot ke recovery, `mount -a` test.

## Fungsi Perintah

| Perintah | Fungsi |
| --- | --- |
| `lsblk` | Menampilkan struktur block device, partisi, filesystem, dan mount point. |
| `blkid` | Menampilkan UUID dan tipe filesystem pada device. |
| `fdisk` | Memeriksa atau mengelola tabel partisi; operasi penulisan harus dilakukan dengan sangat hati-hati. |
| `mkfs` | Membuat filesystem baru pada device. Perintah ini menghapus data yang ada pada device tersebut. |
| `mount` / `umount` | Menghubungkan atau melepas filesystem dari mount point. |
| `findmnt` | Menampilkan filesystem yang sedang ter-mount dan memvalidasi fstab. |
| `df` | Menampilkan kapasitas filesystem yang terpakai dan tersedia. `-i` menampilkan penggunaan inode. |
| `du` | Menghitung penggunaan ruang oleh file atau direktori. |
| `lsof` | Menampilkan file yang sedang dibuka oleh proses; berguna untuk menemukan file yang sudah dihapus tetapi masih memakan ruang. |
| `pvs` / `vgs` / `lvs` | Menampilkan physical volume, volume group, dan logical volume LVM. |
| `lvextend` | Memperbesar logical volume. Opsi `-r` juga menyesuaikan filesystem jika didukung. |
| `free` | Menampilkan penggunaan RAM dan swap. |
| `swapon` / `mkswap` | Membuat, mengaktifkan, atau menampilkan area swap. |
| `dmesg` | Membaca pesan kernel, termasuk error disk dan filesystem. |
