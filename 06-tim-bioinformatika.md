# 06 — Pemakaian oleh tim bioinformatika

Untuk: dewi, kila, angelo, jeffrey, reinhart. Panduan langkah demi langkah
portal ada di `docs/panduan/panduan-tim-portal-ondemand.md`.

---

## 1. Cara kerja sehari-hari

1. Aktifkan **Tailscale**, buka **https://dev-bioinfo.basa-minnow.ts.net**.
2. **Jalankan Pipeline** → pilih pipeline → pilih sampel.
3. Mode **Cek dulu** → Launch. Kalau `output.log` berakhir `BERHASIL`, ulangi
   dengan **Jalankan pipeline**.
4. Pantau di **My Interactive Sessions** (log), **Jobs → Active Jobs**, atau
   **Help → Status Cluster (detail per node)**.
5. Ambil hasil di menu **Files**.

Tidak perlu mengisi CPU/RAM. Browser boleh ditutup; run tetap jalan.

## 2. Kalau menjalankan dari terminal

- Selalu di dalam **tmux** (`tmux new -s nama`, keluar `Ctrl-b d`).
- Selalu lewat Slurm: `-profile slurm`, jangan menjalankan tahap berat langsung
  di node. Pekerjaan di luar Slurm tidak terlihat penjadwal dan bisa membuat
  node kehabisan RAM.
- Satu **work dir dan folder peluncuran per sampel**, agar run paralel tidak
  saling mengunci dan bisa di-resume sendiri-sendiri.
- Kunci versi Nextflow: `export NXF_VER=25.10.2`.

## 3. Aturan

| Aturan | Alasan |
|---|---|
| Jangan menjalankan dua run untuk sampel yang sama bersamaan | Saling mengunci cache Nextflow |
| Jangan mengubah `conf/cluster.config` / menghapus include profil bersama tanpa kabar ke IT | Formulir dan retry otomatis bergantung padanya |
| Pasang paket baru → kabari IT | Resep env perlu diperbarui |
| Hapus `work_*` setelah hasil dicek | bio-pool tinggal ±2 TB |
| Data pasien hanya di folder tim | Privasi |

## 4. Kalau ada masalah

| Gejala | Artinya |
|---|---|
| Status "Queued"/antre lama | Cluster atau kuota penuh, bukan error. Cek halaman status: alasan antre tertulis di sana |
| `QOSGrpCpuLimit` | Kuota total QoS `normal` habis dipakai user lain |
| `QOSMaxJobsPerUserLimit` | Jatah jumlah run Anda sedang penuh; run berikutnya mulai otomatis |
| `output.log` berakhir `GAGAL` | Kirim isi log ke IT |
| Tidak bisa membuka portal | Tailscale belum aktif, atau Secure DNS browser (lihat panduan portal) |
