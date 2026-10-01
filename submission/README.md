# Bahan Submission JAWIR Sentinel

Dokumen ini menyediakan bahan untuk mengisi `Submission Template _ AI Builder Cup.pptx`. Narasi menggunakan Bahasa Indonesia. Nama teknologi, role, endpoint, field, dan enum mengikuti kontrak aslinya.

**Status:** draft bahan submission berdasarkan desain final, bukan laporan prototype yang sudah berjalan.  
**Tanggal pemeriksaan:** 2 Oktober 2026 (Asia/Jakarta).  
**Baseline repo docs:** commit `6eb1bec32113776f2ec723bfcb0537b6c6d8b5ac`.

## Peta template

Nomor berikut mengikuti urutan slide pada file yang diunggah, termasuk slide instruksi dan slide tanpa teks konten yang teridentifikasi.

| Slide | Bagian template | Sumber bahan | Kesiapan |
|---|---|---|---|
| 1 | Team Details | [Ringkasan solusi](solution-brief.md) | Team name tersedia; nama ketua perlu diisi |
| 2 | Instruksi pengunduhan template | Tidak membutuhkan dokumen konten | Ikuti instruksi template saat menyiapkan PPT |
| 3 | Brief about the idea | [Ringkasan solusi](solution-brief.md) | Draft narasi tersedia |
| 4 | Opportunities: perbedaan, penyelesaian masalah, USP | [Peluang dan nilai solusi](opportunities.md) | Draft tersedia; klaim pasar belum divalidasi |
| 5 | List of features | [Fitur dan alur](features-and-flow.md) | Desain final tersedia; implementasi belum diverifikasi |
| 6 | Process flow / Use-case diagram | [Fitur dan alur](features-and-flow.md) | Diagram alur desain tersedia |
| 7 | Wireframes/Mock diagrams (optional) | [Demo dan aset](demo-and-assets.md) | Daftar tampilan tersedia; aset visual belum tersedia |
| 8 | Architecture diagram | [Arsitektur dan teknologi](architecture-and-technology.md) | Diagram desain tersedia |
| 9 | Technologies to be used | [Arsitektur dan teknologi](architecture-and-technology.md) | Stack final tersedia |
| 10 | Estimated implementation cost (optional) | [Evaluasi, biaya, dan pengembangan](evaluation-cost-roadmap.md) | Model biaya tersedia; nominal belum dihitung |
| 11 | Snapshots of the prototype | [Demo dan aset](demo-and-assets.md) | Screenshot runtime belum tersedia |
| 12 | Prototype Performance report/Benchmarking | [Evaluasi, biaya, dan pengembangan](evaluation-cost-roadmap.md) | Metode evaluasi tersedia; hasil belum diukur |
| 13 | Additional Details/Future Development | [Evaluasi, biaya, dan pengembangan](evaluation-cost-roadmap.md) | Draft arah evaluasi tersedia |
| 14 | GitHub, Demo Video (3 Minutes), Final Product | [Demo dan aset](demo-and-assets.md) | Repo public tersedia; video dan produk belum diverifikasi |
| 15–16 | Tidak ada teks konten pada template yang diperiksa | Tidak ada requirement teks yang teridentifikasi | Tentukan penggunaan saat menyusun PPT |

## Cara menggunakan bahan

Gunakan bagian **Draft untuk slide** sebagai titik awal copy PPT. Tabel detail, protokol pengujian, dan catatan sumber mendukung diskusi tim dan bukti submission. Panjang copy tetap perlu disesuaikan dengan ruang slide.

File dalam folder ini menjelaskan solusi untuk submission. Kontrak implementasi tetap berada pada [workflow](../workflow/workflow.md), [architecture](../architecture/architecture.md), [API](../api/api-contract.md), dan [database](../database/database-design.md). Jika ada perbedaan, perbaiki bahan submission dengan mengikuti kontrak terbaru.

## Batas bukti

Pemeriksaan branch `main` menemukan README pada repo BE/FE dan dokumen spesifikasi pada repo docs. Pemeriksaan ini belum memverifikasi aplikasi yang berjalan, deployment, benchmark, atau hasil pilot. Implementasi mungkin berada pada branch atau lingkungan lain yang belum diperiksa.

Pisahkan status berikut:

| Status | Makna |
|---|---|
| Terdokumentasi | Perilaku/stack ditetapkan dalam spesifikasi |
| Draft narasi | Rumusan untuk presentasi, masih dapat diedit |
| Hipotesis manfaat | Dampak yang perlu diuji |
| Belum terverifikasi | Bukti implementasi/runtime belum tersedia |
| Terukur | Hasil uji dengan konfigurasi dan bukti yang dapat diperiksa |

## Data yang masih perlu dilengkapi

- Nama ketua tim dan informasi anggota yang akan ditampilkan.
- Bukti prototype berjalan dan screenshot sesuai skenario demo.
- Hasil evaluasi AI, workflow, latency, serta biaya aktual.
- Video demo berdurasi tiga menit dan URL produk yang dapat diakses penilai.
