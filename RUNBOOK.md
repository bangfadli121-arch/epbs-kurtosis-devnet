# RUNBOOK — PoC Kurtosis devnet ePBS fork-transition

## A. Feasibility yang SUDAH dijalankan di VM ini (12 Nov 2026 ~11:40 WIB)

Perintah (persis):

```bash
nproc; free -h; df -h /
which docker; docker info
which kurtosis; kurtosis version
which kubectl kind minikube enclave; python3 --version; go version
ps aux --sort=-%mem | head -8
grep -E "MemTotal|MemFree|MemAvailable|Buffers|Cached|Slab|SReclaimable|Shmem" /proc/meminfo
id; which sudo apt-get podman nerdctl
```

Hasil (persis):

- `nproc` → 4 (alokasi runtime cell: 1,5 CPU shared)
- `free -h` → Mem total 7,7 GiB, used 7,2 GiB, free 134 MiB, available 577 MiB
- `/proc/meminfo` → MemTotal 8.124.900 kB, MemFree 131.700 kB,
  MemAvailable 591.676 kB, Cached 987.056 kB, Shmem 346.976 kB, Slab 247.068 kB
- Swap 1,0 GiB, terpakai 402 MiB
- `df -h /` → 7,4 GiB bebas
- `docker` → command not found; `kurtosis` → command not found; `go` → not found
- `id` → uid=0 (root), `sudo` + `apt-get` tersedia
- Proses terbesar: `hatch daemon` 4,6% mem (~350 MiB) — membunuhnya = membunuh
  runtime sendiri; tidak dilakukan.

**Keputusan: BERHENTI di langkah 1.** Docker tidak ada DAN RAM tidak cukup
(~0,58 GiB tersedia vs ~3,5–5,5 GiB minimal untuk 1 EL + 1 CL + engine).
Menginstal Docker tidak mengubah hasil (RAM-lah yang tidak ada).

---

## B. Instalasi di mesin yang mampu (Linux 64-bit, ≥8 GiB RAM bebas, 4 vCPU)

```bash
# 1. Docker Engine (Debian/Ubuntu) — ikuti docs.docker.com untuk distro mesin itu.
#    Intinya: install docker-ce, pastikan `docker info` jalan tanpa sudo
#    (atau tambahkan user ke grup docker + relogin).
sudo apt-get update && sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
# (tambahkan repo docker sesuai distro, lalu:) sudo apt-get install -y docker-ce docker-ce-cli containerd.io
docker info   # harus sukses

# 2. Kurtosis CLI (dokumen resmi Kurtosis — Linux)
echo "deb [trusted=yes] https://apt.fury.io/kurtosis-tech/ /" | sudo tee /etc/apt/sources.list.d/kurtosis.list
sudo apt-get update && sudo apt-get install -y kurtosis-cli
kurtosis version   # harus sukses
kurtosis config init   # kalau diminta
```

## C. Menjalankan devnet (mesin yang mampu)

```bash
cd ~/workspace/bug-bounty/ethereum-foundation/poc-glamsterdam/kurtosis-devnet
# Verifikasi lebih dulu kunci fork-epoch + tag image di params.yaml (lihat header file).
kurtosis run github.com/ethpandaops/ethereum-package "$(cat params.yaml)" --enclave ePBS-gloas
kurtosis enclave inspect ePBS-gloas
```

Tunggu sampai chain melewati epoch Gloas (epoch 2 per params.yaml) — beberapa
menit. Jangan sentuh apa pun sebelum boundary tercapai.

## D. Observasi di boundary fork (epoch Gloas)

```bash
# Daftar service + port CL (ganti <cl> dengan nama service CL dari inspect)
kurtosis enclave inspect ePBS-gloas
kurtosis service logs ePBS-gloas <cl-service-name> | grep -iE "gloas|fork|payload_status|get_head|on_block|finaliz" | tail -100

# Via API CL (port dari inspect; contoh port 4000):
curl -s localhost:<CL_HTTP_PORT>/eth/v1/beacon/headers | head -50
curl -s localhost:<CL_HTTP_PORT>/eth/v1/beacon/states/head/finality_checkpoints
curl -s "localhost:<CL_HTTP_PORT>/eth/v1/beacon/blocks/head" | head -20

# Yang dicatat:
#  1. Slot + epoch blok Gloas pertama; apakah ada, apakah finalized.
#  2. Log di sekitar slot pertama epoch Gloas (error / AttributeError / reorg).
#  3. Kalau jalan normal: bukti client menyimpang dari spec (cari di log/kode
#     client: parent pra-Gloas sebagai FULL, payload pra-fork verified,
#     penanganan parent Fulu tanpa signed_execution_payload_bid).
kurtosis files inspect ePBS-gloas   # kalau perlu ambil log ke file
```

## E. Cleanup (mesin yang mampu)

```bash
kurtosis enclave stop ePBS-gloas
kurtosis clean -a   # hapus semua enclave + image yang tidak dipakai
docker system df    # cek disk kembali
```

## F. Yang TIDAK dilakukan di VM ini

- Tidak ada instalasi Docker/Kurtosis (tidak ada gunanya tanpa RAM).
- Tidak ada pull image, tidak ada container, tidak ada enclave.
- Tidak ada patch client, tidak ada klaim reproduksi.
- PoC pyspec tetap satu-satunya bukti eksekusi cacat (exit 0, di
  ../poc_fork_transition_empty_parent.py).
