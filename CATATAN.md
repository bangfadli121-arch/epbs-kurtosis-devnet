# Kurtosis devnet — CATATAN (hasil feasibility + penyiapan)

**Tanggal:** 12 Nov 2026, ~11:40–11:50 WIB
**Status: TIDAK DIBANGUN — devnet TIDAK BISA jalan di VM ini. Bukan kegagalan
hunting; blocker infrastruktur. Semua yang di bawah ini terukur, bukan dugaan.**

---

## 1. Verdict singkat

| Item | Hasil |
|---|---|
| Docker di VM ini | **Tidak terpasang** (`docker: command not found`) |
| Kurtosis CLI | **Tidak terpasang** (`kurtosis: command not found`) |
| Go (untuk build client dari source) | **Tidak terpasang** |
| RAM total VM | 8.124.900 kB (~7,75 GiB) |
| RAM **tersedia** saat cek (12 Nov 11:40 WIB) | **591.676 kB (~578 MiB)** — `MemAvailable` |
| RAM bebas murni (`MemFree`) | 131.700 kB (~129 MiB) |
| Swap | 1 GiB, sudah terpakai 402 MiB |
| CPU | 4 core terlihat, tapi alokasi runtime cell hanya **1,5 CPU** shared |
| Disk bebas di `/` | 7,4 GiB |
| Devnet jalan? | **TIDAK.** Dua blocker independen: (1) Docker tidak ada, (2) RAM tidak cukup — selisih ~10x. |

Perintah feasibility yang dijalankan (persis) ada di `RUNBOOK.md` bagian A.
Tidak ada yang diinstal, tidak ada image yang di-pull, tidak ada container yang
dijalankan — sesuai instruksi langkah 1: kalau Docker tidak ada / RAM tidak
cukup, BERHENTI dan laporkan blockernya.

---

## 2. Kenapa RAM ini tidak cukup (angka, bukan perasaan)

Devnet minimal adalah: 1 EL (mis. geth) + 1 CL (mis. Lighthouse) + 1 validator
+ Kurtosis engine + Docker daemon. Perkiraan kebutuhan realistis (kasus devnet
segar, bukan sync mainnet):

| Komponen | Perkiraan RAM |
|---|---|
| Geth (EL, devnet) | ~1–2 GiB |
| Lighthouse (CL + validator, devnet) | ~1–2 GiB |
| Kurtosis engine | ~0,5–1 GiB |
| Docker daemon + overhead | ~0,5 GiB |
| **Total minimal** | **~3,5–5,5 GiB bebas** |
| **Tersedia di VM ini** | **~0,58 GiB** |

Selisihnya ~6–10x. Swap 1 GiB (sudah 40% terpakai) tidak menolong — client
Ethereum yang di-swap akan timeout slot dan devnet tidak akan finalisasi.
Menginstal Docker saja tidak akan mengubah hasil: Docker butuh RAM untuk
container-nya, dan RAM itulah yang tidak ada. CPU juga tipis (1,5 CPU shared
untuk EL+CL yang harus memproses slot 12 detik).

Disk 7,4 GiB bebas juga mepet untuk image (geth ~0,5 GiB, lighthouse ~0,3 GiB,
Kurtosis engine + ethereum-package + genesis assets), tapi ini blocker kedua;
RAM-lah yang mematikan.

Satu-satunya hal yang bisa "membebaskan" RAM adalah membunuh proses runtime
(`hatch daemon`, ~350 MiB) — itu sama saja membunuh VM tempat kerja ini.
Tidak dilakukan.

---

## 3. Spesifikasi mesin yang dibutuhkan (untuk menjalankan config ini)

| Komponen | Minimum yang masuk akal | Nyaman |
|---|---|---|
| OS | Linux 64-bit, kernel mendukung Docker | sama |
| Docker | Docker Engine 24+ (daemon jalan) | sama |
| Kurtosis CLI | versi terbaru (lihat RUNBOOK.md §B) | sama |
| RAM bebas | **8 GiB** | 16 GiB |
| CPU | 4 vCPU dedicated | 8 vCPU |
| Disk bebas | 30 GiB (SSD) — untuk image + volume enclave | 50 GiB |
| Internet | untuk pull image client + ethereum-package | sama |

Config siap-jalan ada di `params.yaml` (DRAFT — BELUM DIJALANKAN, lihat §4).
Perintah persis untuk mesin yang mampu ada di `RUNBOOK.md` §B–§E.

**Catatan soal Kaggle: tidak bisa.**
Kaggle kernel tidak memberi Docker daemon (tidak ada privileged mode), dan
Kurtosis mutlak butuh Docker. RAM Kaggle (~13 GiB) cukup, tapi tanpa Docker,
Kurtosis tidak bisa jalan. Jadi Kaggle bukan jalan keluar untuk ini.

---

## 4. Config yang disiapkan (`params.yaml`) — DRAFT, BELUM DIJALANKAN

Isinya: devnet minimal, 1 participant (geth + Lighthouse), genesis Fulu,
fork Gloas di epoch kecil (supaya boundary cepat tercapai), validator secukupnya
untuk finalisasi.

**Yang harus diverifikasi di mesin yang mampu SEBELUM `kurtosis run`:**
- Nama kunci fork-epoch di `network_params` sesuai versi ethereum-package yang
  dipakai (repo ethpandaops/ethereum-package — cek `network_params` schema di
  versi itu; kunci fork epoch berubah antar versi).
- Tag image client harus yang **mendukung Gloas** (mis. dari tabel rilis Sepolia:
  Lighthouse 8.3.0-rc.0 / Prysm 7.2.0 / Teku 26.9.1 dan geth 1.17.7, atau yang
  lebih baru). Image lama tanpa Gloas tidak akan mencapai fork.
- `validator_count`: cukup untuk finalisasi (1 participant dengan validator
  tunggal bisa finalisasi karena 1/1, tapi pakai nilai default package bila
  ragu; tune di run time).

Config ini TIDAK pernah dieksekusi di VM ini. Jangan dikutip sebagai hasil.

---

## 5. Yang diamati devnet ini

1. Apakah chain mencapai boundary fork (epoch Gloas) dan apakah **blok Gloas
   pertama ter-build + ter-propagasi + ter-finalisasi** — via log CL
   (`kurtosis service logs <enclave> <cl-service>`) dan endpoint REST CL
   (`/eth/v1/beacon/headers`, `/eth/v1/beacon/blocks`, finalized checkpoint).
2. Log CL di sekitar slot pertama epoch Gloas (error / reorg / finalisasi).
3. Snapshot API beacon sebelum dan sesudah boundary.

Devnet yang jalan normal **bukan** bukti apa pun tentang spec — client memang
menangani boundary ini di implementasi mereka (itu kenapa Sepolia/Devnet-11
jalan). Devnet **tidak membuktikan** apa pun soal mainnet. Glamsterdam belum
di mainnet.

**Jangan patch client** untuk memaksakan stall lalu mengklaim itu bukti
dunia nyata. Client yang dimodifikasi adalah eksperimen sintetis yang terpisah
— kalau dilakukan, harus dilabeli sebagai itu, bukan sebagai reproduksi.

Analisis detail dari hasil devnet bersifat privat dan tidak ada di repo ini.

---

## 6. File di repo ini

- `CATATAN.md` — file ini (temuan feasibility + spek mesin).
- `params.yaml` — config ethereum-package DRAFT (belum dijalankan).
- `RUNBOOK.md` — perintah persis: (A) feasibility yang sudah dijalankan di
  VM ini + hasilnya, (B) instalasi di mesin yang mampu, (C) perintah run,
  (D) perintah observasi di boundary fork, (E) cleanup.
- `.github/workflows/kurtosis-epbs-devnet.yml` — workflow GitHub Actions
  (manual trigger): install Kurtosis CLI → run devnet → tunggu epoch Gloas
  → kumpulkan log + snapshot → upload artifact → cleanup.
