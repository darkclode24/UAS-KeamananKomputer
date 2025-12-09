d# 🛡️ Penetration Testing: Automatic Clothesline Control (ESP8266) Laporan analisis keamanan pada perangkat IoT "Jemuran Otomatis Pintar" berbasis ESP8266 terhadap kerentanan Authentication Bypass, Denial of Service (DoS), dan Network Sniffing. ---

## 👥 Anggota Kelompok II

Berikut adalah tim yang terlibat dalam penyusunan laporan dan pengujian ini:

| No | Nama                         | NIM            |
|----|------------------------------|----------------|
| 1  | **Bintang Darmawan**         | 09030000000000 |
| 2  | **Hana Zakiyah Nur Aliyah**  | 09030282327053 |
| 3  | **Niken Carolin**            | 09030282327071 |
| 4  | **Sania Apriza**             | 09030282327059 |
| 5  | **Daniel Juankarel P. S.**   | 09030000000000 |
| 6  | **Ario Faturrahmat**         | 09030000000000 |


---

## 📌 Latar Belakang & Tujuan
Proyek ini bertujuan untuk menguji keamanan antarmuka kontroler yang berfungsi menggerakkan motor jemuran otomatis. Perangkat ini dirancang untuk efisiensi rumah tangga dengan menarik jemuran saat hujan turun menggunakan sensor cuaca. **Tujuan Pengujian (Graybox Method):** 1.  **Reconnaissance & Enumeration:** Memetakan jaringan lokal untuk menemukan perangkat target menggunakan protokol ARP dan scanning port. 2.  **Exploitation:**
    * Menguji celah **Broken Access Control** pada API kontrol motor. * Menguji ketahanan server terhadap serangan **Denial of Service (DoS)** tipe Socket Exhaustion. * Membuktikan kerentanan privasi melalui **Network Sniffing**. ---

## 🤖 Deskripsi Perangkat (Smart Clothesline)
Sebelum masuk ke pengujian keamanan, berikut adalah gambaran umum perangkat keras yang digunakan. ### 1. Cara Kerja
Sistem bekerja secara otomatis maupun manual:
* **Sensor Hujan:** Mendeteksi air hujan untuk memicu motor menarik jemuran. * **Limit Switch:** Keamanan posisi motor agar tidak berputar melebihi batas. * **Web Emergency Panel:** Fitur kendali manual via WiFi (Port 80) jika internet utama terputus. ### 2. Komponen Utama
Berikut adalah komponen yang membangun sistem ini: | Komponen | Fungsi |
| :--- | :--- |
| **ESP8266 (NodeMCU/Wemos)** | Mikrokontroler utama yang terhubung ke Wi-Fi & Web Server. |
| **Motor DC** | Aktuator untuk menarik dan mengulur tali jemuran. |
| **Rain Sensor** | Mendeteksi kondisi cuaca (Hujan/Cerah). |
| **Web Interface** | Dashboard "Emergency Web Panel" pada Port 80 untuk kendali manual. |

---

## 🎯 Target Pengujian
Bagian ini berfokus pada sisi *software* dan *network* yang berjalan di atas perangkat keras tersebut. * **Perangkat:** ESP8266 Web Server (Jemuran Otomatis).
* **Fitur Web:** Emergency Control Panel (Maju, Mundur, Stop). * **IP Target:** `10.185.29.155` (Berdasarkan hasil scanning ARP). * **Port Terbuka:** 80/tcp (HTTP). **Deskripsi Web Service:**
Web service menyediakan antarmuka sederhana yang memungkinkan pengguna mengirim perintah ke motor tanpa login yang memadai. | Tampilan Web Panel (Normal) | Hasil Scanning Port (Nmap) |
| :---: | :---: |
| <img src="URL_GAMBAR_WEB_PANEL_DISINI" width="100%" alt="Tampilan Web Panel"> | <img src="URL_GAMBAR_NMAP_DISINI" width="100%" alt="Hasil Nmap"> |
| *Antarmuka Kendali Manual* | *Port 80 HTTP Open* |

---

## 🛠️ Tools & Lingkungan Pengujian
Pengujian dilakukan menggunakan sistem operasi **Kali Linux** (`10.185.29.164`) dengan tools berikut: 1.  **Arp-scan / Nmap:** Untuk penemuan perangkat dan pemindaian port (*Reconnaissance*).
2.  **cURL:** Untuk memanipulasi permintaan HTTP dan bypass antarmuka web.
3.  **Tcpdump:** Untuk penyadapan (*sniffing*) lalu lintas jaringan.
4.  **Python Script:** Custom script untuk melakukan stress-testing (DoS).

---

## ⚔️ Skenario Serangan 1: Authentication Bypass (IDOR)

### Apa itu IDOR?
**Insecure Direct Object Reference (IDOR)** atau dalam kasus ini *Broken Access Control*, terjadi ketika aplikasi memberikan akses ke fungsi perangkat keras (motor) berdasarkan input pengguna tanpa memvalidasi otorisasi. Pada perangkat ini, API endpoint `/action` terekspos tanpa proteksi password. ### 🚀 Langkah-Langkah Pengujian

#### 1. Analisis Struktur HTML
Melalui *Inspect Element*, ditemukan bahwa tombol kontrol memanggil endpoint berikut: * `href="/action?go=maju"`
* `href="/action?go=mundur"`

#### 2. Eksekusi Serangan (cURL)
Penyerang tidak perlu membuka browser. Cukup menggunakan terminal untuk mengirim perintah langsung ke mikrokontroler. **Command:**
```bash
curl "[http://10.185.29.155/action?go=maju](http://10.185.29.155/action?go=maju)"
