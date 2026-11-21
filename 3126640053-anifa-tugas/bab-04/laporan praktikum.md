# Bab 4: Web Service Container: Apache, Nginx, Reverse Proxy, dan TLS
<div align="center">
  <img src="./assets/halaman-01-gambar-01.jpeg" alt="Logo PENS">
  <h3 style="text-align: center;">Disusun Oleh : </h3>
  <p style="text-align: center;">
    <strong>Anifa Aulia Abdari (3126640053)</strong><br>
    Mata Kuliah : Workshop DevOps<br>
    Dosen Pengampu : Dr. Ferry Astika Saputra, S.T., M.Sc.<br>
    Tanggal pelaksanaan : 3 Oktober 2026<br>
  </p>
  <h3 style="text-align: center;line-height: 1.5">Politeknik Elektronika Negeri Surabaya<br>Departemen Teknik Informatika Dan Komputer<br>Program Studi Teknik Informatika<br>Kelas B - StrLJ D4 Teknik Informatika<br>2026</h3>
  <hr><hr>
</div>

---

## Daftar Isi

- [1. Tujuan Praktikum](#1-tujuan-praktikum)
- [2. Dasar Teori Singkat](#2-dasar-teori-singkat)
- [3. Alat dan Bahan](#3-alat-dan-bahan)
- [4. Langkah Praktikum](#4-langkah-praktikum)
  - [4.1 Menyiapkan Direktori Kerja dan Struktur Proyek](#41-menyiapkan-direktori-kerja-dan-struktur-proyek)
  - [4.2 Membuat Sertifikat TLS Laboratorium (Self-Signed)](#42-membuat-sertifikat-tls-laboratorium-self-signed)
  - [4.3 Membuat File compose.yaml](#43-membuat-file-composeyaml)
  - [4.4 Membuat Konfigurasi Reverse Proxy Nginx (default.conf)](#44-membuat-konfigurasi-reverse-proxy-nginx-defaultconf)
  - [4.5 Membuat Halaman Web Statis Apache (index.html)](#45-membuat-halaman-web-statis-apache-indexhtml)
  - [4.6 Membuat Requirements Flask](#46-membuat-requirements-flask)
  - [4.7 Membuat Source Code Backend Flask (app.py)](#47-membuat-source-code-backend-flask-apppy)
  - [4.8 Membuat Dockerfile](#48-membuat-dockerfile)
  - [4.9 Validasi Konfigurasi Docker Compose](#49-validasi-konfigurasi-docker-compose)
  - [4.10 Membangun Image dan Menjalankan Stack Compose](#410-membangun-image-dan-menjalankan-stack-compose)
  - [4.11 Pengujian Fungsional Web Ingress dan API](#411-pengujian-fungsional-web-ingress-dan-api)
  - [4.12 Pemeriksaan Boundary dan Isolasi Jaringan](#412-pemeriksaan-boundary-dan-isolasi-jaringan)
  - [4.13 Pemeriksaan dan Audit Log](#413-pemeriksaan-dan-audit-log)
  - [4.14 Membersihkan Lingkungan (Cleanup)](#414-membersihkan-lingkungan-cleanup)
- [5. Hasil Pengujian](#5-hasil-pengujian)
- [6. Evaluasi dan Latihan Mandiri](#6-evaluasi-dan-latihan-mandiri)
- [7. Threat Statement](#7-threat-statement)
- [8. Analisis](#8-analisis)
- [9. Tindak Lanjut](#9-tindak-lanjut)
- [10. Kesimpulan](#10-kesimpulan)
- [11. Referensi](#11-referensi)

---

## 1. Tujuan Praktikum

Praktikum Bab 4 dirancang untuk memberikan pemahaman konseptual dan pengalaman praktis dalam merancang, mengorkestrasi, mengamankan, dan mengaudit web service multi-tier berbasis container menggunakan Docker Compose. Secara rinci, praktikum ini bertujuan agar mahasiswa mampu:

1. Menjalankan Apache HTTP Server dan Nginx sebagai web server container dengan konfigurasi kustom (*custom configuration*).
2. Memahami perbedaan arsitektur, karakteristik konkurensi, dan *use-case* antara Apache httpd dan Nginx.
3. Menerapkan sertifikat TLS (*self-signed certificate*) menggunakan OpenSSL dengan ekstensi Subject Alternative Name (SAN) untuk simulasi HTTPS lokal.
4. Mengonfigurasi Nginx sebagai reverse proxy cerdas yang melakukan *path-based routing*, terminasi TLS (TLS offloading), dan pengalihan otomatis (301 permanent redirect) dari HTTP ke HTTPS.
5. Menerapkan HTTP Security Headers (`X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`) pada layer reverse proxy untuk memperkuat pertahanan aplikasi.
6. Membangun container backend Python Flask yang berjalan menggunakan WSGI server *production-ready* (Gunicorn) di bawah akun non-root user.
7. Mengimplementasikan service healthcheck pada Docker Compose untuk memvalidasi kesiapan (*readiness*) layanan backend sebelum menerima trafik.
8. Menegakkan prinsip segregasi jaringan (*network isolation*), di mana hanya port reverse proxy yang dipublikasikan ke host, sedangkan origin backend terisolasi di internal network.
9. Membaca, menganalisis, dan mengekstrak bukti audit melalui access log dan error log yang dipetakan secara persisten ke direktori host via bind mount.
10. Mendiagnosis kendala konfigurasi runtime serta menyusun analisis risiko keamanan dan rekomendasi arsitektur produksi.

---

## 2. Dasar Teori Singkat

Dalam lanskap komputasi modern dan ekosistem cloud-native, web service merupakan pintu gerbang utama (*ingress boundary*) yang menghubungkan pengguna eksternal dengan sistem aplikasi dan data bisnis di lingkungan internal. Penerapan containerization menggunakan Docker memungkinkan aplikasi web beserta web server pendukungnya dikemas secara portabel, terisolasi, dan mudah direplikasi di berbagai tahapan pipeline DevSecOps.

Arsitektur aplikasi modern jarang sekali mengekspos server aplikasi (*application origin server*) secara langsung ke jaringan publik. Pendekatan yang lazim dan direkomendasikan adalah menerapkan Reverse Proxy di baris terdepan. Nginx sering kali difungsikan sebagai *edge reverse proxy* yang bertugas mengelola terminasi enkripsi TLS/HTTPS (TLS Offloading), melakukan Layer-7 routing (mengarahkan rute statis ke web server Apache httpd dan rute dinamis ke backend API Python Flask), serta menerapkan filter keamanan berupa HTTP security headers. Selain itu, prinsip keamanan *Least Exposure* mengharuskan server backend seperti Apache dan Flask berjalan dalam *isolated private bridge network* tanpa membuka *published port* langsung ke host, sehingga seluruh lalu lintas data (*traffic flow*) wajib melewati gerbang inspeksi proxy.

**Apache httpd vs Nginx.** Apache HTTP Server (httpd) dan Nginx adalah dua web server terkemuka dengan filosofi desain dan arsitektur konkurensi yang sangat berbeda.

- **Apache httpd** menggunakan model Multi-Processing Modules (MPM) seperti *prefork*, *worker*, atau *event*. Pada model klasik prefork, setiap koneksi klien ditangani oleh proses terpisah, sedangkan pada worker/event koneksi ditangani oleh kombinasi proses dan thread. Apache sangat kaya fitur, mendukung konfigurasi terdesentralisasi per direktori melalui `.htaccess`, dan memiliki modul dinamis yang luas (`mod_rewrite`, `mod_ssl`, `mod_proxy`). Namun, pada konkurensi masif (fenomena *C10K problem*), model berbasis thread/process membutuhkan memori yang lebih besar dan *context switching overhead*.
- **Nginx** mengadopsi arsitektur *asynchronous*, *event-driven*, dan *non-blocking*. Satu worker process dapat menangani ribuan koneksi secara simultan menggunakan multiplexing I/O kernel Linux (`epoll`). Nginx efisien dalam pemakaian CPU dan memori, unggul dalam menyajikan berkas statis, memiliki modul reverse proxy dan load balancer bawaan, serta populer sebagai TLS termination point.

| Aspek Komparasi | Apache HTTP Server (httpd) | Nginx |
| --- | --- | --- |
| Model arsitektur | Process/thread-per-connection (MPM prefork/worker/event) | Asynchronous event-driven non-blocking loop (epoll) |
| Penggunaan resource | Meningkat secara linier seiring pertambahan koneksi | Sangat rendah dan stabil pada konkurensi tinggi |
| Direktori konfigurasi | `/usr/local/apache2/conf` | `/etc/nginx` |
| Document root bawaan | `/usr/local/apache2/htdocs` | `/usr/share/nginx/html` |
| Konfigurasi desentralisasi | Didukung via `.htaccess` (fleksibel namun menambah I/O disk) | Tidak didukung (seluruh konfigurasi harus terpusat) |
| Fitur reverse proxy | Memerlukan modul tambahan (`mod_proxy`, `mod_proxy_http`) | Kemampuan bawaan terintegrasi (`proxy_pass`) |
| Kasus penggunaan ideal | Aplikasi monolitik legacy, PHP `mod_php`, web hosting bersama | Reverse proxy, API gateway, TLS terminator, web statis |

**TLS, penyimpanan private key, dan TLS offloading.** Transport Layer Security (TLS) menjamin kerahasiaan data (enkripsi simetris AES-GCM), integritas data (hash SHA-256), dan autentikasi identitas server (sertifikat X.509). TLS Offloading (terminasi TLS) memusatkan proses dekripsi kriptografis pada reverse proxy Nginx sehingga beban komputasi backend diringankan dan sertifikat dikelola terpusat di satu titik masuk. Kunci privat (`lab.key`) tidak boleh dimasukkan ke dalam image atau repositori Git. Kunci harus disuntikkan secara dinamis saat runtime melalui read-only bind mount (`:ro`) dengan izin berkas ketat (`chmod 600`) untuk meminimalkan risiko pencurian.

![Peta konsep TLS, penyimpanan private key, dan TLS offloading](./assets/halaman-06-gambar-01.jpeg)

*Gambar 1. Peta konsep TLS, penyimpanan private key, dan TLS offloading.*

**Reverse proxy, routing, dan filter keamanan.** Reverse Proxy bertindak sebagai perantara yang mewakili server origin backend di hadapan klien eksternal. Nginx mengevaluasi URI request untuk melakukan Layer-7 Path Routing: rute root `/` diteruskan ke Apache web server, sedangkan rute `/api/` dialihkan ke backend Flask. Proxy meneruskan header klien terpercaya (`Host`, `X-Real-IP`, `X-Forwarded-For`, `X-Forwarded-Proto`) untuk menjaga konteks alamat asli klien. Sebagai filter keamanan, Nginx menyuntikkan header `X-Content-Type-Options: nosniff` (mencegah MIME sniffing), `X-Frame-Options: DENY` (mencegah clickjacking), `Referrer-Policy: no-referrer`, serta pengalihan otomatis HTTP 301 ke HTTPS port 8443.

![Peta konsep reverse proxy, routing, dan filter keamanan](./assets/halaman-07-gambar-01.jpeg)

*Gambar 2. Peta konsep reverse proxy, routing, dan filter keamanan.*

---

## 3. Alat dan Bahan

| No | Komponen | Keterangan |
| --- | --- | --- |
| 1 | Sistem operasi | macOS |
| 2 | Docker Desktop | Docker Engine dan Docker Compose v2 (`docker compose`) |
| 3 | Image `nginx:alpine` | Reverse proxy dan TLS termination (service `proxy`) |
| 4 | Image `httpd:2.4-alpine` | Web server statis Apache (service `apache-web`) |
| 5 | Image `python:3.12-slim` | Image dasar backend Flask (service `flask-app`) |
| 6 | Flask 3.1.2 dan Gunicorn 23.0.0 | Framework backend dan WSGI server |
| 7 | OpenSSL | Membuat sertifikat self-signed dan memeriksa handshake TLS |
| 8 | curl | Menguji web ingress dan API |
| 9 | Direktori kerja | `~/docker-lab/bab-4` |
| 10 | Berkas utama | `compose.yaml`, `nginx/conf/default.conf`, `apache/sites/index.html`, `app/Dockerfile`, `app/requirements.txt`, `app/app.py`, `certs/lab.crt`, `certs/lab.key` |

---

## 4. Langkah Praktikum

Struktur berkas yang dibangun pada direktori kerja Bab 4 adalah sebagai berikut:

```text
bab-4/
├── compose.yaml
├── nginx/conf/default.conf
├── apache/sites/index.html
├── app/Dockerfile
├── app/requirements.txt
├── app/app.py
├── certs/lab.crt
├── certs/lab.key
└── logs/nginx/
```

Berikut adalah tahapan implementasi praktikum secara sistematis sesuai dengan langkah pada cheatsheet Bab 4.

### 4.1 Menyiapkan Direktori Kerja dan Struktur Proyek

Langkah awal adalah membuat hierarki direktori kerja `bab-4` yang memisahkan konfigurasi Apache, konfigurasi Nginx, sertifikat TLS, berkas log, dan kode backend Flask.

```bash
mkdir -p ~/docker-lab/bab-4/{apache/sites,nginx/conf,certs,logs/nginx,app}
cd ~/docker-lab/bab-4
```

![Pembuatan struktur direktori kerja Bab 4](./assets/halaman-08-gambar-01.jpeg)

*Gambar 3. Pembuatan struktur direktori kerja Bab 4.*

### 4.2 Membuat Sertifikat TLS Laboratorium (Self-Signed)

Pasangan kunci privat RSA 2048-bit dan sertifikat X.509 self-signed berumur 365 hari dibuat dengan OpenSSL. Ekstensi Subject Alternative Name (SAN) ditambahkan untuk `localhost` dan `127.0.0.1`. Hak akses kunci privat dibatasi dengan `chmod 600`, sedangkan sertifikat publik menggunakan `chmod 644`.

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout certs/lab.key \
  -out certs/lab.crt \
  -subj "/CN=localhost/O=DevSecOps Docker Lab" \
  -addext "subjectAltName=DNS:localhost,IP:127.0.0.1"

chmod 600 certs/lab.key
chmod 644 certs/lab.crt
```

![Pembuatan sertifikat TLS dan pembatasan permission kunci privat](./assets/halaman-09-gambar-01.jpeg)

*Gambar 4. Pembuatan sertifikat TLS laboratorium dan pembatasan permission kunci privat.*

### 4.3 Membuat File compose.yaml

File `compose.yaml` mendefinisikan tiga service: `proxy` (Nginx), `apache-web` (Apache httpd), dan `flask-app` (Flask). Ketiganya terhubung ke bridge network `web-net`. Hanya `proxy` yang memiliki published port, dan dibatasi ke `127.0.0.1`.

```bash
nano compose.yaml
```

```yaml
services:
  proxy:
    image: nginx:alpine
    ports:
      - "127.0.0.1:8080:80"
      - "127.0.0.1:8443:443"
    volumes:
      - ./nginx/conf:/etc/nginx/conf.d:ro
      - ./certs:/etc/nginx/certs:ro
      - ./logs/nginx:/var/log/nginx
    networks:
      - web-net
    depends_on:
      apache-web:
        condition: service_started
      flask-app:
        condition: service_healthy

  apache-web:
    image: httpd:2.4-alpine
    volumes:
      - ./apache/sites:/usr/local/apache2/htdocs:ro
    networks:
      - web-net

  flask-app:
    build:
      context: ./app
    networks:
      - web-net
    healthcheck:
      test:
        - CMD
        - python
        - -c
        - "import urllib.request; urllib.request.urlopen('http://127.0.0.1:5000/health')"
      interval: 5s
      timeout: 3s
      retries: 5
      start_period: 5s

networks:
  web-net:
    driver: bridge
```

![Isi file compose.yaml](./assets/halaman-10-gambar-01.jpeg)

*Gambar 5. Isi file `compose.yaml`.*

### 4.4 Membuat Konfigurasi Reverse Proxy Nginx (default.conf)

Konfigurasi Nginx mengalihkan HTTP port 80 ke HTTPS port 8443, mengaktifkan TLS 1.2 dan 1.3, menambahkan security header, serta meneruskan request `/` ke Apache dan `/api/` ke Flask.

```bash
nano nginx/conf/default.conf
```

```nginx
upstream apache_backend {
    server apache-web:80;
}

upstream flask_backend {
    server flask-app:5000;
}

server {
    listen 80;
    server_name localhost;

    return 301 https://$host:8443$request_uri;
}

server {
    listen 443 ssl;
    server_name localhost;

    ssl_certificate     /etc/nginx/certs/lab.crt;
    ssl_certificate_key /etc/nginx/certs/lab.key;
    ssl_protocols       TLSv1.2 TLSv1.3;

    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "DENY" always;
    add_header Referrer-Policy "no-referrer" always;

    location / {
        proxy_pass http://apache_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /api/ {
        proxy_pass http://flask_backend/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

![Isi file nginx/conf/default.conf](./assets/halaman-11-gambar-01.jpeg)

*Gambar 6. Isi file `nginx/conf/default.conf`.*

### 4.5 Membuat Halaman Web Statis Apache (index.html)

File `apache/sites/index.html` di-mount sebagai document root Apache (`/usr/local/apache2/htdocs`).

```bash
nano apache/sites/index.html
```

```html
<!DOCTYPE html>
<html lang="id">
<head>
  <meta charset="UTF-8">
  <title>DevSecOps Bab 4</title>
</head>
<body>
  <h1>Apache di Belakang Nginx</h1>
  <p>Halaman Bab 4 berhasil dilayani melalui reverse proxy.</p>
  <p><a href="/api/">Uji Flask API</a></p>
</body>
</html>
```

![Isi file apache/sites/index.html](./assets/halaman-12-gambar-01.jpeg)

*Gambar 7. Isi file `apache/sites/index.html`.*

### 4.6 Membuat Requirements Flask

File `app/requirements.txt` berisi library yang dibutuhkan backend: Flask 3.1.2 dan Gunicorn 23.0.0 sebagai WSGI server.

```bash
nano app/requirements.txt
```

```text
Flask==3.1.2
gunicorn==23.0.0
```

![Isi file app/requirements.txt](./assets/halaman-12-gambar-02.jpeg)

*Gambar 8. Isi file `app/requirements.txt`.*

### 4.7 Membuat Source Code Backend Flask (app.py)

File `app/app.py` menyediakan dua route. Endpoint `/` mengembalikan JSON berisi status, nama service, pesan, dan nilai header `X-Forwarded-Proto`. Endpoint `/health` mengembalikan HTTP 200 dengan `{"status": "healthy"}` untuk keperluan healthcheck.

```bash
nano app/app.py
```

```python
from flask import Flask, jsonify, request

app = Flask(__name__)


@app.get("/")
def index():
    return jsonify(
        status="ok",
        service="flask-app",
        message="API Bab 4 berhasil diakses melalui Nginx",
        forwarded_proto=request.headers.get("X-Forwarded-Proto"),
    )


@app.get("/health")
def health():
    return jsonify(status="healthy"), 200
```

![Isi file app/app.py](./assets/halaman-13-gambar-01.jpeg)

*Gambar 9. Isi file `app/app.py`.*

### 4.8 Membuat Dockerfile

`app/Dockerfile` membangun image backend dari `python:3.12-slim`. Container dijalankan sebagai user sistem non-root `appuser` (UID 10001), bukan root.

```bash
nano app/Dockerfile
```

```dockerfile
FROM python:3.12-slim
WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

RUN useradd --system --uid 10001 --no-create-home appuser
USER appuser

EXPOSE 5000
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "2", "app:app"]
```

![Isi file app/Dockerfile](./assets/halaman-13-gambar-02.jpeg)

*Gambar 10. Isi file `app/Dockerfile`.*

### 4.9 Validasi Konfigurasi Docker Compose

Sebelum deployment, struktur berkas diperiksa dan sintaks Compose divalidasi.

```bash
cd ~/docker-lab/bab-4
find . -maxdepth 3 -type f | sort
docker compose config
```

![Pemeriksaan struktur berkas dan validasi docker compose config](./assets/halaman-14-gambar-01.jpeg)

*Gambar 11. Pemeriksaan struktur berkas dan validasi `docker compose config`.*

Service yang harus muncul pada `docker compose config --services` adalah `proxy`, `apache-web`, dan `flask-app`.

```bash
docker compose config --services
```

![Daftar service pada konfigurasi Compose](./assets/halaman-14-gambar-02.jpeg)

*Gambar 12. Daftar service pada konfigurasi Compose.*

### 4.10 Membangun Image dan Menjalankan Stack Compose

Image Flask dibangun dan seluruh container dijalankan di background. Status dan log diperiksa setelahnya.

```bash
docker compose up -d --build
docker compose ps
docker compose logs --tail 100
```

![Proses build image dan startup stack Compose](./assets/halaman-15-gambar-01.jpeg)

*Gambar 13. Proses build image dan startup stack Compose.*

![Status container dan log awal seluruh service](./assets/halaman-15-gambar-02.jpeg)

*Gambar 14. Status container (`docker compose ps`) dan log awal seluruh service.*

### 4.11 Pengujian Fungsional Web Ingress dan API

**1. Redirect HTTP ke HTTPS (port 8080)**

```bash
curl -I http://localhost:8080/
```

![Hasil curl -I ke port 8080](./assets/halaman-15-gambar-03.png)

*Gambar 15. Hasil `curl -I` ke port 8080.*

**2. Halaman Apache melalui Nginx (port 8443)**

```bash
curl -k -i https://localhost:8443/
```

![Hasil curl HTTPS ke endpoint /](./assets/halaman-16-gambar-01.png)

*Gambar 16. Hasil `curl` HTTPS ke endpoint `/`.*

**3. Flask API melalui Nginx**

```bash
curl -k https://localhost:8443/api/
```

![Hasil curl ke endpoint /api/](./assets/halaman-16-gambar-02.jpeg)

*Gambar 17. Hasil `curl` ke endpoint `/api/`.*

**4. Health endpoint Flask**

```bash
curl -k -i https://localhost:8443/api/health
```

![Hasil curl ke endpoint /api/health](./assets/halaman-16-gambar-03.jpeg)

*Gambar 18. Hasil `curl` ke endpoint `/api/health`.*

**5. Informasi TLS**

```bash
openssl s_client \
  -connect localhost:8443 \
  -servername localhost \
  -brief </dev/null
```

![Informasi handshake TLS pada endpoint HTTPS](./assets/halaman-17-gambar-01.png)

*Gambar 19. Informasi handshake TLS pada endpoint HTTPS.*

Opsi `curl -k` hanya digunakan karena sertifikat bersifat self-signed untuk laboratorium.

### 4.12 Pemeriksaan Boundary dan Isolasi Jaringan

Pemeriksaan ini memastikan hanya proxy yang memiliki published port, sementara Apache dan Flask hanya dapat dijangkau melalui network internal `web-net`.

```bash
docker compose ps
```

![Status container dan published port](./assets/halaman-17-gambar-02.jpeg)

*Gambar 20. Status container dan published port hasil `docker compose ps`.*

```bash
docker network inspect bab-4_web-net
```

![Hasil docker network inspect](./assets/halaman-17-gambar-03.jpeg)

*Gambar 21. Hasil `docker network inspect bab-4_web-net`.*

```bash
docker compose exec proxy wget -qO- http://apache-web/
```

![Akses dari container proxy ke apache-web](./assets/halaman-18-gambar-01.jpeg)

*Gambar 22. Akses dari container `proxy` ke `apache-web` melalui nama service.*

```bash
docker compose exec proxy wget -qO- http://flask-app:5000/health
```

![Akses dari container proxy ke flask-app](./assets/halaman-18-gambar-02.jpeg)

*Gambar 23. Akses dari container `proxy` ke `flask-app` melalui nama service.*

### 4.13 Pemeriksaan dan Audit Log

Log diperiksa melalui `docker compose logs` dan dengan membaca langsung berkas log pada bind mount `logs/nginx/`.

```bash
docker compose logs proxy
docker compose logs apache-web
docker compose logs flask-app
```

![Log service proxy, apache-web, dan flask-app](./assets/halaman-18-gambar-03.jpeg)

*Gambar 24. Log service `proxy`, `apache-web`, dan `flask-app`.*

```bash
tail -n 20 logs/nginx/access.log
tail -n 20 logs/nginx/error.log
```

![Isi access.log dan error.log Nginx pada direktori host](./assets/halaman-19-gambar-01.jpeg)

*Gambar 25. Isi `access.log` dan `error.log` Nginx pada direktori host.*

### 4.14 Membersihkan Lingkungan (Cleanup)

Setelah pengujian selesai, stack dihentikan. Image lokal hasil build dapat dihapus bila diperlukan.

```bash
cd ~/docker-lab/bab-4
docker compose down
docker compose down --rmi local
```

![Hasil docker compose down](./assets/halaman-19-gambar-02.png)

*Gambar 26. Hasil `docker compose down`.*

---

## 5. Hasil Pengujian

| Pemeriksaan | Hasil Aktual | Bukti | Status |
| --- | --- | --- | --- |
| `docker compose config` | Konfigurasi divalidasi tanpa galat | Gambar 11 | Terpenuhi |
| `docker compose ps` | Service `flask-app` berstatus *healthy* | Gambar 14 | Terpenuhi |
| `curl -I http://localhost:8080/` | HTTP port 8080 menghasilkan redirect 301 | Gambar 15 | Terpenuhi |
| `curl -k -i https://localhost:8443/` | HTTPS port 8443 menampilkan halaman Apache | Gambar 16 | Terpenuhi |
| `curl -k https://localhost:8443/api/` dan `/api/health` | `/api/` menghasilkan JSON Flask | Gambar 17, 18 | Terpenuhi |
| `openssl s_client` | TLS 1.2 atau TLS 1.3 berhasil dinegosiasikan | Gambar 19 | Terpenuhi |
| `docker compose ps` | Apache dan Flask tidak memiliki published port | Gambar 20 | Terpenuhi |
| `docker compose logs` dan `access.log` | Request tercatat pada access log Nginx | Gambar 24, 25 | Terpenuhi |

---

## 6. Evaluasi dan Latihan Mandiri

### 1. Mengapa reverse proxy tidak seharusnya menjalankan semua logic aplikasi?

Reverse proxy dirancang untuk tugas di tepi jaringan: terminasi TLS, routing, buffering, dan menyajikan konten statis. Jika logika bisnis ikut dijalankan di sana, proses yang berat atau memblokir akan menahan worker proxy dan menurunkan kinerja untuk semua koneksi. Pemisahan juga memungkinkan proxy dan backend di-scale secara terpisah, serta memperkecil dampak bila ada celah di kode aplikasi.

### 2. Apa perbedaan TLS termination dan end-to-end TLS?

Pada TLS termination, enkripsi hanya berlaku antara klien dan proxy. Proxy mendekripsi data lalu meneruskannya ke backend lewat HTTP di jaringan privat, sehingga sertifikat dikelola di satu titik dan beban CPU backend lebih ringan. Pada end-to-end TLS, koneksi tetap terenkripsi sampai ke backend (proxy melakukan re-encryption atau pass-through). Ini lebih aman untuk jaringan internal yang tidak dipercaya, tetapi pengelolaan sertifikat dan bebannya bertambah.

### 3. Bagaimana cara mengisolasi backend agar tidak langsung diakses dari host?

Hapus direktif `ports:` pada `apache-web` dan `flask-app`, lalu tempatkan keduanya di bridge network privat (`web-net`) sehingga hanya dapat dijangkau lewat nama service di dalam network itu. Publikasikan port hanya pada `proxy`, dan batasi ke loopback (`127.0.0.1:8080` dan `127.0.0.1:8443`) agar tidak terbuka di interface host yang lebih luas.

### 4. Apa konsekuensi menyimpan private key TLS di bind mount?

Keamanan kunci bergantung pada permission berkas di host: jika tidak `chmod 600`, pengguna lain di host dapat membacanya. Tanpa opsi `:ro`, container yang disusupi dapat mengubah atau menimpa berkas kunci. Folder `certs/` juga berisiko ikut ter-commit ke repository bila tidak masuk `.gitignore`. Selain itu, rotasi sertifikat dilakukan manual pada berkas host, tidak lewat secret manager.

### 5. Bandingkan log Nginx dan log Apache dari sisi format dan kegunaan debugging.

Keduanya memakai format akses yang mirip (IP klien, waktu, method dan URI, status, ukuran respons, referer, user-agent). Nginx lebih unggul untuk peran proxy karena `log_format` dapat memuat variabel seperti `$upstream_addr`, `$upstream_status`, dan `$request_time`, sehingga mudah membedakan lambatnya proxy dari lambatnya backend saat terjadi 502/504. Log error Apache lebih deskriptif, dengan kode modul seperti `AH00558`, sehingga membantu menelusuri masalah internal Apache, sedangkan log error Nginx lebih ringkas dan berfokus pada koneksi, DNS upstream, dan timeout.

---

## 7. Threat Statement

Aset yang dilindungi meliputi private key TLS (`lab.key`), source code backend Flask, container, dan data lalu lintas antara klien dan service. Risiko yang diperhatikan meliputi pencurian kunci privat, penyadapan trafik, pemalsuan header, kebocoran informasi lewat log, serta serangan HTTP flood. Mitigasi yang diterapkan meliputi pembatasan published port hanya pada `proxy` dan hanya ke `127.0.0.1`, isolasi backend pada private bridge network `web-net`, `chmod 600` dan mount `:ro` pada kunci privat, security header pada Nginx, penggunaan user non-root `appuser` pada container Flask, serta healthcheck sebagai syarat startup proxy. Risiko yang masih tersisa pada lingkungan laboratorium ini dirangkum pada tabel berikut.

| Risiko | Penjelasan |
| --- | --- |
| Sertifikat self-signed | Tidak memiliki rantai kepercayaan CA. Di produksi, pengguna terbiasa mengabaikan peringatan browser sehingga rawan MITM. |
| Trafik internal tanpa enkripsi | Nginx ke Apache dan Flask memakai HTTP biasa. Jika host atau container lain terkompromi, trafik dapat disadap. |
| Pemalsuan header | Backend mempercayai `X-Forwarded-Proto` dan `X-Real-IP`. Bila backend dapat dijangkau langsung, header dapat dipalsukan. |
| Kebocoran lewat log | URI dicatat di `access.log`. Token atau kunci yang dikirim lewat query string tersimpan sebagai teks polos. |
| Tanpa rate limit dan batas resource | Proxy rentan terhadap HTTP flood dan resource exhaustion. |

---

## 8. Analisis

Praktikum ini menjalankan tiga service, yaitu Nginx (`proxy`), Apache httpd (`apache-web`), dan Flask (`flask-app`) dalam satu bridge network privat `web-net`. Nginx menjadi satu-satunya pintu masuk: melakukan terminasi TLS, mengalihkan HTTP ke HTTPS, menambahkan security header, lalu meneruskan `/` ke Apache dan `/api/` ke Flask. Bind mount pada `logs/nginx/` menyediakan jejak audit yang persisten di host.

**Masalah yang muncul dan cara diagnosis**

**a. Pesan *read-only file system* pada log Nginx.** Pada log startup proxy muncul informasi bahwa `10-listen-on-ipv6-by-default.sh` tidak dapat memodifikasi `/etc/nginx/conf.d/default.conf`. Diagnosis dilakukan lewat `docker compose logs proxy`. Penyebabnya adalah folder konfigurasi di-mount dengan opsi `:ro`, sehingga skrip bawaan image tidak bisa menulis ke berkas tersebut. Pesan ini bersifat informasi dan tidak mengganggu layanan, karena konfigurasi sudah mendengarkan IPv4 secara eksplisit (`listen 80;` dan `listen 443 ssl;`).

**b. `curl: (60) SSL certificate problem` pada HTTPS.** Pemanggilan `curl https://localhost:8443/` tanpa opsi tambahan ditolak karena sertifikat self-signed tidak ada di daftar CA tepercaya klien. Untuk laboratorium, pengujian memakai `curl -k`, sedangkan parameter TLS tetap diverifikasi terpisah melalui `openssl s_client`.

---

## 9. Tindak Lanjut

1. Menggunakan sertifikat dari CA tepercaya dengan pembaruan otomatis (ACME/Let's Encrypt).
2. Mengenkripsi jalur proxy ke backend (TLS internal atau mTLS).
3. Menyimpan private key di secret manager (Docker/Kubernetes Secrets atau Vault), bukan di bind mount.
4. Menambahkan `limit_req_zone`/`limit_conn_zone` di Nginx, dan mempertimbangkan WAF dengan OWASP Core Rule Set.
5. Membatasi resource container (`deploy.resources.limits`), menggunakan `read_only: true`, dan `cap_drop: ["ALL"]` dengan `cap_add: ["NET_BIND_SERVICE"]` bila diperlukan.
6. Melakukan sanitasi parameter sensitif di log dan mengirim log ke sistem terpusat (SIEM).

---

## 10. Kesimpulan

Praktikum Bab 4 mengimplementasikan stack web tiga service (Nginx, Apache httpd, dan Flask) dengan Docker Compose. Kesimpulan yang dapat diambil:

1. Nginx sebagai reverse proxy memisahkan antarmuka publik dari backend dan mengatur routing berbasis path (`/` ke Apache, `/api/` ke Flask).
2. Terminasi TLS di Nginx dengan TLS 1.2/1.3 dan pengalihan HTTP 301 ke HTTPS memusatkan pengelolaan sertifikat di satu titik masuk.
3. Prinsip *least exposure* diterapkan dengan private bridge network `web-net`: hanya proxy yang memiliki published port, dibatasi ke `127.0.0.1`.
4. Pengerasan dasar diterapkan melalui security header, `chmod 600` pada kunci privat, mount `:ro`, backend berjalan sebagai user non-root, dan healthcheck sebagai syarat startup proxy.
5. Bind mount pada `logs/nginx/` menyediakan jejak audit yang persisten di host untuk pemantauan dan investigasi.

---

## 11. Referensi

Ferry Astika Saputra, *Bab 4 — Web Service Container: Apache, Nginx, Reverse Proxy, dan TLS*, Repository DevSecOps PENS, diakses pada 3 Oktober 2026.
