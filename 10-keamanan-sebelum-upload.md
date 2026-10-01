# 10 — Keamanan sebelum upload ke GitHub

Baca ini sebelum folder panduan (atau perubahan lain di repo `infra-docs`)
di-upload.

---

## 1. Keadaan repo saat ini (cek 1 Oktober 2026)

| Hal | Temuan |
|---|---|
| Repo | `ilham863/insfra-docs` di GitHub |
| **Visibilitas** | **PUBLIC**: bisa dibaca siapa pun di internet |
| Sudah ter-push | `main` + 3 branch lain (20 commit di `main`) |
| Belum di-commit | 3 berkas diubah, ±14 folder/berkas baru (docs, scripts) |
| Rahasia (password, hash, kunci privat, token) | **Tidak ditemukan**, baik di berkas kerja, riwayat commit, maupun folder panduan ini (pemindaian pola; lihat §4) |
| Informasi internal yang **sudah publik** di `origin/main` | IP internal 192.168.x.x (15 berkas), IP Tailscale 100.x (2 berkas), info BMC/IPMI (19 berkas), nomor seri perangkat (20 berkas) |

Jadi tidak ada password yang bocor, tetapi **peta jaringan internal sudah
terbuka untuk umum**: alamat server, BMC, nama host, model dan nomor seri
perangkat. Itu memudahkan orang yang berniat menyerang.

## 2. Keputusan yang perlu diambil

| Pilihan | Kelebihan | Kekurangan |
|---|---|---|
| **A. Jadikan repo private** (disarankan) | Dokumentasi bisa lengkap dan praktis | Yang sudah publik mungkin sudah tersalin/terindeks; tetap anggap sudah diketahui |
| B. Tetap public, samarkan | Aman untuk umum | Semua IP, hostname, username, tailnet harus diganti placeholder; dokumen kurang praktis; riwayat lama tetap berisi data asli |
| C. Repo private baru, repo lama dihapus/diarsipkan | Bersih | Perlu pindah riwayat |

Mengubah ke private: GitHub → Settings → General → Danger Zone →
*Change repository visibility*.

## 3. Yang TIDAK BOLEH pernah masuk repo (private sekalipun)

| Jenis | Contoh di cluster ini |
|---|---|
| Password & daftar password | `/root/ood-password-awal.txt` |
| Hash password, passphrase | `/etc/ood/config/ood_portal.yml`, `/root/ood-dex-users.yml`, `/etc/ood/dex/config.yaml` |
| Kunci privat | `~/.ssh/id_*` (tanpa `.pub`), `/etc/ssl/ondemand/*.key`, `/etc/munge/munge.key` |
| Token | kunci Tailscale (`tskey-...`), token GitHub, kredensial rclone/Google Drive |
| Data & identitas pasien | ID sampel layanan, nama, VCF/CRAM/laporan |
| Kredensial BMC/IPMI, database Slurm (`slurmdbd.conf`) | |

Salinan konfigurasi di `scripts/ood/` sengaja **tanpa** berkas-berkas di atas.

## 4. Informasi internal di folder panduan ini

Folder ini **tidak** berisi password, hash, kunci, token, atau ID sampel
pasien. Tetapi berisi informasi internal berikut. Kalau repo tetap public,
ganti dengan placeholder:

| Informasi | Muncul | Ganti dengan |
|---|---|---|
| IP internal `192.168.18.x`, `192.168.30.x` | 17× | `<IP-CONTROLLER>`, `<IP-HPC>`, dst. |
| Nama tailnet `basa-minnow.ts.net` | 5× | `<tailnet>.ts.net` |
| Nama user tim (dewi, kila, angelo, jeffrey, reinhart, ilham) | banyak | `userA`, `userB`, ... |
| Nama PC Windows (`DESKTOP-...`) | 1× | hapus |
| Struktur folder data (`/media/bio-pool/KIL`, dst.) | banyak | boleh, atau samarkan |
| Kelemahan yang belum diperbaiki (dokumen 09: folder terbuka, node tidak stabil) | – | **jangan dipublikasikan sebelum diperbaiki** |

## 5. Pemeriksaan sebelum setiap upload

```bash
cd infra-docs
git status                       # lihat apa yang akan ikut
git diff --cached --stat         # hanya yang sudah di-add

# pola rahasia (harus kosong)
git diff --cached | grep -nE '\$2[aby]\$[0-9]{2}\$|BEGIN [A-Z ]*PRIVATE KEY|tskey-|ghp_|oidc_crypto_passphrase: *[^ <]|(password|passwd|secret|token) *[:=] *[^ <{$]{6,}'
```
Alat yang lebih teliti: `gitleaks detect` (riwayat) dan `gitleaks protect --staged`.

Aturan kerja:
1. `git add` berkas satu per satu, **jangan** `git add .` atau `git add -A`.
2. Jangan commit berkas sampah. Di akar repo ada berkas bernama aneh
   (`", d.get(url))...`, 313 byte, 23/9), sisa perintah yang salah ketik:
   hapus, jangan di-commit.
3. Tambahkan ke `.gitignore`: `*.key`, `*password*`, `ood_portal.yml`,
   `ood-dex-users.yml`, `id_*` (kecuali `*.pub`), `.env`.
4. Kalau rahasia terlanjur ter-push: **ganti rahasianya** (password/kunci
   baru). Menghapus commit saja tidak cukup.

## 6. Urutan yang disarankan

1. Review isi folder ini.
2. Putuskan visibilitas repo (§2).
3. Perbaiki dulu temuan dokumen 09 nomor 3 (izin folder tim).
4. Pindahkan folder ini ke repo (misalnya `docs/panduan-cluster/`), jalankan
   pemeriksaan §5, commit di branch baru, lalu push dan buat PR.
