# Demo, Aset Visual, dan Tautan Submission

**Untuk slide:** 7, 11, dan 14.  
**Status:** rencana pengisian. Screenshot, video, dan deployment belum terverifikasi.

## Mapping persona demo

| User demo | Role pada case |
|---|---|
| Operations User | MAKER / OWNER |
| Risk User | CHECKER |
| Manager User | SIGNER |
| Development User | EXECUTER |

Keempat user berbeda. ADMIN pada Manager User adalah system role dan tidak memberi case authority tambahan di luar assignment SIGNER.

## Skenario demo utama

Gunakan case synthetic reconciliation mismatch yang tersedia dalam baseline. Tunjukkan initial analysis, Checker reject dengan reason/evidence, analysis berikutnya, approval Checker dan Signer, execution BLOCKED, analysis berikutnya, approval ulang, lalu execution SUCCESS menjadi DONE. History harus menunjukkan perubahan version dan dasar keputusan.

Pengujian multi-Checker adalah skenario tambahan dengan user Checker lain yang berbeda dari seluruh participant. Demo utama empat persona tidak memakai Maker sebagai Executer.

## Wireframe dan screenshot

| Tampilan yang direncanakan | Tujuan komunikasi | Aset saat ini |
|---|---|---|
| Create Case + participants | Menunjukkan input case dan pemisahan role | Belum tersedia |
| AI Analysis | Menunjukkan facts, assumptions, policy/evidence references, recommendation, dan uncertainty | Belum tersedia |
| Checker Review | Menunjukkan reason reject terhadap exact analysis version | Belum tersedia |
| Analysis Version + History | Menunjukkan re-analysis dan decision historis | Belum tersedia |
| Signer Authorization | Menunjukkan authorization setelah required approval | Belum tersedia |
| Execution | Menunjukkan BLOCKED/SUCCESS dan supporting evidence | Belum tersedia |
| Escalation | Menunjukkan terminal cause dan stopped workflow | Belum tersedia |

Wireframe/mock dapat menjelaskan rancangan pada slide 7 yang opsional. Slide 11 membutuhkan screenshot prototype nyata. Jangan memakai mock sebagai bukti runtime tanpa label mock. Aset yang dipublikasikan harus memakai data demo synthetic.

## Rencana video tiga menit

Urutan ini adalah draft produksi video, bukan bukti flow sudah berhasil.

| Waktu | Isi |
|---|---|
| 0:00–0:15 | Problem dan ringkasan Sentinel |
| 0:15–0:35 | Case synthetic dan role assignment |
| 0:35–1:00 | Analysis v1, policy/evidence reference, dan uncertainty |
| 1:00–1:30 | Checker reject, analysis v2, dan review ulang |
| 1:30–1:55 | Checker approval dan Signer authorization |
| 1:55–2:20 | Execution BLOCKED dan analysis v3 |
| 2:20–2:40 | Approval ulang dan execution SUCCESS |
| 2:40–3:00 | Audit/version history dan penutup |

Jika generation lebih lama, edit jeda secara transparan dan beri penanda bahwa video dipotong. Durasi video bukan hasil latency benchmark.

## Tautan untuk slide 14

| Requirement | Tautan/status |
|---|---|
| GitHub Public Repository | [jawir-sentinel-docs](https://github.com/jawir-team/jawir-sentinel-docs) |
| Backend repository | [jawir-sentinel-be](https://github.com/jawir-team/jawir-sentinel-be) |
| Frontend repository | [jawir-sentinel-fe](https://github.com/jawir-team/jawir-sentinel-fe) |
| Demo Video Link (3 Minutes) | Belum tersedia |
| Final Product Link | Belum tersedia/belum diverifikasi |

Repo dapat menjadi entry point untuk submission. Public visibility tidak membuktikan prototype sudah tersedia. Pastikan penilai dapat mengikuti tautan produk/video tanpa akses khusus yang belum dijelaskan.

## Bukti yang perlu dikumpulkan

Saat prototype tersedia, catat deployment version/commit, tanggal perekaman, screenshot path, dan hasil skenario. Tautkan aset sebenarnya pada dokumen ini. Hindari menuliskan URL contoh sebagai URL final.

## Sumber

[Workflow demo](../workflow/workflow.md), [database demo](../database/database-design.md), dan [README Frontend](https://github.com/jawir-team/jawir-sentinel-fe/blob/main/README.md).

