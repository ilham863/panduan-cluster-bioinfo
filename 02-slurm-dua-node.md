# 02 — Slurm dua node

Slurm 23.11.4, Ubuntu 24.04, munge 0.5.15. Cluster `bioinfo`.

---

## 1. Komponen

| Peran | Host | Alamat |
|---|---|---|
| Controller (`slurmctld`) + accounting (`slurmdbd`) | VM 100 `pipeline` | 192.168.18.194 |
| compute001 | HPC-GPU (server fisik) | 192.168.18.178 |
| compute002 | VM 200 `dev-bioinfo` (di Proxmox) | 192.168.18.191 |

Pengaturan inti `slurm.conf`:
```
SelectType=select/cons_tres
SelectTypeParameters=CR_Core_Memory      # CPU dan RAM sama-sama dijatah
ProctrackType=proctrack/cgroup
TaskPlugin=task/cgroup,task/affinity     # job tidak bisa melewati jatahnya
AccountingStorageEnforce=associations,limits,qos
PriorityType=priority/multifactor
ReturnToService=2
```

## 2. Node dan partisi

```
NodeName=compute001 NodeAddr=192.168.18.178 CPUs=256 SocketsPerBoard=2 CoresPerSocket=64 ThreadsPerCore=2 RealMemory=1024000 Gres=gpu:a100:1
NodeName=compute002 NodeAddr=192.168.18.191 CPUs=96  SocketsPerBoard=1 CoresPerSocket=96 ThreadsPerCore=1 CoreSpecCount=8 RealMemory=114688
```

| Partisi | Node | Waktu maks | Untuk |
|---|---|---|---|
| `cpu` (default) | compute001 | 2 hari | tahap pipeline biasa |
| `long` | compute001 | 7 hari | tahap panjang |
| `gpu` | compute001 | 2 hari | tahap GPU |
| `debug` | compute001 | 30 menit | uji cepat |
| `cpu-only` | compute002 | 7 hari | kepala Nextflow, pipeline ringan–menengah |

`CoreSpecCount=8` menyisihkan 8 inti VM 200 untuk portal, SSH, dan sistem.

## 3. QoS (batas pemakaian)

| QoS | Total semua user (GrpTRES) | Per user (MaxTRESPU) | Waktu | Job/user | Untuk |
|---|---|---|---|---|---|
| `normal` | cpu=128, mem=384G, gpu=1 | cpu=128, mem=480G | 2 hari | 20 | tahap pipeline |
| `besar` | – | cpu=248, mem=960G | 7 hari | 4 | kepala Nextflow, tahap > 2 hari |

Akun: `bioinfo` (angelo, dewi, ilham, jeffrey, kila, reinhart) dan `pipeline`.

> **Masalah terbuka:** batas total QoS `normal` (128 inti) dibuat saat cluster
> masih 1 node. Sekarang total 344 inti, tapi hanya satu user yang bisa memakai
> jatah penuhnya pada satu waktu. Usulan: `GrpTRES=cpu=344`. Lihat dokumen 09.

Periksa selalu empat jenis batas:
```bash
sacctmgr show qos format=Name,GrpTRES%40,MaxTRESPU%30,MaxWall,MaxJobsPU
```

## 4. Prosedur menambah compute node baru

Contoh untuk node ketiga (misalnya server V100).

1. **Siapkan OS** (Ubuntu 24.04), jam (chrony), jaringan ke 192.168.18.x dan
   192.168.30.x (untuk NFS).
2. **User & UID sama dengan cluster.** Semua user tim harus punya UID/GID yang
   sama dengan node lain (lihat `scripts/uid-gid/` di repo).
3. **Mount penyimpanan bersama** (`/etc/fstab`):
   ```
   192.168.30.2:/bio-pool /media/bio-pool nfs defaults,nofail,hard,timeo=600,retrans=2,_netdev 0 0
   ```
4. **Pasang Slurm dan munge**, lalu samakan UID `slurm`:
   ```bash
   apt install slurmd munge
   systemctl stop slurmd
   groupmod -g 9999 slurm; usermod -u 9999 -g 9999 slurm
   chown -R slurm:slurm /var/log/slurm /var/lib/slurm /var/spool/slurmd
   ```
5. **Salin dari controller**: `/etc/munge/munge.key` (mode 400, milik munge),
   `/etc/slurm/slurm.conf`. Salin `gres.conf` **hanya** kalau node punya GPU dan
   isinya sudah disesuaikan.
6. **Lihat spesifikasi asli node**: `slurmd -C`. Pakai angka itu untuk baris
   `NodeName=`, kurangi sedikit `RealMemory` untuk sistem.
7. **Di controller**, saat antrean sepi:
   ```bash
   cp -a /etc/slurm/slurm.conf /etc/slurm/slurm.conf.bak-$(date +%F)
   # tambah NodeName=... dan PartitionName=...
   grep -c "^NodeName=compute003" /etc/slurm/slurm.conf   # HARUS 1
   systemctl restart slurmctld
   ```
   Sebarkan `slurm.conf` yang sama ke **semua** node, lalu `systemctl restart slurmd`.
8. **Verifikasi**:
   ```bash
   sinfo -N
   srun -p <partisi-baru> -c1 hostname
   ```

## 5. Hal yang perlu dijaga

- `slurm.conf` **harus identik** di semua node. Salinan di HPC-GPU sempat
  tertinggal; versi baru disiapkan di `/home/ilham/slurm.conf.baru` dan perlu
  disalin + `restart slurmd` (lihat dokumen 09).
- Jangan restart `slurmctld` saat ada run panjang tanpa kebutuhan mendesak.
- Setiap perubahan `slurm.conf`/QoS dicatat di `CHANGELOG.md` repo.
