# ePBS Kurtosis devnet (ethereum-package)

Minimal devnet untuk mengamati boundary fork **Fulu → Gloas** (EIP-7732)
dengan `ethereum-package` di GitHub Actions — mesin gratis, bukan VM lokal.

## Kenapa repo ini ada

VM kerja tidak bisa menjalankan Kurtosis (RAM bebas cuma ~426 MiB;
devnet minimal butuh ~3,5–5,5 GiB; Docker tidak terpasang). Runner
GitHub Actions (`ubuntu-24.04`: 4 vCPU, 16 GB RAM, 14 GB SSD, Docker
sudah terpasang) memenuhi spek, dan gratis tanpa batas di repo publik.

## Isi repo

- `params.yaml` — config devnet minimal: genesis Fulu, fork Gloas epoch 2,
  1 participant (geth + Lighthouse), 8 validator.
  **DRAFT, belum pernah dijalankan.** Kunci fork-epoch + tag image client
  harus sesuai versi `ethereum-package` yang dipakai saat run (workflow
  memvalidasi ini sebelum jalan).
- `.github/workflows/kurtosis-epbs-devnet.yml` — workflow manual
  (`workflow_dispatch`): install Kurtosis CLI → run devnet → tunggu
  melewati epoch Gloas → kumpulkan log + snapshot API → upload artifact
  → cleanup enclave.
- `RUNBOOK.md` — perintah yang sama untuk mesin sendiri (bila nanti ada).
- `CATATAN.md` — laporan feasibility VM + angka blocker + spek mesin.

## Yang diamati di boundary fork

- Apakah blok Gloas pertama ter-build, ter-propagasi, ter-finalisasi?
- Log CL di sekitar slot pertama epoch Gloas.
- **Kalau jalan normal (yang diharapkan):** di mana client menyimpang
  dari spec (parent pra-Gloas diperlakukan FULL, dsb.) — itu bukti
  mitigasi, bukan bantahan.
- **Kalau macet:** slot, pesan error persis, dan kecocokan dengan
  rantai kegagalan D1–D6 dari PoC pyspec.

## Yang TIDAK ada di repo ini

Temuan / draf laporan / PoC pyspec — semua itu tetap privat sampai
disubmit ke Ethereum Foundation. Repo ini hanya berisi config devnet
generik + skrip observasi.

## Cara pakai

1. Buat repo publik baru, push isi direktori ini.
2. Actions → `kurtosis-epbs-devnet` → Run workflow.
3. Ambil artifact `epbs-devnet-observations` setelah job selesai.
