# Peluang dan Nilai Solusi

**Untuk slide:** 4.  
**Status:** draft argumentasi desain. Manfaat operasional belum diukur dan perbandingan produk pasar belum dilakukan.

## Bagaimana solusi menangani masalah

Sentinel menghubungkan analisis AI dengan proses keputusan yang eksplisit. Recommendation memiliki reference policy/evidence, reviewer memberikan decision terhadap exact analysis version, dan execution result kembali menjadi context pada business re-analysis yang sah. Riwayat tetap tersimpan ketika recommendation berubah.

| Masalah pada use case | Mekanisme desain | Manfaat yang perlu diuji |
|---|---|---|
| Pencarian konteks dan SOP memerlukan pekerjaan terpisah | Selective context dan policy retrieval | Waktu menyiapkan analisis berkurang |
| Recommendation sulit diperiksa dasarnya | Provenance, facts/assumptions/unknowns, verifier | Reviewer lebih mudah menemukan gap atau klaim tanpa dukungan |
| Approval dapat tertinggal setelah perubahan analisis | Decision terikat analysis_id dan review ulang | Approval tidak digunakan pada versi yang berbeda |
| Blocker execution kehilangan konteks | Evidence hasil execution dan governed re-analysis | Analisis berikutnya mempertimbangkan kendala nyata |
| Riwayat keputusan sulit direkonstruksi | Versioned references dan append-only audit | Review historis memiliki konteks yang lebih lengkap |

## Perbedaan pendekatan

Perbandingan ini menjelaskan jenis pendekatan, bukan audit capability produk tertentu. Tidak diasumsikan bahwa semua chatbot atau workflow software memiliki keterbatasan yang sama.

| Pendekatan | Fokus umum dalam perbandingan | Fokus desain Sentinel |
|---|---|---|
| Percakapan dengan AI | Menghasilkan respons dari input pengguna | Analisis terstruktur di dalam case workflow dengan decision versioning |
| Pencarian dokumen/policy | Menemukan isi dokumen yang relevan | Menggunakan retrieved policy untuk recommendation yang diperiksa dan diotorisasi |
| Workflow approval | Mengarahkan task antar role | Menggabungkan analisis AI, policy/evidence, review, dan feedback execution |

Keunikan terhadap kompetitor bernama atau klaim “pertama/satu-satunya” belum memiliki bukti. USP di bawah adalah positioning yang diusulkan berdasarkan kombinasi capability desain.

## USP yang diusulkan

**Analisis kasus berbasis SOP dan evidence yang tetap terlacak sepanjang review, authorization, execution, dan perubahan recommendation.**

Human accountability menjadi bagian mekanisme produk: backend memvalidasi role/state, seluruh required Checker menyetujui versi saat ini, dan Signer memberikan authorization sebelum execution.

## Draft untuk slide 4

- Sentinel menyatukan policy retrieval dan analisis AI dengan review serta authorization per role.
- Setiap recommendation dapat ditelusuri ke policy version dan evidence yang digunakan.
- Reviewer reject atau execution BLOCKED/FAILED memperbarui analisis melalui workflow yang dibatasi, lalu memerlukan review ulang.
- Nilai utama yang ingin dibuktikan adalah analisis lebih mudah diperiksa dan keputusan lebih mudah direkonstruksi.

## Validasi yang masih diperlukan

Uji kebutuhan pengguna pada kasus operasional yang sesuai. Bandingkan waktu dan kualitas review dengan proses pembanding menggunakan input yang setara. Catat kapan sumber tersedia, kapan tidak lengkap, dan seberapa banyak correction yang diperlukan reviewer. Jangan mengubah hasil uji synthetic menjadi klaim dampak institusi tanpa pilot yang sesuai.

## Sumber

[Workflow](../workflow/workflow.md) dan [architecture](../architecture/architecture.md) mendasari mekanisme produk. [Protokol evaluasi](evaluation-cost-roadmap.md) menjelaskan cara menguji hipotesis manfaat.

