# 01 — Catatan implementasi

Apa yang dikerjakan, urut waktu, beserta masalah yang ditemui dan cara
mengatasinya. Waktu dalam WIB.

---

## Ringkasan

| Bidang | Sebelum | Sesudah |
|---|---|---|
| Node komputasi | 1 node (HPC-GPU) | **2 node**: compute001 (HPC-GPU) + compute002 (VM 200) |
| Cara menjalankan pipeline | SSH + ketik perintah panjang, sering minta IT `su` ke akun user | **Formulir portal** di browser, sebagai akun sendiri |
| Ukuran CPU/RAM | Ditebak user | **Diatur IT** per tahap dari pengukuran `sacct` |
| Gangguan node/controller | Run Nextflow langsung gagal | **Retry otomatis** sampai 5× (profil bersama) |
| Env conda | Di disk lokal HPC-GPU (`/mnt/scratch`) | **Di bio-pool**, terlihat dari semua node |
| Pemantauan | Halaman di HPC-GPU (ikut mati saat node reboot) | **Halaman status di VM 200**, detail per node, riwayat reboot |
| Akses dari luar kantor | – | **Tailscale** + HTTPS (sertifikat Tailscale) |

---

## 28 September 2026

### Registrasi VM 200 sebagai node kedua (compute002)
- Pasang `slurmd` + `munge` di VM 200, salin `munge.key` dan `slurm.conf` dari
  controller (VM 100).
- Tambah `NodeName=compute002 ... CPUs=96 CoreSpecCount=8 RealMemory=114688`
  dan partisi baru `cpu-only`.
- **Masalah:** UID user `slurm` dari paket apt = 64030, cluster memakai 9999 →
  slurmd menolak perintah controller. **Perbaikan:** `usermod/groupmod` ke 9999.
- **Masalah:** `gres.conf` (GPU) tidak boleh ikut disalin ke node tanpa GPU.
- Kunci SSH root VM 100 → VM 200 dipasang agar konfigurasi bisa disinkronkan.

### Env conda dipindah ke bio-pool
- Env tim dibangun ulang di `/media/bio-pool/software/conda-envs/`
  (dewi, ancestry_env, ancestry_pca, dan env otomatis Nextflow).
- **Env lama tidak dihapus** (tetap di tempat lama sebagai cadangan).
- **Masalah:** resep `conda list --explicit` tidak mencatat paket pip
  (NanoPlot, eggnog-mapper, reportlab hilang). **Perbaikan:** pasang ulang dari
  berkas `*.pip.txt`; resep disimpan di repo `scripts/conda-envs/`.

### Pipeline metagenome (kila) dirapikan
- QoS `normal`, batas waktu 2 hari, ukuran METAFLYE/GTDBTK disesuaikan.
- Pembungkus `run-metagenome.sh`: Nextflow dikunci `NXF_VER=25.10.2`, folder
  peluncuran **per barcode** (agar run paralel tidak saling mengunci), mode
  `CEK=1` (hanya `-preview`).
- Bug argumen GTDBTK di `main.nf` diperbaiki (backup `.bak-2026-09-28-*`).

### Portal Open OnDemand 4.1.7 di VM 200
- Login Dex dengan akun statis per user (`user@bioinfo.local`).
- HTTPS dengan sertifikat Tailscale, diperbarui otomatis mingguan.
- Halaman depan: panduan (MOTD), pintasan folder per user, tombol
  "Jalankan Pipeline", menu Help → standar resource & status cluster.
- Kunci SSH internal per user, agar menu Shell tidak minta password.
- Formulir pertama: **Metagenome isolat** (diuji: job 447 selesai).

---

## 29 September 2026

### Insiden: controller Slurm mati ±4 menit (00:32–00:37)
- Penyebab: baris `NodeName=compute002` tercatat dua kali di `slurm.conf` →
  `slurmctld` gagal start.
- Dampak: run ULTIMA angelo dihentikan Nextflow walau job-nya sukses.
- Perbaikan: baris ganda dihapus; dibuat **profil Slurm bersama** dengan
  `exitReadTimeout 30 min` dan retry untuk kasus "job hilang".

### Profil Nextflow bersama
- `/media/bio-pool/software/nextflow/slurm-bersama.config`, di-include di akhir
  profil `slurm` pipeline kila, dewi (2 pipeline), dan angelo.

### HPC-GPU tidak stabil
- Reboot mendadak 6× dalam 7 hari (4× dalam 10 jam: 28/9 16:44, 17:26, 22:56;
  29/9 02:17). Tidak ada shutdown bersih; sensor IPMI berhenti update sejak 24/9.
- Dampak: job angelo terbunuh (NODE_FAIL), run diulang dengan `-resume`.
- **Belum diselesaikan**: perlu cek SEL/PSU di BMC (lihat dokumen 09).

### Halaman Status Cluster (port 8088)
- Dibangun ulang di VM 200 (tetap hidup saat HPC-GPU mati): status per node,
  kuota QoS, job berjalan/antre dengan alasan dalam bahasa biasa, riwayat reboot
  7 hari, job gagal.
- **Masalah:** browser tidak bisa membuka `http://...:8088` karena domain
  `.ts.net` termasuk daftar HSTS preload (dipaksa https). **Perbaikan:** Apache
  melayani HTTPS di :8088, aplikasi hanya mendengar `127.0.0.1:8089`.

### Formulir PRS (pgsc_calc) untuk kila
- Awalnya hanya bisa di HPC-GPU (singularity & pipeline di `/mnt/scratch`).
- Dipindah ke compute002: singularity-ce 4.1.1 dipasang di VM 200, pipeline
  pgsc_calc v2.3.0 disalin ke `/media/bio-pool/KIL/.nextflow`.

### Data SSD lab (t4-storage)
- 5 SSD bekas PC Windows dipasang di t4-storage. Ternyata **volume spanned**
  (bukan RAID 0) dynamic disk Windows, 7,5 TB, isi 3,4 TB.
- Dibuka **read-only** dengan `ldmtool` + `blockdev --setro`, di-mount di
  `/mnt/ssd-spanned-F`.
- Backup di HDD-POD5 diverifikasi: daftar & ukuran berkas, penyamaan 2 log
  MinKNOW + folder sistem Windows, lalu **perbandingan isi byte-per-byte
  (`cmp`) 11.838 berkas: 0 beda**. Kesimpulan: identik 100%.
- Salinan ketiga ke 80-Storage1 (rsync selesai 30/9 01:29, 12.905 item dan
  3.449.334.824.498 byte sama; isi belum dibandingkan byte-per-byte).

---

## 30 September – 1 Oktober 2026

- **Mount NVMe t4 di VM 200** (`/media/t4-storage-nvme`, fstab, opsi sama dengan
  HPC-GPU) agar pipeline dari portal bisa membaca output sequencer.
- **Formulir Human Genomics (ancestry)** untuk angelo. Pipeline angelo pada
  1/10 diubah ke input VCF dan folder `conf/` (config Slurm) dihapus, sehingga
  formulir membawa config Slurm sendiri (`slurm-humangenomics.config`).
- Kepemilikan berkas sisa uji IT di folder angelo dikembalikan ke angelo.
- Run tim berjalan di 2 node: metagenome T4 9 barcode (kila), PRS (kila),
  ancestry ER2 & HG002 (angelo).

---

## Pelajaran (jangan diulang)

1. **Jangan menjalankan pipeline dengan akun IT di folder user.** Berkas
   `.nextflow/`, log, atau folder hasil milik akun yang salah memblokir pemilik
   aslinya. Selalu `sudo -u <pemilik>`.
2. **Cek duplikat sebelum restart `slurmctld`**, dan ubah `slurm.conf` saat
   antrean sepi.
3. **UID `slurm` harus sama** di semua node (9999).
4. **Resep conda `--explicit` tidak memuat paket pip.** Simpan juga `pip freeze`.
5. **Domain `.ts.net` wajib HTTPS** untuk semua layanan web.
6. **Alat yang hanya ada di `/mnt/scratch` (disk lokal HPC-GPU) mengikat pipeline
   ke HPC-GPU.** Pindahkan ke bio-pool kalau ingin bisa jalan di node lain.
