# Lab Wazuh: deteksi brute force SSH

Lab SIEM skala kecil yang dikerjakan langsung di laptop sendiri (7 November 2026). Status: **lab dasar, hands-on**.

## Tujuan

Menjalankan Wazuh single-node di Docker, menghubungkan satu agent, mensimulasikan login SSH gagal ke target milik sendiri, membaca alert yang muncul, memetakannya ke MITRE ATT&CK, dan menulis satu aturan deteksi kustom.

## Arsitektur

```
Laptop (Windows 11, Docker Desktop + WSL 2)
 ├─ lab-wazuh   : manager + indexer + dashboard (Wazuh 4.14.9, port hanya ke 127.0.0.1)
 └─ lab-target  : Ubuntu 24.04 + sshd + rsyslog + agent Wazuh 4.14.9
        ▲
        └─ 12 percobaan login SSH gagal (dari host ke 127.0.0.1:2222)
```

## Lingkungan

| Komponen                                   | Versi                    |
| :----------------------------------------- | :----------------------- |
| Wazuh (manager, indexer, dashboard, agent) | 4.14.9                   |
| Docker / Compose / Docker Desktop          | 29.9.2 / v5.6.1 / 4.98.0 |
| Target                                     | Ubuntu 24.04 (container) |

## Langkah singkat

1. Clone `wazuh-docker` tag `v4.14.9` (di luar repo ini), buat sertifikat dengan generator resmi, ikat semua port ke `127.0.0.1`, jalankan dengan nama proyek `lab-wazuh`.
2. Buat container `lab-target` (sshd + rsyslog), pasang agent versi sama dengan manager, daftarkan ke manager.
3. Tambahkan `/var/log/auth.log` sebagai sumber log di konfigurasi agent.
4. Simulasikan 12 percobaan login dengan pengguna yang tidak ada ke `127.0.0.1:2222`.
5. Tambahkan aturan kustom dan uji.

Dashboard memakai kredensial bawaan Wazuh dan hanya dapat diakses dari localhost.

## Hasil

| Rule ID | Level | Jumlah | Arti                                           |
| :------ | :---- | :----- | :--------------------------------------------- |
| 5710    | 5     | 11     | Percobaan login dengan pengguna yang tidak ada |
| 5712    | 10    | 1      | Brute force terdeteksi                         |
| 100100  | 12    | 1      | Aturan kustom (lihat di bawah)                 |

Dashboard mencatat 12 authentication failure, sesuai jumlah percobaan. Di MITRE ATT&CK terlihat taktik Credential Access, Lateral Movement, dan Defense Evasion, dengan teknik Password Guessing, SSH, Brute Force, dan Disable or Modify Tools. Penyebab alert Defense Evasion belum dianalisis.

Bukti: [agent aktif](screenshots/01-agent-active.png), [Threat Hunting](screenshots/02-threat-hunting.png), [MITRE ATT&CK](screenshots/03-mitre.png), [aturan kustom](screenshots/04-custom-rule.png).

## Aturan kustom

[`rules/lab_custom_rules.xml`](rules/lab_custom_rules.xml): rule 100100 (level 12) dipasang di bawah rule 5712 dan menaikkan prioritas saat brute force mengarah ke nama akun `admin`. Terpicu satu kali pada uji.

## Kendala yang ditemui

| Gejala                                         | Penyebab                                          | Perbaikan                                                    |
| :--------------------------------------------- | :------------------------------------------------ | :----------------------------------------------------------- |
| Push ditolak (GH007)                           | Email commit pribadi tidak boleh dipublikasikan   | Memakai email noreply GitHub di repo ini                     |
| Perintah di container gagal membaca `/var/...` | Git Bash mengubah path Linux menjadi path Windows | Awali dengan `MSYS_NO_PATHCONV=1`                            |
| Alert SSH tidak muncul                         | Agent belum diperintahkan membaca `auth.log`      | Menambahkan blok `localfile`, restart agent, ulangi simulasi |
| Lab terhapus sendiri                           | Perintah `down -v` ikut menghapus volume          | Menyalakan ulang dari awal (image tetap ada)                 |

## Batasan

- Ini simulasi lokal, bukan serangan nyata. Alamat sumber adalah gerbang jaringan Docker (`172.20.0.1`).
- Alert dari pemindaian CIS (SCA) otomatis ikut tercatat. Sebagian menyebut policy yang tidak sesuai target. Belum saya telusuri.
- Satu agent saja, tanpa tuning, tidak untuk produksi.
