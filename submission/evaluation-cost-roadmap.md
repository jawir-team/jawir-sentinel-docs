# Evaluasi, Biaya, dan Pengembangan

**Untuk slide:** 10, 12, dan 13.  
**Status:** protokol dan draft rencana. Belum ada hasil benchmark atau estimasi biaya nominal yang terverifikasi.

## Slide 12: Prototype Performance / Benchmarking

### Pertanyaan evaluasi

Apakah Sentinel menghasilkan analysis yang memiliki dasar policy/evidence, menjaga workflow authority, dan menyimpan riwayat yang benar ketika review atau execution membutuhkan perubahan?

Uji AI quality terpisah dari deterministik workflow correctness. Recommendation yang terdengar baik tidak membuktikan guard backend benar, dan guard benar tidak membuktikan recommendation berkualitas.

### Dataset dan prosedur yang diusulkan

Gunakan synthetic case dengan expected policy/evidence references dan rubric yang direview manusia. Sertakan policy found, policy partial, no applicable policy, insufficient evidence, policy conflict, serta PDF/JPEG/PNG supported evidence. Jangan mengisi sample size sebelum dataset benar-benar ditetapkan.

Catat model/version, prompt version, exact policy versions, dataset version, retrieval configuration, environment, retry configuration, jumlah run, dan timestamp. Ulangi case untuk memeriksa variasi output. Reviewer menilai correctness dan unsupported claims terhadap reference yang sama.

### Metrik dan status hasil

| Metrik | Cara menghitung/mengukur | Hasil saat ini |
|---|---|---|
| Validitas structured output | Output valid schema dibagi total generation attempt; laporkan first attempt dan final setelah retry secara terpisah | Belum diukur |
| Ketepatan policy retrieval | Recall@8 terhadap policy chunks relevan pada ground truth | Belum diukur |
| Validitas citation | Reference yang cocok dengan source/version/section dibagi reference yang diperiksa | Belum diukur |
| Unsupported claim rate | Klaim faktual/policy tanpa source yang memadai dibagi klaim yang direview | Belum diukur |
| Workflow correctness | Passed deterministic scenarios dibagi skenario yang dijalankan | Belum diukur |
| Latency analysis | Durasi sejak durable job intent commit hingga finalization; laporkan jumlah sampel, p50, p95, retry, dan failure | Belum diukur |
| Waktu proses manusia | Waktu dari case siap direview sampai decision, pada tugas/input setara | Belum diukur |
| Biaya AI per cycle | Biaya generation + verifier + embeddings + technical retry dibagi completed/failed cycles yang dilaporkan | Belum diukur |

### Skenario workflow minimum

Verifikasi happy path, Checker/Signer reject, execution BLOCKED/FAILED, stale approval, SoD violation, quota exhaustion, verifier FAIL, technical retry exhaustion, duplicate RabbitMQ delivery, stale worker finalization, close saat proses aktif, dan policy activation sebelum READY/effective.

Untuk quota exhaustion, pastikan business action/feedback tetap tersimpan dan tidak ada analysis/outbox baru. Untuk duplicate delivery, periksa tidak ada double finalization. Untuk failed analysis, periksa current_analysis_id tetap menunjuk reviewable analysis sebelumnya atau NULL.

### Pembanding manfaat

Jika ingin mengklaim penghematan waktu, bandingkan dengan proses manual atau setup pembanding yang didefinisikan jelas. Gunakan input/tugas setara, kontrol pengalaman reviewer dan urutan pengerjaan, lalu laporkan jumlah kasus serta variasinya. Persentase improvement tidak tersedia saat ini.

**Draft pengisian slide saat ini:** “Metode evaluasi mencakup policy grounding, citation validity, workflow correctness, latency, dan biaya per analysis cycle. Hasil benchmark belum tersedia.” Ganti dengan tabel hasil aktual sebelum mengklaim performance prototype.

## Slide 10: Estimated Implementation Cost (optional)

### Model biaya

| Komponen | Input yang perlu ditetapkan | Status |
|---|---|---|
| Cloud Run web/API | Region, CPU/memory, request/duration, minimum instances | Belum dihitung |
| Cloud Run Worker Pool | CPU/memory, running hours, instance count | Belum dihitung |
| Cloud SQL PostgreSQL | Instance tier, storage, uptime, backup | Belum dihitung |
| RabbitMQ | Hosting choice, kapasitas, uptime, storage | Belum dihitung |
| Vertex AI Gemini | Generation model, input/output tokens, verifier calls, retries | Belum dihitung |
| Embedding | Chunk/query volume dan re-indexing | Belum dihitung |
| Cloud Storage/network/logs | Object size/count, operations, transfer, log volume | Belum dihitung |
| Tenaga pengembangan | Lingkup pekerjaan, jam kerja, dan rate bila dihitung | Belum dihitung |

Pisahkan biaya development sekali, infrastructure bulanan, dan variable AI cost. Uptime worker/broker/database jangan diasumsikan mengikuti traffic request saja.

Rumus kerja: total bulanan = fixed infrastructure + AI/embedding usage + storage/network/log usage. AI usage mencakup generator, verifier, initial analysis, business re-analysis, dan technical attempts yang benar-benar berjalan.

Gunakan rate resmi yang berlaku pada tanggal estimasi setelah model, region, hosting, volume, dan konfigurasi dipilih. Jangan memasukkan angka tanpa sumber/tanggal atau memperlakukan kredit promosi sebagai biaya dasar. Saat ini belum cukup data untuk estimasi nominal yang dapat dipertanggungjawabkan.

## Slide 13: Additional Details / Future Development

### Kebutuhan sebelum submission

Prioritas penyelesaian bahan adalah membuktikan flow baseline pada runtime, mengumpulkan hasil evaluasi, merekam demo, dan menyiapkan tautan yang dapat diakses. Ini adalah pekerjaan pembuktian/submission, bukan perubahan scope final.

### Arah setelah MVP

Arah berikut adalah draft untuk diskusi tim, belum menjadi requirement implementasi:

- Pilot pada use case organisasi yang sudah mendapat izin untuk menilai manfaat operasional.
- Penyesuaian model/provider melalui boundary adapter yang sudah ada dalam desain, jika kebutuhan privacy/cost/latency menuntutnya.
- Evaluasi perluasan berdasarkan temuan pilot dan benchmark; masukkan requirement baru melalui revisi contract yang disepakati.

### Draft untuk slide 13

Pengembangan berikutnya diarahkan pada validasi manfaat melalui pilot terkontrol dan penyempurnaan berdasarkan hasil evaluasi. Arsitektur memisahkan workflow dari model/provider sehingga pilihan AI dapat dievaluasi sesuai kebutuhan organisasi. Scope lanjutan ditetapkan setelah baseline memiliki bukti kualitas, reliability, dan biaya.

## Sumber

[Workflow](../workflow/workflow.md), [architecture](../architecture/architecture.md), [database](../database/database-design.md), dan [README Backend](https://github.com/jawir-team/jawir-sentinel-be/blob/main/README.md) menjadi dasar expected behavior. Semua metrik, prosedur uji, model biaya, dan arah pilot dalam dokumen ini adalah bahan evaluasi yang belum dijalankan.

