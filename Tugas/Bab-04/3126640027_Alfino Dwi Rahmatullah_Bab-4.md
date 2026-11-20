**LAPORAN WORKSHOP 4**

**Workshop DevOps**

Apache httpd, Nginx, virtual host, reverse proxy, TLS.

<img src="./media/image1.png" style="width:4.11458in;height:3.90625in" alt="https://lh3.googleusercontent.com/50AxiJHeGJ4AVk6byyh4mtGncGon7VrciFgyBxwv9d_Uk-gUnqhWku54uhgyZyGgLUelSOH7MXF3psL8Ctlp8FUSHVu6XNpqCpy5F2jYP2n_diol-p486IyGik4d_jWuuWFQnOqz" />

> Nama : Alfino Dwi Rahmatullah – 3126640027
>
> Kelas : RPL LJ Teknik Informatika B
>
> Dosen : Dr. Ferry Astika Saputra, ST, M.Sc.

Politeknik Elektronika Negeri Surabaya

Departemen Teknik Informatika dan Komputer

Program Studi Teknik Informatika

2026

1.  Pendahuluan

    Penugasan praktikum ini berfokus pada dua modul, yaitu Bab 4 sebagai landasan teori dan Bab 4 Cheatsheet (CS) sebagai acuan eksekusi praktikum. Modul Bab 4 dijadikan rujukan pemahaman konsep web service sebagai aplikasi, arsitektur peran antara reverse proxy (Nginx) dan origin server (Apache), mekanisme TLS beserta siklus sertifikat, transmisi trusted headers, serta tata kelola logging dan audit keamanan sistem. Sementara itu, seluruh tahapan teknis langkah demi langkah mulai dari penyusunan struktur direktori terisolasi, penerbitan sertifikat digital TLS self-signed dengan ekstensi SAN menggunakan OpenSSL, penulisan orkestrasi compose.yaml, konfigurasi routing proxy beserta redirect HTTP ke HTTPS dan security headers, pembuatan origin server statis Apache, perakitan aplikasi backend Flask dengan Dockerfile non-root, hingga pengujian isolasi jaringan privat web-net dikerjakan secara penuh mengikuti instruksi cheatsheet (bab-04-cs.md).

2.  Peta Konsep Apache httpd, Nginx, virtual host, reverse proxy, TLS.

    Peta konsep disusun oleh kelompok 2, dengan anggota :

    Alfino Dwi Rahmatullah : 3126640027

    Adrian Sondang Irawan : 3126640029

    Muchamad Syikha Akmal : 3126640035

    Peta Konsep

    <img src="./media/image2.png" style="width:6.5in;height:2.72361in" />

    Penjelasan singkat

<!-- -->

1.  **Lapisan 1: Client External & Ingress Traffic**

    Berfungsi sebagai one gate dari jaringan luar. Port publikasi host dibatasi pada antarmuka *loopback* (127.0.0.1:8080 untuk HTTP dan 127.0.0.1:8443 untuk HTTPS). Hal ini mencegah eksposur port yang tidak perlu ke jaringan publik sekaligus menyiapkan mekanisme *HTTP-to-HTTPS redirection*.

2.  **Lapisan 2: Docker Host (Reverse Proxy & TLS Termination)**

    Kontainer proxy (Nginx) bertindak sebagai *security boundary* dan pengendali lalu linta. Di lapisan ini terjadi proses dekripsi (*TLS Termination / SSL Offloading*) sehingga beban komputasi dipusatkan di depan, sebelum aliran data *plain* HTTP didistribusikan ke backend berdasarkan evaluasi URI (*path routing*: / ke Apache dan /api/ ke Flask).

3.  **Lapisan 3: Internal Network (web-net) & Service Containers**

    Menerapkan isolasi perimeter jaringan backend. Kontainer apache-web (port 80) dan flask-app (port 5000) tidak memiliki *published port* ke host dan murni beroperasi di dalam jaringan privat web-net. Komunikasi antar-kontainer mengandalkan fitur *service discovery* dan DNS internal Docker tanpa mengekspos IP fisik atau membuka celah serangan langsung ke aplikasi backend.

4.  **Lapisan 4: Host Storage & Bind Mounts (Filesystem)**

    Menyeimbangkan aspek ketahanan data, pemantauan, dan pengerasan keamanan . Pemasangan atribut *read-only* (:ro) pada berkas konfigurasi Nginx, sertifikat TLS (lab.crt, lab.key), dan web root Apache melindungi integritas sistem dari modifikasi ilegal dari dalam kontainer. Sementara itu, pemasangan volume log secara dua arah memastikan data audit (access.log dan error.log) tetap tersimpan permanen di host untuk kebutuhan analisis.

<!-- -->

3.  Langkah Praktikum Eksploratif

    Struktur File :

    <img src="./media/image3.png" style="width:3.09375in;height:2.83915in" />

**1. Direktori Kerja**

<img src="./media/image4.png" style="width:6.26806in;height:0.57222in" />

> Pembuatan subdirektori memisahkan berkas konfigurasi proxy, sertifikat, kode aplikasi backend, dan log akses ke dalam ruang lingkup yang terisolasi. Struktur ini membatasi build context kontainer agar proses perakitan image aplikasi tidak mengekspos berkas sensitif dari direktori lain.

**2. Sertifikat TLS Laboratorium**

<img src="./media/image5.png" style="width:6.26806in;height:1.88611in" />

> Pembuatan sertifikat self-signed X.509 mengimplementasikan ekstensi modern SAN (Subject Alternative Name) agar koneksi lokal via domain localhost maupun IP 127.0.0.1 valid pada standar TLS kontemporer. hak akses chmod 600 pada private key menggunakan prinsip hak akses guna mencegah kebocoran key kepada pengguna lain di level host.

**3. File compose.yaml**

<img src="./media/image6.png" style="width:4.65694in;height:2.86712in" />

> File ini menerapkan prinsip least exposure dengan hanya memublikasikan port milik Nginx ke antarmuka loopback host (127.0.0.1), sementara service apache-web dan flask-app diisolasi tanpa publikasi port di jaringan web-net.

**4. File nginx/conf/default.conf**

<img src="./media/image7.png" style="width:5.00417in;height:2.92233in" />

> Konfigurasi Nginx memisahkan fungsi publik dengan pengalihan otomatis HTTP ke HTTPS (HTTP 301 Redirect) dan membatasi protokol ke TLSv1.2/TLSv1.3. Penambahan security headers mitigatif (nosniff, DENY) dan transmisi header identitas (X-Forwarded-Proto, X-Real-IP)

**5. File apache/sites/index.html**

<img src="./media/image8.png" style="width:5.26111in;height:2.11878in" />

Penyediaan web statis ini menguji kapabilitas Apache HTTPD sebagai server internal yang bertugas melayani root endpoint (\`/\`). Pemasangan dokumen melalui \*bind mount read-only\* memastikan konten tetap terlindungi dari potensi modifikasi ilegal di runtime kontainer.

**6. File app/requirements.txt**

<img src="./media/image9.png" style="width:5.51111in;height:0.54952in" />

**7. File app/app.py**

<img src="./media/image10.png" style="width:5.16389in;height:2.67406in" />

> Backend Python Flask dirancang untuk mengekstrak header X-Forwarded-Proto guna memvalidasi bahwa koneksi eksternal telah terenkripsi sebelum mencapai aplikasi. Penyediaan endpoint /health berfungsi sebagai pengujian yang secara berkala diuji oleh eksekutor kontainer untuk menjamin status ketersediaan layanan API.

**8. File app/Dockerfile**

<img src="./media/image11.png" style="width:6.26806in;height:2.78472in" />

> Dockerfile ini menerapkan hardening sistem dengan base image minimalis python:3.12-slim untuk memangkas attack surface bawaan OS. Penetapan hak eksekusi kepada akun non-root appuser mitigasi risiko hak akses sistem jika proses aplikasi web berhasil dieksploitasi oleh penyerang.

**9. Validasi**

<img src="./media/image12.png" style="width:5.28472in;height:1.30098in" />

<img src="./media/image13.png" style="width:5.32043in;height:3.28681in" />

<img src="./media/image14.png" style="width:5.21931in;height:0.74306in" />

> Tahap verifikasi direktori ini memastikan keberadaan seluruh artefak dependensi sebelum eksekusi dimulai. Pemeriksaan sintaks via docker compose config memvalidasi parsing YAML dan resolusi service (proxy, apache-web, flask-app), mencegah kegagalan runtime akibat kesalahan struktur berkas.**\**

**10. Build dan Menjalankan Stack**

docker compose up -d –build

<img src="./media/image15.png" style="width:5.32361in;height:1.53291in" />

docker compose ps

<img src="./media/image16.png" style="width:5.53889in;height:1.35741in" />

docker compose logs --tail 100

<img src="./media/image17.png" style="width:5.50694in;height:2.53755in" />

> Perintah up -d --build mengotomatisasi kompilasi image backend dan ketiga kontainer di latar belakang. Observasi status menunjukkan bahwa kontainer proxy dan apache-web berstatus Up, sementara flask-app mencapai status Up (healthy) setelah berhasil melewati tahapan verifikasi berkala pada rute kesehatannya.

**11. Pengujian Cepat**

> **\# HTTP harus redirect ke HTTPS**
>
> curl -I http://localhost:8080/
>
> <img src="./media/image18.png" style="width:5.43056in;height:1.64012in" />
>
> <img src="./media/image19.png" style="width:3.69358in;height:1.71528in" />
>
> **\# Halaman Apache melalui Nginx**
>
> curl -k -i https://localhost:8443/
>
> <img src="./media/image20.png" style="width:3.94444in;height:2.71208in" />
>
> **\# Flask API melalui Nginx**
>
> curl -k https://localhost:8443/api/
>
> <img src="./media/image21.png" style="width:5.77778in;height:0.39368in" />
>
> **\# Health endpoint Flask**
>
> curl -k -i https://localhost:8443/api/health
>
> <img src="./media/image22.png" style="width:5.71338in;height:1.66667in" />
>
> **\# Informasi TLS**
>
> openssl s_client \\
>
> -connect localhost:8443 \\
>
> -servername localhost \\
>
> -brief \</dev/null
>
> <img src="./media/image23.png" style="width:3.64583in;height:1.91596in" />
>
> Pengujian fungsional memvalidasi keberhasilan aturan pengalihan HTTP 301 ke HTTPS serta pemisahan rute proksi ke Apache (/) dan Flask (/api/). Negosiasi TLS yang diuji via openssl s_client membuktikan bahwa sertifikat digital berhasil digunakan dengan cipher suite aman pada protokol TLSv1.2/TLSv1.3.

**12. Pemeriksaan Isolasi**

> docker compose ps
>
> <img src="./media/image24.png" style="width:5.56944in;height:0.6047in" />
>
> docker network inspect bab-4_web-net
>
> <img src="./media/image25.png" style="width:3.93056in;height:2.52486in" />
>
> docker compose exec proxy wget -qO- <http://apache-web/>
>
> <img src="./media/image26.png" style="width:4.75694in;height:1.44036in" />
>
> docker compose exec proxy wget -qO- http://flask-app:5000/health
>
> <img src="./media/image27.png" style="width:4.85417in;height:0.31192in" />
>
> Pengujian inspeksi membuktikan arsitektur perimeter security, di mana service backend tidak mengekspos port ke sistem host dan hanya dapat diakses melalui antarmuka internal web-net oleh Nginx. Uji pemanggilan wget dari dalam kontainer proksi memvalidasi efektivitas resolusi DNS internal Docker antar-layanan.

**13. Pemeriksaan Log Akses**

> docker compose logs proxy
>
> <img src="./media/image28.png" style="width:6.26806in;height:1.09028in" />
>
> docker compose logs apache-web
>
> <img src="./media/image29.png" style="width:6.26806in;height:1.21042in" />
>
> docker compose logs flask-app
>
> <img src="./media/image30.png" style="width:6.26806in;height:1.04583in" />
>
> tail -n 20 logs/nginx/access.log
>
> <img src="./media/image31.png" style="width:6.26806in;height:1.14375in" />
>
> tail -n 20 logs/nginx/error.log
>
> <img src="./media/image32.png" style="width:6.26806in;height:2.19861in" />
>
> Pemasangan direktori ./logs/nginx via bind mount membuktikan rekaman audit trafik web langsung pada level host. Seluruh jejak permintaan HTTPS klien tercatat secara runtut di access.log dengan kode status 200/301, mendukung kebutuhan observabilitas, forensik insiden, dan akuntabilitas operasional.

4.  **Checklist PASS**

- docker compose config tidak menghasilkan galat.

  <img src="./media/image33.png" style="width:4.61111in;height:1.71833in" />

- flask-app berstatus healthy.

  <img src="./media/image34.png" style="width:5.68056in;height:0.41512in" />

- HTTP port 8080 menghasilkan redirect 301.

  <img src="./media/image35.png" style="width:5.61111in;height:1.60725in" />

- HTTPS port 8443 menampilkan halaman Apache.

  <img src="./media/image36.png" style="width:5.38591in;height:2.11806in" />

- /api/ menghasilkan JSON Flask.

  <img src="./media/image37.png" style="width:5.48611in;height:0.32471in" />

- TLS 1.2 atau TLS 1.3 berhasil dinegosiasikan.

  <img src="./media/image38.png" style="width:5.38542in;height:1.1864in" />

- Apache dan Flask tidak memiliki published port.

  <img src="./media/image39.png" style="width:5.48611in;height:0.70042in" />

- Request tercatat pada access log Nginx.

  <img src="./media/image40.png" style="width:5.48611in;height:0.82702in" />

5.  Verifikasi dan Skenario Pengujian

<table>
<colgroup>
<col style="width: 16%" />
<col style="width: 30%" />
<col style="width: 41%" />
<col style="width: 11%" />
</colgroup>
<thead>
<tr>
<th><strong>Skenario Pengujian</strong></th>
<th><strong>Perintah</strong></th>
<th><strong>Output Keberhasilan</strong></th>
<th><strong>Status</strong></th>
</tr>
</thead>
<tbody>
<tr>
<td>Nginx reverse proxy dapat mengakses Apache melalui nama service.</td>
<td>docker compose exec proxy wget -qO- http://apache-web/</td>
<td>Mengembalikan dokumen HTML Apache di Belakang Nginx secara langsung via DNS internal.</td>
<td>Berhasil</td>
</tr>
<tr>
<td colspan="4" style="text-align: center;"><img src="./media/image41.png" style="width:5.38194in;height:1.64448in" /></td>
</tr>
<tr>
<td>Endpoint /api diarahkan ke backend Flask.</td>
<td>curl -k https://localhost:8443/api/</td>
<td>enghasilkan respons JSON: {"forwarded_proto":"https","message":"API Bab 4 berhasil...","service":"flask-app","status":"ok"}</td>
<td>Berhasil</td>
</tr>
<tr>
<td colspan="4"><img src="./media/image42.png" style="width:6.5in;height:0.37986in" /></td>
</tr>
<tr>
<td>HTTPS self-signed berjalan di port 8443</td>
<td>openssl s_client -connect localhost:8443 -servername localhost -brief &lt;/dev/null</td>
<td>Koneksi TLS berhasil terjalin (CONNECTION ESTABLISHED) menggunakan protokol TLSv1.2 atau TLSv1.3.</td>
<td>Berhasil</td>
</tr>
<tr>
<td colspan="4" style="text-align: center;"><img src="./media/image43.png" style="width:5.91667in;height:1.29585in" /></td>
</tr>
<tr>
<td>Access log tersimpan di direktori host.</td>
<td>tail -n 10 logs/nginx/access.log</td>
<td>Berkas di host menampilkan entri rekaman IP 127.0.0.1, method HTTP, path URL, dan status code 200/301.</td>
<td>Berhasil</td>
</tr>
<tr>
<td colspan="4"><img src="./media/image44.png" style="width:6.5in;height:1.14444in" /></td>
</tr>
</tbody>
</table>

6.  Jawaban Evaluasi dan Latihan Mandiri

<!-- -->

1.  **Mengapa reverse proxy tidak seharusnya menjalankan semua logic aplikasi?\**
    Pemisahan ini didasarkan pada prinsip pemisahan tanggung dan arsitektur konkurensi. Reverse proxy seperti Nginx menggunakan arsitektur event-driven non-blocking yang dirancang khusus untuk menangani ribuan koneksi I/O secara cepat (seperti terminasi TLS dan mitigasi DDoS dasar). Menjalankan logika bisnis, pemrosesan transaksi, atau komputasi database pada reverse proxy akan memblokir worker process, menghabiskan memori, serta memperluas permukaan serangan di perimeter terluar sistem. Pemisahan ini memungkinkan layer backend diskalakan secara independen tanpa mengganggu stabilitas ingress gateway.

2.  **Apa perbedaan TLS termination dan end-to-end TLS?**

- Enkripsi TLS didekripsi di lapisan terluar, yaitu pada reverse proxy. Komunikasi dari reverse proxy menuju layanan backend di jaringan internal menggunakan protokol plaintext. Keuntungannya adalah penghematan beban komputasi CPU di backend dan sentralisasi manajemen sertifikat, namun terdapat risiko jika jaringan internal host berhasil disadap penyerang.

- End-to-End TLS: Enkripsi dipertahankan secara utuh mulai dari browser klien hingga mencapai kontainer backend terdalam. Proxy hanya bertindak sebagai TCP passthrough atau melakukan re-encryption (TLS di-dekripsi lalu di-enkripsi ulang dengan sertifikat internal baru). Pola ini wajib diterapkan pada standar arsitektur Zero-Trust, atau multi-tenant cluster yang tidak mengasumsikan jaringan internal sebagai area aman.

3.  **Bagaimana cara mengisolasi backend agar tidak langsung diakses dari host?**

> Isolasi dilakukan dengan tidak mendefinisikan blok ports (misalnya "5000:5000" atau "80:80") pada service backend di compose.yaml. Layanan backend hanya didaftarkan ke dalam user-defined bridge network internal (misalnya networks: \[web-net\]). Dengan ini, Docker Engine tidak akan membuat aturan NAT/iptables pada antarmuka host, sehingga port backend tidak pernah terbuka ke dunia luar maupun loopback host. Akses ke backend secara fisik dibatasi dan hanya dapat dicapai melalui resolusi DNS internal oleh kontainer yang berada di dalam jaringan virtual yang sama.

4.  **Apa konsekuensi menyimpan private key TLS di bind mount?**

- Keamanan berkas private key bergantung penuh pada kontrol akses berkas di mesin host. Jika berkas tidak diatur dengan hak ketat (chmod 600), pengguna lain atau proses non-privilese di mesin host dapat membaca berkas kunci privat tersebut. Lalu jika direktori bind mount tidak dipasang dengan atribut read-only (:ro), kontainer yang mengalami celah keamanan dapat memanipulasi, mengganti, atau menghapus sertifikat tersebut dari dalam kontainer.

5.  **Bandingkan log Nginx dan log Apache dari sisi format dan kegunaan debugging!**

- Log Nginx :

  - Secara default menggunakan format combined yang ringkas, satu baris per request (mencakup IP klien, identitas user, timestamp, HTTP request line, status code, bytes sent, referrer, dan user-agent).

  - Nginx unggul untuk melacak metrik proxy melalui variabel tambahan seperti \\(upstream_status, \\)upstream_response_time, dan upstream_connect_time. Hal ini memudahkan operator mendeteksi apakah kelambatan atau galat 502/504 bersumber dari Nginx sendiri atau dari server backend yang lambat merespons.

- Log Apache :

  - Memisahkan access_log dan error_log dengan struktur penandaan modul eksplisit yang mencantumkan level severitas (seperti \[module:severity\] \[pid X:tid Y\] \[client IP:port\] pesan rincian).

  - Apache error log jauh lebih deskriptif untuk mendiagnosis masalah internal aplikasi tingkat rendah, seperti kegagalan modul CGI, parsing berkas konfigurasi, penolakan akses akibat aturan .htaccess, atau kesalahan izin baca direktori fisik document root.
