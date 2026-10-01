# Ringkasan Solusi dan Tim

**Untuk slide:** 1 dan 3.  
**Status:** draft narasi berdasarkan scope dan desain yang terdokumentasi.

## Team Details

| Field template | Isi |
|---|---|
| Team name | JAWIR |
| Team leader name | Belum tersedia; isi dengan nama yang digunakan dalam pendaftaran |
| Solution name | JAWIR Sentinel |
| Problem Statement | Tim operasional finansial perlu menilai kasus yang melibatkan fakta, evidence, dan SOP yang tersebar, lalu memperoleh review, authorization, dan mencatat hasil execution. JAWIR Sentinel dirancang untuk menyatukan konteks keputusan dan menjaga jejak review ketika analisis perlu diperbarui. |

Nama ketua tidak disimpulkan dari username GitHub atau nama panggilan. Informasi anggota, organisasi afiliasi, dan kontak perlu berasal dari data tim yang dikonfirmasi.

## Ruang masalah

Fokus solusi adalah operational exception dan kasus operasional yang memerlukan penilaian risiko serta kepatuhan. Contoh kebutuhan yang menjadi dasar desain:

| Kebutuhan | Dampak pada proses | Respons desain |
|---|---|---|
| Mengumpulkan fakta, log, dokumen, dan SOP yang relevan | Reviewer membutuhkan konteks yang dapat diperiksa | Case dan evidence terstruktur dengan provenance |
| Menentukan SOP yang applicable dan masih berlaku | Recommendation perlu memiliki dasar yang jelas | Retrieval policy ACTIVE, READY, dan effective |
| Menilai tindakan ketika informasi belum lengkap | Keputusan perlu memperlihatkan asumsi dan ketidakpastian | Facts, assumptions, unknowns, alternatif, dan missing information |
| Mengubah recommendation setelah review/execution bermasalah | Approval lama dapat merujuk analisis yang sudah diganti | Analysis versioning dan review ulang |

Ini adalah rumusan masalah berdasarkan use case proyek. Belum ada data survei, volume kasus nyata, atau besaran kerugian yang membuktikan prevalensi dan dampaknya pada sebuah institusi.

## Pengguna dan tanggung jawab

| Role per case | Kebutuhan utama |
|---|---|
| MAKER | Menyusun case dan evidence, menetapkan participant, lalu submit |
| CHECKER | Memeriksa reasoning, dasar policy/evidence, risiko, dan recommendation |
| SIGNER | Mengotorisasi tindakan setelah required Checker menyetujui |
| EXECUTER | Menjalankan approved action dan mencatat SUCCESS, BLOCKED, atau FAILED |

Role memakai user berbeda pada case yang sama. Owner adalah Maker. System role ADMIN mengelola fungsi administratif sesuai authorization dan tidak memberikan kewenangan workflow di luar assignment.

## Draft untuk slide 3

JAWIR Sentinel membantu tim operasional finansial menganalisis kasus menggunakan evidence dan SOP aktif yang relevan. AI menyusun penilaian risiko, recommendation, alternatif, dan ketidakpastian. Checker memeriksa analisis, Signer mengotorisasi tindakan, dan Executer mencatat hasilnya. Reject atau execution yang terblokir/gagal memulai analisis berikutnya dalam batas workflow. Setiap versi analisis, dasar policy, decision, dan hasil execution tersimpan untuk audit.

## Contoh case demo

Case synthetic: 37 transaksi tidak cocok pada reconciliation, settlement cutoff tersisa 90 menit, delapan transaksi kekurangan approval evidence, dan proses downstream sedang menunggu.

Angka tersebut adalah input skenario demo dari baseline proyek. Angka ini bukan statistik operasional atau hasil pengukuran manfaat.

## Sumber

- [Workflow](../workflow/workflow.md): role, strict SoD, ownership, analysis loop, dan audit.
- [README Frontend](https://github.com/jawir-team/jawir-sentinel-fe/blob/main/README.md): persona dan synthetic demo case.
- [README Backend](https://github.com/jawir-team/jawir-sentinel-be/blob/main/README.md): scope layanan dan seed demo.

