# 05 — Penyimpanan

Semua penyimpanan bersama berada di **t4-storage**, dibagikan lewat NFS di
jaringan storage 192.168.30.0/24.

---

## 1. Peta penyimpanan (per 1 Oktober 2026)

| Storage | Jenis | Kapasitas | Sisa | Di-mount di node? |
|---|---|---|---|---|
| **bio-pool** (`/media/bio-pool`) | SSD, ZFS (7 SSD) | 19,9 TB | **2,26 TB (87%)** | Ya, semua node |
| NVME-3.6TB (`/media/t4-storage-nvme`) | NVMe | 3,3 TB | 2,6 TB | Ya (compute001, compute002) |
| 80-Storage1 | HDD RAID5 | 73 TB | ±13,6 TB | Tidak |
| 96-Storage | HDD RAID5 | 87 TB | **1,3 TB (99%)** | Tidak |
| scratch2tb | SSD | 2 TB | 1,9 TB | Tidak |
| HDD-POD5 | HDD 12 TB | 11 TB | ±6,6 TB | Tidak |
| 5 SSD lab (bekas Windows) | SSD | ±7,5 TB | – | Read-only, menunggu diformat |

**Peringatan:** bio-pool (satu-satunya penyimpanan cepat untuk pipeline) sudah
87%. Di atas 80% ZFS melambat; kalau penuh, semua pipeline gagal. Snapshot
harian juga menahan ruang dari berkas yang sudah dihapus.

Pemakaian bio-pool per folder: DR 7,7 TB · Backup 3,0 TB · JY 2,6 TB ·
IAM 1,6 TB · AGO 1,3 TB · KIL 572 GB · snapshot ±1 TB.

## 2. Mount NFS di node

```
# /etc/fstab
192.168.30.2:/bio-pool             /media/bio-pool        nfs defaults,nofail,hard,timeo=600,retrans=2,_netdev 0 0
192.168.30.2:/media/t4/NVME-3.6TB  /media/t4-storage-nvme nfs defaults,nofail,soft,timeo=30,retrans=2,x-systemd.automount,x-systemd.requires=network-online.target,x-systemd.idle-timeout=10min 0 0
```
Ekspor di t4-storage (`/etc/exports`) mengizinkan seluruh 192.168.30.0/24.
Node baru cukup diberi IP di jaringan itu.

## 3. Prosedur: membuka & memverifikasi disk bekas Windows

Dipakai untuk 5 SSD lab (dynamic disk Windows, volume spanned).

1. **Kunci read-only dulu** sebelum apa pun:
   ```bash
   for d in sdX sdY ...; do blockdev --setro /dev/$d /dev/${d}3; done
   ```
2. Baca metadata dan rangkai volume: `ldmtool scan`, `ldmtool show`,
   `ldmtool create volume <diskgroup> Volume1`.
3. Mount read-only: `mount -t ntfs3 -o ro /dev/mapper/ldm_vol_... /mnt/<nama>`.
4. Verifikasi backup dalam 3 tahap:
   - daftar berkas + ukuran byte (`find -printf '%s|%p'`, lalu `diff`)
   - samakan yang kurang (`rsync -rt`, **tanpa** `--delete`)
   - **isi byte-per-byte** (`cmp -s` per berkas). Hanya tahap ini yang
     menjamin backup identik. Kecepatan dibatasi HDD (±250 MB/s → 3,4 TB ≈ 4 jam)
5. Tulis log yang mudah dibaca di samping folder backup (bukan di dalamnya).
6. **Baru format disk sumber setelah tahap isi selesai dengan 0 beda.**

Hasil untuk SSD lab: HDD-POD5 `backupdata-sequencehddlab` identik 100%
(11.838 berkas dibandingkan isi, 0 beda). Salinan ketiga di 80-Storage1.
Log: `/media/t4/HDD-POD5/LOG-verifikasi-backup-sequencehddlab.txt`.

Sebelum SSD diformat, lepas dulu:
```bash
umount /mnt/ssd-spanned-F
dmsetup remove ldm_vol_DESKTOP-2QLDRU6-Dg0_Volume1
```

## 4. Catatan t4-storage

- IP manajemen berubah dari 192.168.18.193 ke **192.168.18.202** (kemungkinan
  DHCP). Perlu IP statis / reservasi DHCP dan inventaris diperbarui.
- Akses root dari Proxmox lewat jaringan storage `192.168.30.2`.
