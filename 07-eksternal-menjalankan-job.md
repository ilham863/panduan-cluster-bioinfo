# 07 — Pengguna eksternal: menjalankan job

Untuk pihak luar yang memakai cluster (tidak ditagih). Prinsip: **hanya bisa
masuk lewat dev-bioinfo, hanya melihat foldernya sendiri, dibatasi kuota, dan
otomatis berakhir.**

---

## 1. Sebelum menerima: pastikan data tim tidak terbaca

**Wajib dikerjakan dulu.** Hasil cek 1 Oktober 2026: semua folder tim di
bio-pool (`DR`, `KIL`, `AGO`, `JY`, `IAM`, `Backup`) berizin `drwxrwsr-x`, dan
`/media/t4-storage-nvme/output` berizin `drwxrwxr-x`. Artinya **semua user di
server bisa membacanya**. Pengguna eksternal akan bisa membaca data tim,
termasuk data pasien, kalau ini tidak ditutup.

```bash
# lihat folder yang terbuka untuk "others"
find /media/bio-pool /media/t4-storage-nvme -maxdepth 2 -type d -perm -o+r -printf '%m %u:%g %p\n'
# tutup untuk others (tim tetap bisa lewat grup bioinfo)
chmod o-rwx /media/bio-pool/{DR,KIL,AGO,JY,IAM,Backup} /media/t4-storage-nvme/output
```
Lakukan di t4-storage (sumber NFS), lalu cek bahwa tim tetap bisa bekerja
(semua anggota tim ada di grup `bioinfo`). Folder `IAM` grupnya tidak dikenal di
VM 200; periksa grupnya sebelum menutup izin.
Uji dengan akun eksternal: `sudo -u <ext> ls /media/bio-pool/KIL` harus
`Permission denied`.

## 2. Data yang diminta dari pihak eksternal

Nama + email tiap pengguna, proyek & periode, jenis/ukuran data dan ada data
pasien atau tidak, kebutuhan storage, CPU/RAM per job, GPU/CUDA, software.
(Lihat pesan ke admin riset.)

## 3. Langkah setup

### 3.1 Tailscale
Pilihan paling sederhana dan aman: **Share** mesin `dev-bioinfo` saja ke email
pengguna (Tailscale admin → Machines → dev-bioinfo → Share). Pengguna hanya
melihat mesin itu, tidak melihat t4-storage, Proxmox, HPC-GPU, atau BMC.

Tambahkan ACL agar hanya port yang perlu yang terbuka (contoh; sesuaikan dengan
policy yang sudah ada, jangan menimpa aturan tim):
```json
{
  "groups":    { "group:eksternal": ["nama@institusi.ac.id"] },
  "tagOwners": { "tag:portal": ["autogroup:admin"] },
  "acls": [
    { "action": "accept", "src": ["group:eksternal"], "dst": ["tag:portal:22,443,8088"] }
  ]
}
```
(`dev-bioinfo` diberi tag `tag:portal`.) Cabut akses dengan menghapus share /
user setelah proyek selesai.

### 3.2 Akun Linux (di **semua** node: VM 200 dan HPC-GPU, UID sama)
```bash
groupadd -g 4000 eksternal                         # sekali saja
useradd -m -u <UID> -g eksternal -e 2026-12-31 -s /bin/bash <user>   # -e = tanggal kedaluwarsa
```
- **Bukan** anggota grup `bioinfo`.
- Login SSH hanya dengan kunci (tambahkan kunci publik mereka ke
  `~/.ssh/authorized_keys`).

### 3.3 Folder kerja dan kuota
```bash
mkdir -p /media/bio-pool/EXT/<user>
chown <user>:eksternal /media/bio-pool/EXT/<user>; chmod 700 /media/bio-pool/EXT/<user>
# di t4-storage: kuota per user pada dataset bio-pool
zfs set userquota@<user>=1T bio-pool
zfs get userused@<user>,userquota@<user> bio-pool
```
Karena bio-pool tinggal ±2 TB, pertimbangkan area eksternal di storage lain
(misalnya 5 SSD lab yang sudah dibackup), lalu di-mount ke node.

### 3.4 Slurm: akun dan QoS khusus
```bash
sacctmgr -i add qos eksternal set GrpTRES=cpu=64,mem=256G,gres/gpu=0 \
  MaxTRESPU=cpu=32,mem=128G MaxWall=2-00:00:00 MaxJobsPU=10 Priority=5
sacctmgr -i add account eksternal Description="pengguna eksternal"
sacctmgr -i add user <user> account=eksternal qos=eksternal defaultqos=eksternal
```
Angka di atas adalah usulan awal; sesuaikan dengan jawaban kebutuhan mereka.
Kalau butuh GPU, ubah `gres/gpu` dan arahkan ke partisi GPU yang ditentukan.

### 3.5 Akun portal
Ikuti dokumen 04 §3 (hash password, `update_ood_portal --insecure --force`,
kunci SSH internal). Kirim password secara pribadi.

### 3.6 Uji sebelum diserahkan
```bash
sudo -u <user> ls /media/bio-pool/KIL                         # harus ditolak
sudo -u <user> sbatch -A eksternal --qos=eksternal -c 64 --wrap=true --test-only   # harus ditolak (melebihi kuota)
sudo -u <user> srun -A eksternal --qos=eksternal -p cpu-only -c 2 hostname          # harus jalan
```

## 4. Aturan untuk pengguna eksternal

- Bekerja hanya di `/media/bio-pool/EXT/<user>`.
- Semua pekerjaan lewat Slurm (`sbatch`/`srun`/Nextflow `-profile slurm`).
- Tidak menyimpan data pasien kecuali ada perjanjian (NDA/DUA).
- Data dihapus/diserahkan saat akses berakhir.

## 5. Mengakhiri akses (offboarding)

1. Hapus share/user di Tailscale.
2. Matikan akun: `usermod -L -e 1 <user>` di semua node; hapus dari Dex.
3. `sacctmgr -i remove user <user> account=eksternal`.
4. Serahkan/hapus data di `/media/bio-pool/EXT/<user>` sesuai kesepakatan.
