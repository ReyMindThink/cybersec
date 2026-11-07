# Lab analisis jaringan: Wireshark, Suricata, Zeek

Satu rekaman trafik SSH dari lab sendiri dibaca tiga alat, lalu hasilnya dibandingkan. Status: **lab dasar, hands-on**.

## Bahan

Rekaman dibuat dengan `tcpdump` (4.99.4) di dalam container `lab-target`, selama 12 percobaan login SSH gagal dari laptop sendiri ke `127.0.0.1:2222`. Isi SSH terenkripsi, jadi berkas tidak memuat nama pengguna atau kata sandi.

| Item      | Nilai                                                                                                                                     |
| :-------- | :---------------------------------------------------------------------------------------------------------------------------------------- |
| Berkas    | `ssh_lab.pcap` (tidak ada di repo, ukuran 95.644 byte, 274 paket)                                                                         |
| SHA256    | `f359024050748e0d865e15a4c239c1747ebc8fe4c9dee06b088aa77c9d414145`                                                                        |
| Wireshark | 4.6.9                                                                                                                                     |
| Suricata  | 8.0.8, image `jasonish/suricata`, digest `sha256:d5a3a1785299ab40918d465e54fedf8e3afeeedba76bb64b0cc74dbc20bda2c3`, 53.393 aturan ET Open |
| Zeek      | 9.0.1, image `zeek/zeek`, digest `sha256:524cb0ce29f26a6a954fb9c357a8f92b95812476d8240f0ece26617ace249dd0`                                |

## Cara membuat rekaman

```
docker exec -d lab-target bash -c "timeout 45 tcpdump -i eth0 -w /tmp/ssh_lab.pcap port 22"
# lalu 12 kali: ssh -p 2222 -o BatchMode=yes admin@127.0.0.1 true
docker cp lab-target:/tmp/ssh_lab.pcap ssh_lab.pcap
```

Suricata dan Zeek dijalankan di container sementara dalam mode baca berkas (`suricata -r`, `zeek -C -r`). Rekaman dan log hasil disimpan di luar repo.

## Hasil per alat

| Alat      | Yang terlihat                                                                                                                                               |
| :-------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Wireshark | Pemeriksaan manual berkas: jabat tangan SSH dan awal tiap koneksi (lihat [screenshot](screenshots/01-wireshark-ssh.png))                                    |
| Suricata  | 12 `flow`, 12 `ssh`. Dengan `HOME_NET` bawaan tidak ada alert ancaman. Dengan `HOME_NET` dipersempit ke alamat target: 1 alert "ET SCAN Potential SSH Scan" |
| Zeek      | `conn.log` 12 koneksi (durasi sekitar 0,07 detik, status SF), `ssh.log` 12 entri: versi klien dan server, cipher, algoritma pertukaran kunci                |

## Apa yang dilihat jaringan dan apa yang tidak

Ketiga alat sepakat ada 12 sesi SSH pendek dari `172.20.0.1` ke `172.20.0.5:22`. Tetapi isi SSH terenkripsi: kolom `auth_success` dan `auth_attempts` di `ssh.log` Zeek kosong dan 0, jadi dari jaringan tidak terlihat siapa yang mencoba masuk. Wazuh, yang membaca log di dalam host (lihat [`labs/wazuh`](../wazuh/)), mencatat pengguna yang tidak ada (rule 5710) dan brute force (rule 5712). Kedua sudut pandang saling melengkapi.

## Catatan dan batasan

- **274 alert "invalid checksum" dari Suricata adalah artefak rekaman**, bukan deteksi serangan: rekaman dibuat di dalam container sebelum checksum dihitung ulang. Opsi `-k none` tidak menghilangkannya dan penyebabnya belum saya telusuri.
- **Mengubah `HOME_NET` adalah penyesuaian saya sendiri**, dan hasilnya satu percobaan: dengan nilai bawaan tidak ada alert pemindaian, dengan nilai dipersempit ada satu. Saya menduga penyebabnya alamat sumber `172.20.0.1` (rentang privat) dianggap jaringan dalam. Belum diuji lebih jauh.
- Satu rekaman kecil dan benign dari lab sendiri, bukan trafik malware atau trafik produksi. Tidak ada klaim kemampuan deteksi di luar itu.
- Snort tidak dicoba.
