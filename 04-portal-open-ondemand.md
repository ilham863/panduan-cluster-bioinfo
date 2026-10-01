# 04 — Portal Open OnDemand

Open OnDemand 4.1.7 di VM 200. Alamat: **https://dev-bioinfo.basa-minnow.ts.net**
(hanya lewat Tailscale). Runbook rinci: `docs/sop/runbook-portal-dan-cluster.md`.

---

## 1. Komponen

| Bagian | Lokasi (VM 200) |
|---|---|
| Config portal | `/etc/ood/config/ood_portal.yml` (**berisi hash password & passphrase: jangan disalin ke repo**) |
| Login | Dex, akun statis per user (`user@bioinfo.local`) |
| Sertifikat | `/etc/ssl/ondemand/`, `tailscale cert`, timer `ood-cert-renew.timer` mingguan |
| Cluster | `/etc/ood/config/clusters.d/bioinfo.yml` |
| Halaman depan | `ondemand.d/ondemand.yml` (pinned apps, help menu), `motd-bioinfo.md` |
| Pintasan folder | `apps/dashboard/initializers/ood.rb` (dibungkus `after_initialize`) |
| Formulir | `/var/www/ood/apps/sys/jalankan-*/` |
| Resolusi nama lokal | `/etc/hosts`: `127.0.0.1 dev-bioinfo.basa-minnow.ts.net` |

## 2. Formulir "Jalankan Pipeline"

| Formulir | Pemilik | Pengendali | Tahap berjalan di |
|---|---|---|---|
| Metagenome isolat | kila | compute002 | compute001 (Slurm) |
| PRS (pgsc_calc) | kila | compute002, 16 inti / 64 GB | compute002 (lokal dalam alokasi) |
| Human Genomics (ancestry) | angelo | compute002 | compute001 (Slurm, config dari formulir) |

Semua formulir punya mode **Cek dulu** (hanya `-preview`, tidak menjalankan apa
pun) dan **Jalankan pipeline**, serta `-resume` agar bisa dilanjutkan.

Struktur satu formulir:

| Berkas | Isi |
|---|---|
| `manifest.yml` | nama, ikon, `category: "Jalankan Pipeline"` |
| `form.yml.erb` | daftar sampel dibaca otomatis dari folder input (`Dir.glob`) |
| `submit.yml.erb` | partisi, QoS, CPU/RAM pengendali |
| `template/script.sh.erb` | perintah Nextflow; **izin 755 wajib** |

Langkah menambah formulir: salin formulir yang ada → sesuaikan → tambah ke
`pinned_apps` → uji mode **Cek dulu** sebagai pemilik data → umumkan.

## 3. Menambah user portal

1. Akun Linux ada di VM 200 dengan UID/GID sama seperti cluster.
2. Buat hash bcrypt password, tambahkan ke `dex: static_passwords:` di
   `ood_portal.yml`.
3. Terapkan **dengan kedua opsi ini** (tanpanya semua orang tidak bisa login):
   ```bash
   /opt/ood/ood-portal-generator/sbin/update_ood_portal --insecure --force
   systemctl restart ondemand-dex apache2
   ```
4. Buat kunci SSH internal user (agar menu Shell tidak minta password).
5. Kirim password **secara pribadi**, jangan di grup chat.

## 4. Halaman Status Cluster

- Alamat: **https://dev-bioinfo.basa-minnow.ts.net:8088** (juga di menu Help).
- Kode `/opt/status-cluster/`, data `/var/lib/status-cluster/`, user `statuscluster`.
- Layanan: `status-cluster-kumpul` (mengumpulkan data tiap ±10 detik) dan
  `status-cluster-web` (`127.0.0.1:8089`), di belakang Apache HTTPS :8088.
- Isi: status per node, kuota QoS, job jalan/antre dengan alasan, riwayat
  reboot 7 hari, job gagal.
- Sumber di repo: `scripts/monitoring/status-cluster/`.

## 5. Log

| Aktivitas | Lokasi |
|---|---|
| Akses/unduh berkas lewat portal | `/var/log/apache2/dev-bioinfo.basa-minnow.ts.net_access_ssl.log` |
| Per user | `/var/log/ondemand-nginx/<user>/access.log` |
| Login SSH | `journalctl -u ssh` |
| Run dari formulir | `~<user>/ondemand/data/sys/dashboard/batch_connect/sys/<app>/output/<id>/output.log` |
