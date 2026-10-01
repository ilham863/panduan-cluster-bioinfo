# 09 — Rekomendasi & pekerjaan tertunda

Urut dari yang paling mendesak.

---

## Prioritas tinggi

| # | Masalah | Dampak | Tindakan |
|---|---|---|---|
| 1 | **HPC-GPU reboot mendadak** 6× dalam 7 hari, tanpa shutdown bersih | Job terbunuh (NODE_FAIL), run diulang | Cek BMC 192.168.18.119: SEL, *Last Power Event*, status PSU, suhu. Perbaiki pengumpulan sensor IPMI (data berhenti sejak 24/9) |
| 2 | **bio-pool 87% penuh** (sisa 2,26 TB) | ZFS melambat; kalau penuh semua pipeline gagal | Rapikan `Backup/` (3 TB) dan `work_*` lama dengan persetujuan pemilik; hapus snapshot terkait; target < 80% |
| 3 | **Folder tim bisa dibaca semua user** (`o+rx`) | Data pasien terbaca oleh siapa pun yang punya akun | `chmod o-rwx` folder tim (dokumen 07 §1). **Wajib sebelum ada pengguna eksternal** |
| 4 | **Kuota QoS `normal` total 128 inti** padahal cluster 344 inti | Job antre (`QOSGrpCpuLimit`) walau node kosong | `sacctmgr -i modify qos normal set GrpTRES=cpu=344,gres/gpu=1` (batas per user tetap 128) |
| 5 | **96-Storage 99% penuh** | Penulisan ke sana gagal | Pindahkan/rapikan isi |

## Prioritas menengah

| # | Pekerjaan | Catatan |
|---|---|---|
| 6 | Salin `slurm.conf` terbaru ke HPC-GPU | Sudah disiapkan di `/home/ilham/slurm.conf.baru`; `sudo cp` lalu `systemctl restart slurmd` saat antrean sepi |
| 7 | IP t4-storage berubah (.193 → .202) | Jadikan statis / reservasi DHCP; perbarui inventaris |
| 8 | Pipeline angelo masih bergantung `/mnt/scratch` HPC-GPU | Pindahkan somalier, venv PharmCAT, alat miniconda ke bio-pool (±1–1,5 jam) agar bisa jalan di compute002 |
| 9 | Pipeline angelo kehilangan `conf/` (1/10) | Sepakati: formulir portal sebagai cara resmi, atau kembalikan `conf/cluster.config` + include profil bersama |
| 10 | Konfigurasi dewi & angelo belum memakai env conda baru | dewi perlu persetujuan; angelo setelah run selesai |
| 11 | Formulir portal untuk pipeline dewi | Plant-genome & chromosome-scaffolding |
| 12 | Data T4 mentah di formulir metagenome | Tambah pilihan "sumber data: T4 fastq_pass" |
| 13 | HG002_2025: berkas referensi 1KG belum terpublikasi ("Timed out while waiting to publish outputs") | Ulang dengan `-resume` (semua dari cache) |

## Kapasitas yang belum terpakai

| Perangkat | Keadaan | Usulan |
|---|---|---|
| **A100 kedua** di HPC-GPU | Sengaja dimatikan di driver | Kalau diaktifkan: `Gres=gpu:a100:2` di `slurm.conf` + `gres.conf` |
| **V100** (beberapa kartu) | Belum tercatat di inventaris | Jadikan node GPU baru (compute003, partisi `gpu-v100`), cocok untuk pengguna eksternal |
| 2× Tesla T4 di Proxmox | Menganggur (driver nouveau) | Passthrough ke VM 200 |
| 3× Tesla T4 di t4-storage | Tidak dikelola Slurm | Pertimbangkan untuk basecalling |
| 5 SSD lab (±7,5 TB) | Backup sudah terverifikasi | Format ulang → area kerja pengguna eksternal / scratch |

## Perbaikan log & keamanan (opsional)

- Log nama berkas scp/sftp (`sftp-server -l INFO`), format Apache dengan `%I`,
  masa simpan log 90–365 hari.
- VS Code di browser (code-server) sebagai aplikasi portal.
- Tandai panduan PDF lama sebagai usang.
- Bagikan password portal secara pribadi ke tiap user.
