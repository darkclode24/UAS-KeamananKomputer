# Penetration Testing: Kontrol Jemuran Otomatis (ESP8266)

---

## 👥 Anggota Kelompok II

| No | Nama                         | NIM            |
|----|------------------------------|----------------|
| 1  | **Bintang Darmawan**         | 09030282327017 |
| 2  | **Hana Zakiyah Nur Aliyah**  | 09030282327053 |
| 3  | **Niken Carolin**            | 09030282327071 |
| 4  | **Sania Apriza**             | 09030282327059 |
| 5  | **Ario Faturrahmat**         | 09030282327074 |
| 6  | **Seftiana Safutri**         | 09030282327056 |

---

## Latar Belakang & Tujuan
Proyek ini bertujuan untuk menguji keamanan antarmuka kontroler yang berfungsi menggerakkan motor jemuran otomatis. Perangkat ini dirancang untuk efisiensi rumah tangga dengan menarik jemuran saat hujan turun menggunakan sensor cuaca. 

**Tujuan Pengujian (Graybox Method):**
1. Memetakan jaringan lokal untuk menemukan perangkat target menggunakan protokol ARP dan scanning port.
2. Menguji celah **Broken Access Control** pada API kontrol motor.
3. Menguji ketahanan server terhadap serangan **Denial of Service (DoS)** tipe Socket Exhaustion.
4. Membuktikan kerentanan privasi melalui **Network Sniffing**.

---

## Deskripsi Perangkat (Smart Clothesline)

Sebelum masuk ke pengujian keamanan, berikut adalah gambaran umum perangkat keras yang digunakan.

### 1. Cara Kerja
Sistem bekerja secara otomatis maupun manual:

* **Sensor Hujan:** Mendeteksi air hujan untuk memicu motor menarik jemuran.
* **Limit Switch:** Keamanan posisi motor agar tidak berputar melebihi batas.
* **Web Emergency Panel:** Fitur kendali manual via WiFi (Port 80) jika internet utama terputus.

### 2. Komponen Utama
Berikut adalah komponen yang membangun sistem ini:

| Komponen                    | Fungsi                                                                 |
| :-------------------------- | :---------------------------------------------------------------------- |
| **ESP8266 (NodeMCU/Wemos)** | Mikrokontroler utama yang terhubung ke Wi-Fi dan Web Server.           |
| **Motor DC 755**                | Aktuator untuk menarik dan mengulur tali jemuran.                      |
| **Rain Sensor**             | Mendeteksi kondisi cuaca (hujan/cerah).                                |
| **Web Interface**           | Dashboard "Emergency Web Panel" pada Port 80 untuk kendali manual.     |


---

## 🎯 Target Pengujian

Bagian ini berfokus pada sisi *software* dan *network* yang berjalan di atas perangkat keras tersebut.

* **Perangkat:** ESP8266 Web Server (Jemuran Otomatis)
* **Fitur Web:** Emergency Control Panel (Maju, Mundur, Stop)
* **IP Target:** `10.185.29.155` (berdasarkan hasil scanning ARP)
* **Port Terbuka:** 80/tcp (HTTP)

**Deskripsi Web Service:**  
Web service menyediakan antarmuka sederhana yang memungkinkan pengguna mengirim perintah ke motor tanpa mekanisme login yang memadai.

| Tampilan Web Panel (Normal) | Hasil Scanning Port (Nmap) |
| :-------------------------: | :-------------------------: |
| <img src="https://github.com/user-attachments/assets/2563874a-8191-41b6-96c5-41c90a157dde" width="100%" alt="Tampilan Web Panel"> | <img src="https://github.com/user-attachments/assets/abb6a74d-83c8-403f-a3c8-2cf220dd98a4" width="100%" alt="Hasil Nmap"> |
| *Antarmuka Kendali Manual* | *Port 80 HTTP Open* |

---

## 🛠️ Tools & Lingkungan Pengujian
Pengujian dilakukan menggunakan sistem operasi **Kali Linux** (`10.185.29.164`) dengan tools berikut: 
1.  **Arp-scan / Nmap:** Untuk penemuan perangkat dan pemindaian port (*Reconnaissance*).
2.  **cURL:** Untuk memanipulasi permintaan HTTP dan bypass antarmuka web.
3.  **Tcpdump:** Untuk penyadapan (*sniffing*) lalu lintas jaringan.
4.  **Python Script:** Custom script untuk melakukan stress-testing (DoS).

---

## ⚔️ Kerentanan 1: Authentication Bypass (IDOR)

### Apa itu IDOR?
**Insecure Direct Object Reference (IDOR)** atau dalam kasus ini *Broken Access Control*, terjadi ketika aplikasi memberikan akses ke fungsi perangkat keras (motor) berdasarkan input pengguna tanpa memvalidasi otorisasi. Pada perangkat ini, API endpoint `/action` terekspos tanpa proteksi password. ### 🚀 Langkah-Langkah Pengujian

#### 1. Analisis Struktur HTML
Pengujian dilakukan langsung dengan mengirim permintaan HTTP manual menggunakan cURL, ditemukan bahwa tombol kontrol memanggil endpoint berikut: 
* `href="/action?go=maju"`
* `href="/action?go=mundur"`
* `href="/action?go=stop"`

| Hasil cURL  | 
|---------------------------|
| <img src="https://github.com/user-attachments/assets/ffebd5de-05b7-42bc-bd4c-8ddef199563c" width="100%" /> |


#### 2. Eksekusi Serangan (cURL)
Penyerang tidak perlu membuka browser. Cukup menggunakan terminal untuk mengirim perintah langsung ke mikrokontroler. 
**Command:**
```bash
curl "[http://10.185.29.155/action?go=maju](http://10.185.29.155/action?go=maju)"   
```
| Eksekusi cURL pada ESP8266 |
|----------------------------|
| <img src="https://github.com/user-attachments/assets/836f9d78-22fc-461d-8151-13d7c5f68c90" width="100%"> |



#### 3. Hasil
Perintah diterima dan motor bergerak tanpa autentikasi. Sistem tidak memeriksa apakah pengirim adalah pengguna sah, hanya mengeksekusi parameter `go`.


| Hasil Bypass pada Alat |
|----------------------------|
|<img width="855" height="386" alt="image" src="https://github.com/user-attachments/assets/2ade9f55-6919-4630-b47a-6591055d3c1c" />|



**Dampak:**  
Aktor yang berada dalam jaringan yang sama dapat menggerakkan motor kapan saja, termasuk pada kondisi yang berpotensi merusak perangkat (misalnya memaksa motor bergerak saat limit switch mencapai batas).

---

## ⚔️ Kerentanan 2: Denial of Service (DoS) — Socket Exhaustion

### Ringkasan
Web server bawaan ESP8266 hanya mampu menangani sejumlah kecil koneksi simultan. Tidak ada mekanisme rate limiting atau timeout yang memadai, sehingga koneksi dapat dibiarkan terbuka sampai kapasitas habis.

### Langkah Pengujian

#### 1. Pembuatan Script Flooding
Sebuah script Python sederhana digunakan untuk membuka koneksi HTTP tanpa menutupnya.

```python
import socket
import time

TARGET_IP = "10.185.29.155"
TARGET_PORT = 80

while True:
    try:
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.connect((TARGET_IP, TARGET_PORT))
        s.send(b"GET / HTTP/1.1\r\nHost: x\r\n\r\n")
    except:
        pass
```

#### 2. Hasil
Setelah beberapa detik, web server berhenti merespons. Browser menampilkan **Connection timed out**, dan cURL gagal menjalin koneksi baru.

**Dampak:**  
Penyerang dapat membuat panel kontrol tidak dapat diakses pengguna sah, sehingga mengganggu fungsi sistem.

---

## ⚔️ Kerentanan 3: Network Sniffing — Plaintext HTTP Leakage

### Ringkasan
Seluruh komunikasi berlangsung melalui HTTP tanpa enkripsi. Hal ini memungkinkan pihak lain dalam jaringan menangkap dan membaca isi paket.

### Langkah Pengujian

#### 1. Menjalankan Tcpdump
```bash
sudo tcpdump -i eth0 -nn -A host 10.185.29.155
```

#### 2. Hasil
Permintaan ke endpoint `/action?go=maju` muncul dalam bentuk teks biasa, termasuk parameter kontrol.

**Dampak:**  
Aktor pasif dalam jaringan dapat mengetahui kapan pengguna menekan tombol dan dapat meniru permintaan tersebut.

---

## 🔍 Ringkasan Kerentanan

| Kerentanan                    | Dampak Utama                                          | Tingkat Risiko |
|------------------------------|--------------------------------------------------------|----------------|
| Authentication Bypass (IDOR) | Motor dapat dikendalikan pihak tidak sah               | Tinggi         |
| DoS: Socket Exhaustion       | Panel kontrol tidak tersedia bagi pengguna sah         | Sedang-Tinggi  |
| Sniffing (No Encryption)     | Data perintah dapat dicuri dan dipalsukan              | Sedang         |

---

## 🛡️ Rekomendasi Mitigasi

1. **Tambah Autentikasi:** Gunakan token, sesi, atau minimal password untuk endpoint sensitif.  
2. **Implementasi Rate Limiting:** Tutup koneksi idle dan batasi jumlah koneksi simultan.  
3. **Gunakan HTTPS atau WPA2-Enterprise:** Jika HTTPS tidak memungkinkan, pastikan jaringan terisolasi.  
4. **Validasi Input:** Batasi nilai parameter `go` hanya pada daftar perintah valid.  
5. **Segregasi Jaringan:** Tempatkan perangkat IoT pada VLAN atau segmen khusus.

---

## 📌 Kesimpulan

ESP8266 pada konfigurasi default tidak dirancang untuk menghadapi ancaman jaringan lokal yang agresif. Tanpa autentikasi, tanpa enkripsi, dan tanpa manajemen koneksi, perangkat sangat mudah dieksploitasi. Hasil pengujian menunjukkan seluruh fungsi kritis bisa diambil alih atau dilumpuhkan oleh pihak yang berada pada jaringan yang sama.  

