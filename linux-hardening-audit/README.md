# Audit dan hardening Linux (lab dasar)

Audit konfigurasi satu target Ubuntu 24.04 (container `lab-target`) dengan dua alat: SCA Wazuh (CIS Ubuntu 24.04 v1.0.0) dan Lynis 3.0.9, lalu penerapan beberapa perbaikan SSH dan pengukuran ulang. Status: **lab dasar, hands-on**.

## Batasan penting

Target adalah **container**, bukan server atau VM penuh. Banyak pengecekan CIS (modul kernel, partisi, bootloader) tidak berlaku atau tidak bisa diubah di container. Karena itu skor SCA dan Lynis di sini bukan ukuran keamanan server nyata, dan tidak diklaim sebagai pemenuhan CIS Benchmark.

## Hasil pengukuran

| Ukuran                | Sebelum       | Sesudah |
| :-------------------- | :------------ | :------ |
| SCA Wazuh: Passed     | 94            | ... |
| SCA Wazuh: Failed     | 107           | ... |
| SCA Wazuh: Score      | 46%           | ...% |
| Lynis hardening index | 58            | 58      |
| Saran Lynis           | 53            | 51      |
| Warning Lynis         | 1 (PKGS-7392) | 0       |

Skor SCA dihitung dari Passed dibagi (Passed + Failed). Pengecekan Not applicable (78) tidak ikut dihitung.

## Perbaikan yang diterapkan

Semuanya lewat satu berkas drop-in [`99-lab-hardening.conf`](99-lab-hardening.conf) untuk sshd, ditambah izin berkas.

| Pengaturan            | Sebelum          | Sesudah        |
| :-------------------- | :--------------- | :------------- |
| `PermitRootLogin`     | without-password | no             |
| `MaxAuthTries`        | 6                | 4              |
| `LoginGraceTime`      | 120              | 60             |
| `ClientAliveInterval` | 0                | 15             |
| `DisableForwarding`   | no               | yes            |
| `LogLevel`            | INFO             | VERBOSE        |
| `Banner`              | none             | /etc/issue.net |
| Izin `sshd_config`    | 644              | 600            |

Pembaruan paket: hanya `libpcre2-8-0` (pembaruan keamanan Ubuntu). Setelah itu warning Lynis PKGS-7392 hilang. Ini korelasi, bukan pembuktian penyebab.

## Keputusan yang disengaja

- **`wazuh-agent` ditahan di 4.14.9 (`hold`).** Repositori stabil sudah menawarkan 4.14.10, tetapi dokumentasi Wazuh menyebut kompatibilitas terjamin bila versi manager sama atau lebih baru daripada agent. Manager lab ini 4.14.9.
- **`AllowUsers`/`AllowGroups` tidak diterapkan**, walau SCA menyarankannya. Dugaan saya, penolakan lewat daftar izin menghasilkan pesan log yang berbeda dari "Invalid user", sehingga bisa memutus aturan deteksi pada lab Wazuh (5710, 5712, 100100). Ini belum diuji.
- **Cipher, MAC, dan KexAlgorithms tidak diubah.** Salah ubah bisa memutus koneksi dan butuh kajian tersendiri.

## Verifikasi bahwa deteksi tidak rusak

Setelah hardening, 12 percobaan login gagal baru menambah 11 alert rule 5710 di Wazuh (dari 22 menjadi 33), sama seperti sebelum hardening.

## Catatan

- Lynis berasal dari repositori Ubuntu (3.0.9), bukan rilis terbaru proyeknya.
- Lynis masih menyarankan `AllowTcpForwarding`, `X11Forwarding`, dan `AllowAgentForwarding` walau `DisableForwarding yes` aktif. Kemungkinan Lynis memeriksa arahan itu secara literal. Belum diverifikasi.
- Lynis menyarankan `MaxAuthTries 3`, sedangkan lab ini memakai 4 (acuan CIS).
- Perubahan hanya berlaku di container; berkas drop-in disimpan di repo agar bisa diulang.

## Bukti

[SCA sebelum](screenshots/05-sca-before.png), [SCA sesudah](screenshots/06-sca-after.png).
