# Playbook insiden phishing (dokumen, lab dasar)

Playbook ringkas untuk menangani satu laporan email phishing, ditulis sebagai latihan. Status: **dokumen, belum diuji pada insiden nyata**.

## Acuan

- NIST SP 800-61 **Rev. 3** (April 2025), menggantikan Rev. 2 (2012). Rev. 3 tidak memakai siklus empat fase tetap, melainkan memetakan respons insiden ke enam fungsi NIST CSF 2.0 (Govern, Identify, Protect, Detect, Respond, Recover). Tahap di bawah adalah susunan praktis saya, dipetakan ke fungsi itu.
- MITRE ATT&CK **T1566 Phishing** (Initial Access): T1566.001 Spearphishing Attachment, T1566.002 Spearphishing Link, T1566.003 Spearphishing via Service, T1566.004 Spearphishing Voice. Teknik terkait: T1204 User Execution.

## Alur

| Tahap                 | Fungsi CSF 2.0            | Tindakan utama                                                                                         | Bukti yang dikumpulkan               |
| :-------------------- | :------------------------ | :----------------------------------------------------------------------------------------------------- | :----------------------------------- |
| 0. Persiapan          | Govern, Identify, Protect | Tombol "laporkan email", daftar kontak insiden, akses ke gateway email dan log                         | Daftar peran, daftar sumber log      |
| 1. Deteksi dan triase | Detect                    | Terima laporan, simpan email asli (berkas .eml), beri tingkat awal                                     | Email asli beserta header lengkap    |
| 2. Analisis           | Detect                    | Periksa pengirim, header, tautan, lampiran. Cek apakah ada penerima lain                               | Tabel IOC, daftar penerima           |
| 3. Penahanan          | Respond                   | Blokir domain/pengirim, tarik email dari kotak masuk lain, reset kredensial bila ada yang mengisi form | Catatan tindakan beserta waktu       |
| 4. Pemberantasan      | Respond                   | Hapus lampiran atau persistensi, pastikan tidak ada penerusan atau aturan kotak masuk berbahaya        | Hasil pemeriksaan akun dan perangkat |
| 5. Pemulihan          | Recover                   | Cabut blokir sementara yang tidak perlu, pantau akun terdampak                                         | Catatan pemantauan                   |
| 6. Pelajaran          | Govern, Identify          | Tinjau kronologi, perbarui aturan deteksi dan pelatihan                                                | Ringkasan pasca-insiden              |

## Triase awal

| Tingkat | Kriteria contoh                                                              |
| :------ | :--------------------------------------------------------------------------- |
| Rendah  | Dilaporkan, tidak ada yang mengeklik atau membuka lampiran                   |
| Sedang  | Ada yang mengeklik tautan, belum ada kredensial atau eksekusi                |
| Tinggi  | Kredensial dimasukkan, lampiran dijalankan, atau beberapa penerima terdampak |

## Kriteria eskalasi

Naikkan ke penanggung jawab keamanan bila: ada kredensial yang dimasukkan, ada lampiran yang dijalankan, penerima lebih dari satu kelompok, atau pengirim memakai akun internal yang diambil alih.

## Catatan insiden (templat)

```
ID insiden      :
Waktu laporan   :
Pelapor         :
Ringkasan       :
Tingkat         : rendah / sedang / tinggi
Teknik MITRE    : T1566.00x
IOC             : (lihat tabel)
Tindakan + waktu:
Status          : terbuka / ditahan / selesai
Pelajaran       :
```

## IOC

Semua nilai di tabel ini **fiktif** (domain `.example`, alamat dari rentang dokumentasi `203.0.113.0/24`), hanya untuk menunjukkan bentuk. IOC selalu dicatat dalam bentuk _defanged_ supaya tidak bisa diklik tidak sengaja: `http` menjadi `hxxp`, titik menjadi `[.]`, `@` menjadi `[at]`.

| Jenis         | Contoh (fiktif)                       | Catatan                                              |
| :------------ | :------------------------------------ | :--------------------------------------------------- |
| Domain tautan | `login-verify[.]example`              | Domain saja, tanpa path                              |
| URL           | `hxxps://login-verify[.]example/akun` | Hanya bentuk defanged                                |
| Pengirim      | `support[at]bank-palsu[.]example`     | Periksa juga Return-Path dan Reply-To                |
| IP pengirim   | `203[.]0[.]113[.]7`                   | Dari header Received paling awal yang bisa dipercaya |
| Hash lampiran | (SHA256 berkas)                       | Jangan menyimpan berkasnya di repo                   |
| Pola subjek   | "Verifikasi akun Anda segera"         | Indikator lemah, mudah berubah                       |

IOC itu murah diubah penyerang, sedangkan teknik (TTP) lebih sulit diganti. Karena itu playbook mencatat keduanya, dan aturan deteksi di bawah menyasar teknik, bukan satu domain.

## Peran PhishGuard AI

[PhishGuard AI](https://github.com/ReyMindThink/phishing-detection-ml) (proyek terpisah, aplikasi lokal) dapat menjadi **sinyal tambahan di tahap 1 dan 2**: memberi skor dan pita risiko pada email yang dilaporkan untuk membantu urutan triase. Ia **bukan pengambil keputusan**, dan analis tetap menentukan. Keterbatasannya tercatat di repo itu, termasuk uji eksternal yang menunjukkan kerapuhan atau artefak format. Ini lab, bukan produk produksi.

## Deteksi

Satu aturan Sigma: [`sigma/proc_creation_win_office_spawn_shell.yml`](sigma/proc_creation_win_office_spawn_shell.yml), mendeteksi aplikasi Office yang memunculkan shell (pola pasca-lampiran, T1566.001 dan T1204). Sintaksnya divalidasi dengan `sigma check`. **Belum diuji pada log nyata** karena lab ini tidak memiliki endpoint Windows dengan log pembuatan proses.

## Batasan

- Playbook ini dokumen latihan, belum pernah dipakai pada insiden nyata.
- Alat dan integrasi (gateway email, EDR, SIEM) bersifat generik dan tidak dikonfigurasi di lab ini.
- Semua IOC pada contoh fiktif.
