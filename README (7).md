# kf_validator: validator log penulisan dan bukti

Validator aturan V (integritas), C (konsistensi log vs dokumen .docx), P (plausibilitas), dan fitur D
untuk sesi perekaman penulisan. Satu modul Python tanpa dependensi wajib.

## Yang dilakukan dan tidak dilakukan

- **Dilakukan:** memeriksa rantai hash, Merkle root, tanda tangan, cap waktu, konsistensi panjang teks,
  dan struktur biner .docx (wadah ZIP, bagian tak dirujuk, cap waktu entri, properti dokumen).
- **Tidak dilakukan:** mengklasifikasi AI vs manusia. Keputusan hanya salah satu dari
  `evidence_integrity_failure`, `inconsistent_log_document`, `inconclusive_insufficient_data`,
  `no_validator_decision` (aturan R4 dan R6). Flag dan fitur diteruskan ke model.
- **Semua ambang (theta) adalah placeholder** di `DEFAULT_CONFIG`. Estimasi dari data pilot.

## Pakai

```bash
python3 make_sample.py examples                    # buat data simulasi
python3 kf_validator.py --session examples/human/session.json \
  --events examples/human/events.jsonl --bundle examples/human/bundle.json \
  --docx examples/human/doc.docx --config examples/config_demo.json --out report.json
python3 -m unittest -v test_kf_validator           # 21 uji
```

`config_demo.json` mematikan syarat tanda tangan pascakuantum karena liboqs tidak selalu terpasang.
Untuk bukti sungguhan biarkan `require_pq` bernilai `true`.

## Aturan yang diimplementasikan

| ID | Pemeriksaan |
|---|---|
| V1-V5, V4b | seq berurutan, t_ms monotonik, rantai hash, aritmetika panjang, kesinambungan panjang, pos dalam rentang |
| V6 | Merkle root (gaya RFC 6962), chain head, jumlah event, id sesi |
| V8 | validasi schema dan versi |
| V9 | hash dapat direproduksi dari serialisasi kanonik |
| V10 | tanda tangan hybrid: ECDSA P-256 (pustaka `cryptography`) dan ML-DSA (pustaka `oqs`) |
| V11 | waktu cap waktu dalam jendela sesi |
| V12 | jumlah konfirmasi jangkar (hanya angka yang dilaporkan) |
| C1-C3 | panjang akhir, snapshot, event non-pengubah-teks |
| C4, C5, C7, C9 | properti dokumen, TotalTime, struktur ZIP, penghasil dokumen |
| C8 | jumlah rsid (**informasi saja**, hipotesis belum teruji) |
| C10 | konsistensi offset zona waktu di dalam log |
| P1-P4, P6 | interval terlalu cepat, variasi terlalu seragam, kecepatan melebihi batas, event tak wajar, sumber sintetis |
| D1, D2 | rasio paste, pola linear tanpa revisi (fitur, bukan vonis) |

## Batasan yang perlu kamu ketahui

1. **Token RFC 3161 tidak diparse.** V11 memeriksa `tsa_time` yang dilaporkan, bukan tanda tangan token.
2. **V12 tidak mengakses jaringan.** Hanya memeriksa angka konfirmasi dalam bundle.
3. **Tanda tangan PQC hanya terverifikasi bila `oqs` terpasang.** Tanpa itu V10 berstatus `skip` dan
   `integrity_fully_verified` bernilai `false`. Nama algoritma bergantung versi liboqs.
4. **C1 dan C2 berbasis panjang.** Karena karakter tidak disimpan, replay hanya membuktikan konsistensi
   panjang, bukan isi. Pemeriksaan isi butuh `doc_hash` yang dihitung dari teks di perangkat.
5. **C10 hanya memeriksa konsistensi internal log.** Properti dokumen .docx menyimpan waktu dalam UTC
   tanpa offset, jadi zona waktu tidak bisa dibandingkan dengan dokumen.
6. **C4 dan C7 mengandalkan cap waktu yang bisa diubah.** Cap waktu ZIP dan properti dokumen mudah
   dimanipulasi oleh penyerang yang memahaminya; hasil lolos bukan bukti keaslian.
7. **Docx contoh bukan keluaran Word asli.** Perilaku pada berkas Word sungguhan (urutan entri, rsid,
   TotalTime) harus diuji empiris sebelum aturan C dijadikan syarat.
8. **Serialisasi kanonik** setara RFC 8785 hanya karena event memuat integer, string, boolean, dan null.
   Float dilarang oleh schema.
9. **Data contoh adalah simulasi.** Hasil pada data simulasi tidak mengatakan apa pun tentang performa
   pada partisipan nyata.

## Berkas

- `schemas/`: JSON Schema untuk event, session, bundle
- `kf_validator.py`: validator dan CLI
- `make_sample.py`: generator sesi dan docx simulasi (termasuk penandatanganan)
- `test_kf_validator.py`: uji (jalur normal, pemalsuan, rantai dihitung ulang, docx)
- `examples/`: keluaran generator, plus `events_tampered.jsonl` (satu event diubah)
