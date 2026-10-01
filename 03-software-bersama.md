# 03 — Software bersama

Semua yang dipakai lebih dari satu node diletakkan di **bio-pool**
(`/media/bio-pool/software/`), karena bio-pool di-mount di semua node.

---

## 1. Nextflow

- Terpasang di `/usr/local/bin/nextflow` (VM 200).
- Versi dikunci per run dengan `NXF_VER=25.10.2` di pembungkus/formulir, agar
  hasil bisa diulang walau Nextflow terpasang diperbarui.
- Kepala Nextflow berjalan sebagai job Slurm kecil di compute002
  (`cpu-only`, QoS `besar`, 2 inti, 4 GB, 7 hari). Tahap-tahap pipeline
  memesan resource sendiri.

## 2. Profil Slurm bersama

Berkas: `/media/bio-pool/software/nextflow/slurm-bersama.config`

| Isi | Alasan |
|---|---|
| `errorStrategy` retry untuk exit `null`, `Integer.MAX_VALUE`, 1, 104, 134, 137, 139, 140, 143, 247; maksimal 5×, jeda 1→5 menit | Node mati, job terbunuh OOM/SIGTERM, controller sempat mati |
| `exitReadTimeout = 30 min` | Jangan cepat menyimpulkan job hilang saat controller/NFS lambat |
| `submitRateLimit 10/1min`, `pollInterval 30 sec`, `queueStatInterval 1 min` | Tidak membanjiri controller |
| `trace/report/timeline/dag overwrite = true` | Laporan lama tidak menggagalkan run di langkah terakhir |

Cara memakai: tambahkan di **akhir** `conf/cluster.config` pipeline:
```groovy
profiles { slurm { includeConfig "/media/bio-pool/software/nextflow/slurm-bersama.config" } }
```
Mengubah berkas ini berlaku untuk run **berikutnya** semua pipeline.

## 3. Env conda

- Lokasi: `/media/bio-pool/software/conda-envs/` (miniforge di
  `/media/bio-pool/software/miniforge3/`).
- Satu env per user/pipeline; env otomatis Nextflow (`env-<hash>`) juga di sini.
- Resep tiap env disimpan di repo `scripts/conda-envs/`: `*.explicit.txt` +
  **`*.pip.txt`** (paket pip tidak tercatat di resep explicit).
- Env lama di disk lokal **tidak dihapus**, hanya tidak dipakai lagi.

Membangun ulang env dari resep:
```bash
conda create -p /media/bio-pool/software/conda-envs/<nama> --file <nama>.explicit.txt
/media/bio-pool/software/conda-envs/<nama>/bin/python -m pip install --no-deps -r <nama>.pip.txt
```

## 4. Container (Singularity)

- singularity-ce 4.1.1 terpasang di compute001 dan compute002.
- Cache image bersama per user, contoh `/media/bio-pool/KIL/.singularity/cache`
  (`NXF_SINGULARITY_CACHEDIR`).

## 5. Ketergantungan yang masih di disk lokal HPC-GPU

`/mnt/scratch` hanya ada di HPC-GPU. Pipeline yang memanggil alat dari sana
**hanya bisa jalan di compute001**:

| Pipeline | Yang masih di `/mnt/scratch` |
|---|---|
| Human Genomics (angelo) | `somalier`, miniconda angelo (`bcftools`, `samtools`, `python3`, conda), venv PharmCAT (di folder KIL) |

Untuk membuatnya bisa jalan di node mana pun: salin alat ke bio-pool, bangun
ulang venv PharmCAT di bio-pool, arahkan PATH ke env bio-pool
(perkiraan 1–1,5 jam kerja, lihat dokumen 09).
