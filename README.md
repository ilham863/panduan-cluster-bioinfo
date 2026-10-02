# Panduan Cluster Bioinformatika

Dokumentasi infrastruktur cluster: apa yang sudah dibangun, cara setupnya, cara
memakainya, dan cara menerima pengguna eksternal.


---

## Gambaran singkat

```
                         Tailscale (VPN)
  Laptop tim / eksternal ───────────────► dev-bioinfo (VM 200)
                                          ├─ Portal Open OnDemand (https :443)
                                          ├─ Halaman Status Cluster (https :8088)
                                          ├─ SSH
                                          └─ compute002 (Slurm, 88 inti, 112 GB)
                                                  │
                     Slurm controller ◄───────────┤
                     VM 100 "pipeline"            │
                                                  ▼
                                          compute001 = HPC-GPU
                                          (256 inti, 1 TB RAM, A100)

  Penyimpanan bersama (NFS dari t4-storage, jaringan 192.168.30.0/24):
    /media/bio-pool         SSD ZFS  - data kerja & pipeline semua user
    /media/t4-storage-nvme  NVMe     - output sequencer
```

Prinsip utama:
1. **User tidak mengatur CPU/RAM sendiri.** Pipeline dijalankan dari formulir
   portal; ukuran tiap tahap sudah diatur IT dari pengukuran nyata.
2. **Semua pekerjaan berat lewat Slurm.** Tidak ada yang jalan langsung di node.
3. **Satu profil Nextflow bersama** untuk retry dan toleransi gangguan.
4. **Software & env di bio-pool**, supaya terlihat dari semua node.
5. **Dijalankan sebagai pemilik data**, tidak pernah sebagai akun IT.

---

## Isi

| # | Dokumen | Untuk siapa |
|---|---|---|
| 01 | [Catatan implementasi](01-catatan-implementasi.md): apa yang dikerjakan, kapan, masalah yang ditemui | IT, atasan |
| 02 | [Slurm dua node](02-slurm-dua-node.md): registrasi compute node, partisi, QoS | IT |
| 03 | [Software bersama](03-software-bersama.md): Nextflow, profil bersama, conda, singularity | IT |
| 04 | [Portal Open OnDemand](04-portal-open-ondemand.md): login, formulir, halaman status | IT |
| 05 | [Penyimpanan](05-penyimpanan.md): bio-pool, NFS, backup & verifikasi data | IT |
| 06 | [Pemakaian tim bioinformatika](06-tim-bioinformatika.md) | Tim bioinfo |
| 07 | [Pengguna eksternal: menjalankan job](07-eksternal-menjalankan-job.md) | IT, admin riset |
| 08 | [Pengguna eksternal: mengembangkan pipeline](08-eksternal-develop-pipeline.md) | IT, admin riset |
| 09 | [Rekomendasi & pekerjaan tertunda](09-rekomendasi-dan-tertunda.md) | IT, atasan |
| 10 | [Keamanan sebelum upload ke GitHub](10-keamanan-sebelum-upload.md) | IT |

Dokumen terkait yang sudah ada di repo `infra-docs`:
- `docs/sop/runbook-portal-dan-cluster.md` (runbook admin rinci)
- `docs/panduan/panduan-tim-portal-ondemand.md` (panduan pengguna portal)
- `scripts/ood/`, `scripts/monitoring/status-cluster/`, `scripts/conda-envs/`
  (salinan konfigurasi tanpa rahasia)
