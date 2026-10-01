# 08 — Pengguna eksternal: mengembangkan pipeline

Untuk pihak luar yang **membangun pipeline** di cluster ini. Setup akun,
Tailscale, kuota, dan offboarding sama dengan dokumen 07; dokumen ini
menambahkan hal yang khusus untuk pengembangan.

---

## 1. Lingkungan kerja yang disediakan

| Kebutuhan | Disediakan |
|---|---|
| Terminal | Portal → Clusters → Shell, atau SSH lewat Tailscale |
| Editor | Editor berkas di portal. *VS Code di browser (code-server) belum dipasang; bisa ditambahkan sebagai aplikasi portal.* |
| Git & internet | Dari VM 200 (clone/push ke GitHub/GitLab, unduh tools/database) |
| Nextflow | `/usr/local/bin/nextflow`, kunci versi dengan `NXF_VER` |
| Conda / mamba | Miniforge di bio-pool; env pribadi di folder sendiri |
| Container | singularity-ce 4.1.1 di kedua node |
| Data uji | Disediakan oleh pengembang sendiri, di folder sendiri |

## 2. Jatah awal yang disarankan

Pengembangan butuh resource **kecil tapi sering** (banyak run uji pendek,
sesekali run penuh).

| Resource | Usulan |
|---|---|
| CPU | 16–32 inti sekaligus |
| RAM | 64–128 GB |
| Storage | 500 GB – 1 TB (kode, data uji, `work/`) |
| Waktu job | 2 hari |
| GPU | Hanya jika pipeline memang memakai GPU |

Run uji kecil diarahkan ke `cpu-only` (compute002) atau `debug` (30 menit).

## 3. Standar pipeline agar nanti bisa dipakai tim

Pipeline yang dikembangkan di sini harus memenuhi ini supaya bisa langsung
diberi formulir portal:

1. **Kode di Git** sejak awal; tim mendapat akses repo.
2. **Profil Slurm** `slurm` di `conf/cluster.config` yang diakhiri include
   profil bersama:
   ```groovy
   profiles { slurm { includeConfig "/media/bio-pool/software/nextflow/slurm-bersama.config" } }
   ```
3. **Resource per tahap** lewat `withName`/`withLabel` (cpus, memory, time),
   diukur dari `sacct`, bukan tebakan.
4. **Tidak ada path ke disk lokal node** (misalnya `/mnt/scratch/...`). Semua
   alat lewat conda env / container yang bisa dibuat ulang, atau di bio-pool.
5. **Software dideklarasikan**: `conda` (berkas `.yml`) atau `container`
   per proses; versi dikunci.
6. **Profil `test`** dengan data kecil yang selesai < 30 menit.
7. **Parameter input jelas** (`--input`, `--outdir`) dan README cara menjalankan.
8. Tidak menulis ke luar `--outdir` dan `-work-dir`.

Contoh perintah uji:
```bash
export NXF_VER=25.10.2
nextflow run main.nf -profile test,slurm -work-dir /media/bio-pool/EXT/<user>/work_test
```

## 4. Serah terima di akhir proyek

- [ ] Repo Git diserahkan (atau di-fork ke organisasi tim), termasuk lisensi.
- [ ] README: cara install, input, output, contoh perintah.
- [ ] Resep env / daftar container dengan versi.
- [ ] Hasil pengukuran resource per tahap (trace Nextflow).
- [ ] Data uji kecil yang boleh disimpan tim.
- [ ] Data lain di `/media/bio-pool/EXT/<user>` dihapus atau dipindahkan.

Setelah serah terima, IT membuatkan formulir portal untuk pipeline tersebut
(dokumen 04 §2).
