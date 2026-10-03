# Bab 3: Docker Network, Volume, Bind Mount, tmpfs, dan Compose

Disusun untuk memenuhi Mata Kuliah Workshop DevOps

Dosen Pengampu : Dr. Ferry Astika Saputra, S.T., M.Sc

![Logo PENS](assets/logo-pens.png)

**Oleh :**

| | |
|---|---|
| Nama | : Anifa Aulia Abdari |
| NRP | : 3126640053 |
| Kelas | : B - StrLJ D4 Teknik Informatika |

**Politeknik Elektronika Negeri Surabaya**
**Departemen Teknik Informatika dan Komputer**
**Program Studi Teknik Informatika**

---

# LAPORAN PRAKTIKUM BAB 3

## Docker Network, Volume, Bind Mount, tmpfs, dan Compose

| | |
|---|---|
| Nama | : Anifa Aulia Abdari |
| NRP | : 3123512918 |
| Kelas | : Kelas B - D4 StrLJ Teknik Informatika |
| Tanggal pelaksanaan | : 23 September 2026 |

---

## 1. Tujuan Praktikum

Praktikum Bab 3 bertujuan membangun pemahaman praktis atas tiga mekanisme inti Docker yang saling melengkapi: jaringan (network), penyimpanan (volume, bind mount, tmpfs), dan orkestrasi multi-container melalui Compose. Secara rinci, praktikum ini bertujuan untuk:

1. Membuat user-defined bridge network dan membuktikan mekanisme name resolution antar-container berdasarkan nama, bukan alamat IP.
2. Menguji perilaku named volume, khususnya lifecycle-nya yang independen dari container, serta melakukan backup datanya ke berkas arsip.
3. Menyusun dan menjalankan arsitektur aplikasi tiga lapis (Nginx sebagai reverse proxy, Flask sebagai backend, PostgreSQL sebagai database) menggunakan Docker Compose.
4. Menerapkan pemisahan network frontend/backend serta health check pada `depends_on` agar urutan startup antar-service terkendali.
5. Menerapkan praktik keamanan dasar pada konfigurasi Compose: membatasi publikasi port ke loopback (`127.0.0.1`) dan menjalankan container aplikasi sebagai pengguna non-root.

## 2. Dasar Teori Singkat

Jaringan container pada Docker tidak sekadar memberi alamat IP, melainkan membentuk graf keterjangkauan yang menentukan komponen mana yang dapat saling berkomunikasi. Default bridge network menyediakan konektivitas dasar tetapi tidak memiliki resolusi nama otomatis, sehingga container harus saling mengetahui alamat IP satu sama lain, sesuatu yang tidak stabil karena alamat IP dapat berubah setiap kali container dibuat ulang. User-defined bridge network menutup celah ini dengan menyediakan DNS internal berbasis nama atau alias container, sehingga identitas logis service menjadi lebih stabil dibandingkan dengan alamat IP.

Untuk persistensi data, Docker menyediakan tiga mekanisme dengan karakteristik berbeda. Named volume dikelola oleh Docker Engine dan direkomendasikan untuk data yang harus bertahan melampaui lifecycle container, seperti data database. Bind mount memetakan path host secara langsung ke dalam container, cocok untuk source code dan konfigurasi pengembangan, tetapi membuat container memiliki akses tulis ke filesystem host secara default, sehingga perlu dibatasi dengan opsi read-only apabila tidak diperlukan. Tmpfs menyimpan data di memori host dan otomatis hilang saat container berhenti, sesuai untuk cache atau data sensitif yang tidak boleh persisten.

Docker Compose menyatakan model aplikasi multi-container secara deklaratif dalam satu berkas YAML. Compose memungkinkan definisi service, network, volume, environment, dan healthcheck dalam satu unit yang dapat direproduksi. Elemen `depends_on` dengan `condition: service_healthy` memastikan sebuah service tidak dijalankan sebelum dependency-nya benar-benar siap menerima koneksi, berbeda dengan `depends_on` biasa yang hanya mengatur urutan pembuatan container tanpa memverifikasi kesiapan layanan di dalamnya.

## 3. Alat dan Lingkungan

| Komponen | Keterangan |
|---|---|
| Sistem operasi | macOS |
| Docker Desktop | Docker version 29.7.2 |
| Docker Compose | v2 (plugin bawaan Docker Desktop) |
| Image `nginx:alpine` | Web server pada network lab-net dan service web |
| Image `alpine:3.20` | Container uji untuk penulisan dan backup volume |
| Image `postgres:16-alpine` | Database pada service db |
| Image `python:3.12-slim` | Image dasar untuk service app (Flask + gunicorn) |
| Direktori kerja | `~/docker-lab/bab-3` |
| Berkas utama | `compose.yaml`, `nginx.conf`, `app/Dockerfile`, `app/requirements.txt`, `app/app.py`, `html/index.html` |

## 4. Langkah Praktikum

### 4.1 Menyiapkan Struktur Direktori dan Berkas Pendukung Compose

Struktur direktori praktikum:

![Membuat struktur direktori bab-3](assets/4-1-struktur-direktori.png)

Direktori `bab-3` digunakan sebagai direktori utama praktikum. Folder `app` berisi Dockerfile dan source code Flask, sedangkan folder `html` digunakan untuk menyimpan halaman HTML yang akan digunakan oleh Nginx.

### 4.2 Membuat file compose.yaml

File `compose.yaml` dibuat sebagai konfigurasi utama untuk menjalankan seluruh service menggunakan Docker Compose.

![Isi file compose.yaml](assets/4-2-compose-yaml.png)

Docker Compose digunakan untuk mengatur beberapa container sekaligus. Pada praktikum ini, ketiga service tersebut saling terhubung melalui network Docker. Service `web` terhubung ke network `frontend`, service `app` terhubung ke `frontend` dan `backend`, sedangkan service `db` hanya terhubung ke `backend`.

### 4.3 Membuat Requirements Flask

File `requirements.txt` digunakan untuk menentukan library Python yang diperlukan oleh aplikasi Flask.

![Isi file requirements.txt](assets/4-3-requirements-txt.png)

Flask digunakan sebagai framework aplikasi web, psycopg digunakan agar aplikasi Python dapat berkomunikasi dengan PostgreSQL, sedangkan Gunicorn digunakan sebagai application server untuk menjalankan aplikasi Flask di dalam container.

### 4.4 Membuat Source Code Flask

Source code aplikasi dibuat pada file `app/app.py`.

![Isi file app/app.py](assets/4-4-app-py.png)

Pada endpoint `/`, aplikasi melakukan koneksi ke database PostgreSQL menggunakan konfigurasi yang diperoleh dari environment variable. Jika koneksi berhasil, aplikasi menampilkan status ok dan informasi versi PostgreSQL. Endpoint `/health` digunakan sebagai health check aplikasi untuk memastikan database dapat diakses dengan baik.

### 4.5 Membuat Dockerfile

Dockerfile digunakan untuk menentukan bagaimana image aplikasi Flask dibangun.

![Isi file app/Dockerfile](assets/4-5-dockerfile.png)

Konfigurasi Dockerfile menggunakan Python 3.12 sebagai base image dan menginstal dependency dari `requirements.txt`.

### 4.6 Membuat Konfigurasi Nginx

File `nginx.conf` digunakan untuk mengatur Nginx sebagai reverse proxy menuju aplikasi Flask.

![Isi file nginx.conf](assets/4-6-nginx-conf.png)

Nginx meneruskan request yang diterima pada port 80 ke service `app` pada port 5000. Nama `app` merupakan nama service yang didefinisikan pada Docker Compose. Docker menyediakan DNS internal sehingga Nginx dapat menemukan container Flask menggunakan nama service tanpa harus mengetahui alamat IP container secara manual. Selain sebagai reverse proxy, Nginx juga digunakan untuk menyediakan halaman statis `static.html`.

### 4.7 Membuat Halaman HTML

Halaman HTML dibuat pada `nano html/index.html`. File ini berisi halaman sederhana untuk menguji penggunaan bind mount pada Nginx.

![Isi file html/index.html](assets/4-7-index-html.png)

Folder `html` pada komputer di-mount ke direktori `/usr/share/nginx/html` di dalam container Nginx. Dengan demikian, file HTML pada host dapat digunakan langsung oleh container.

### 4.8 Validasi Konfigurasi Docker Compose

Sebelum menjalankan container, konfigurasi Docker Compose dicek dengan command `docker compose config`. Perintah tersebut digunakan untuk memastikan file YAML dapat diproses oleh Docker Compose dan tidak terdapat kesalahan sintaks atau indentasi.

![Output docker compose config (bagian 1)](assets/4-8-compose-config-1.png)

![Output docker compose config (bagian 2)](assets/4-8-compose-config-2.png)

Selanjutnya dilakukan pengecekan service dengan command `docker compose config --services`

![Output docker compose config --services](assets/4-8-compose-config-services.png)

### 4.9 Menjalankan Docker Compose

Setelah konfigurasi dinyatakan valid, seluruh service dijalankan menggunakan: `docker compose up -d --build`. Parameter `-d` digunakan agar container berjalan di background, sedangkan `--build` digunakan untuk membangun kembali image aplikasi berdasarkan Dockerfile.

![Output docker compose up -d --build](assets/4-9-compose-up-build.png)

### 4.10 Memeriksa Status Container

![Output docker compose ps](assets/4-10-compose-ps.png)

Status healthy pada PostgreSQL menunjukkan bahwa health check database berhasil dilakukan.

### 4.11 Pengujian Aplikasi

Pengujian dilakukan menggunakan command : `curl http://localhost:8080/`

![Output curl http://localhost:8080/](assets/4-11-curl-root.png)

Request dikirim ke Nginx melalui port 8080. Nginx kemudian meneruskan request tersebut ke aplikasi Flask. Flask selanjutnya melakukan koneksi ke PostgreSQL. Keberhasilan response menunjukkan bahwa komunikasi antarservice berjalan dengan baik.

Uji health endpoint: `curl -i http://localhost:8080/health`. Endpoint `/health` digunakan untuk memastikan aplikasi Flask dapat terhubung dengan database PostgreSQL.

![Output curl -i http://localhost:8080/health](assets/4-11-curl-health.png)

Uji Halaman Statis : `curl http://localhost:8080/static.html`

![Output curl http://localhost:8080/static.html](assets/4-11-curl-static.png)

### 4.12 Pemeriksaan Log

Log seluruh service diperiksa menggunakan : `docker compose logs --tail 100`. Perintah tersebut digunakan untuk melihat aktivitas dan kemungkinan error dari seluruh container. Log juga dapat digunakan untuk membantu mengetahui penyebab apabila aplikasi tidak dapat diakses.

![Output docker compose logs --tail 100](assets/4-12-logs-semua-service.png)

Pemeriksaan Log berdasarkan services

`docker compose logs app`

![Output docker compose logs app](assets/4-12-logs-app.png)

`docker compose logs db`

![Output docker compose logs db](assets/4-12-logs-db.png)

`docker compose log web`

![Output docker compose logs web](assets/4-12-logs-web.png)

### 4.12 Menghentikan Docker Compose

Menghentikan Docker menggunakan command : `docker compose down`. Perintah tersebut menghentikan dan menghapus container serta network yang dibuat oleh Docker Compose. Volume PostgreSQL tetap dipertahankan sehingga data database tidak langsung terhapus.

![Output docker compose down](assets/4-12-compose-down.png)

## 5. Hasil Pengujian

| Pemeriksaan | Hasil aktual | Status |
|---|---|---|
| `docker compose config` | Konfigurasi berhasil divalidasi tanpa error | Terpenuhi |
| `docker compose up -d --build` | Service web, app, dan db berhasil dijalankan | Terpenuhi |
| `docker compose ps` | Seluruh service berjalan dan db berstatus healthy | Terpenuhi |
| `docker network ls` | Network frontend dan backend berhasil dibuat | Terpenuhi |
| `docker ps` | Container berhasil berjalan | Terpenuhi |
| `curl http://localhost:8080/` | Flask berhasil terhubung dengan PostgreSQL | Terpenuhi |
| `curl -i http://localhost:8080/health` | Status aplikasi healthy | Terpenuhi |
| `curl http://localhost:8080/static.html` | Halaman statis berhasil diakses melalui Nginx | Terpenuhi |
| `ping` | Koneksi antarservice berhasil diuji | Terpenuhi |

## 6. Threat Statement

Aset yang dilindungi meliputi source code Flask, kredensial PostgreSQL, data database, dan container. Risiko yang diperhatikan adalah akses tidak sah, kebocoran kredensial, dan eksploitasi container. Mitigasi yang diterapkan adalah membatasi port Nginx ke `127.0.0.1`, menjalankan aplikasi menggunakan user non-root `appuser`, serta memisahkan network frontend dan backend. Kredensial database masih berada di `compose.yaml` sehingga konfigurasi ini ditujukan untuk lingkungan praktikum.

## 7. Analisis

Docker Compose berhasil menjalankan tiga service, yaitu Nginx, Flask, dan PostgreSQL. Network frontend digunakan untuk komunikasi Nginx dengan Flask, sedangkan network backend digunakan untuk komunikasi Flask dengan PostgreSQL. Health check PostgreSQL juga berhasil diterapkan sehingga aplikasi Flask dapat terhubung setelah database dalam kondisi sehat. Pengujian endpoint `/`, `/health`, dan `/static.html` berhasil dilakukan.

## 8. Tindak Lanjut

1. Menggunakan Docker Secret untuk menyimpan kredensial database.
2. Melakukan backup dan restore database secara berkala.
3. Menambahkan resource limit dan logging.
4. Menggunakan HTTPS apabila aplikasi diakses dari luar host.

## 9. Kesimpulan

Praktikum Bab 3 berhasil mengimplementasikan Docker Compose dengan tiga service, yaitu Nginx, Flask, dan PostgreSQL. Seluruh service berhasil berjalan dan saling terhubung. Pengujian endpoint `/`, `/health`, dan `/static.html` berhasil dilakukan. Penerapan keamanan berupa pembatasan port ke `127.0.0.1`, penggunaan user non-root, serta pemisahan network frontend dan backend juga berhasil diterapkan.

## 10. Referensi

Ferry Astika Saputra, *Bab 3 — Docker Network, Volume, Bind Mount, tmpfs, dan Compose*, Repository DevSecOps PENS, diakses pada 23 September 2026.
