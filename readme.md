
# FINAL-PROJECT-OS-SERVER-SYSTEM-ADMIN---23.83.0961

Pada Repository tersebut berisi dokumentasi pengerjaan saya pada saat memberikan layanan server terhadap WEB yang sebelumnya saya buat.

## Daftar Isi
- [Layanan Server Nginx](#1-layanan-nginx)
- [Layanan Server Apache2](#2-layanan-apache2)
- [Layanan Mysql](#3-layanan-mysql)
- [Layanan Grafana](#4-layanan-grafana)
- [Layanan Docker](#5-layanan-docker)


# 1. LAYANAN NGINX
Nginx adalah perangkat lunak web server open-source yang berfungsi sebagai reverse proxy, load balancer, dan HTTP cache. Nginx memiliki beberapa kelebihan, di antaranya: 
- Dapat menangani banyak koneksi secara bersamaan 
- Efisien dalam penggunaan sumber daya 
- Dukungan OS yang luas, termasuk Linux, Unix, dan Windows 
- Pemeliharaan konfigurasi real-time 

### Konfigurasi

### Langkah 1: Update Sistem
Pertama-tama, pastikan sistem Anda sudah diperbarui dengan perintah berikut:

```bash
sudo apt update
sudo apt upgrade

```

### Langkah 2: Instalasi Nginx
Instal Nginx dengan perintah berikut:

```bash
sudo apt install nginx
```

Setelah instalasi selesai, Anda dapat memulai dan mengaktifkan layanan Nginx:

```bash
sudo systemctl start nginx
sudo systemctl enable nginx
```

### Langkah 3: Izinkan Akses Firewall
Jika Anda menggunakan UFW (Uncomplicated Firewall), izinkan akses ke port 80 (HTTP) dan 443 (HTTPS):

```bash
sudo ufw allow 'Nginx HTTP'
sudo ufw allow 'Nginx HTTPS'
sudo ufw status
```

### Langkah 4: Konfigurasi Virtual Host
Buat direktori untuk situs web Anda, disini saya menamakan file nya itu "server1":
```bash
sudo mkdir -p /var/www/server1
sudo chown -R $USER:$USER /var/www/server1
sudo chmod -R 755 /var/www
```

### Clone File WEB pada github saya
Opsional jika tidak mempunyai WEB, bisa membuat File HTML sederhana seperti:
```bash
echo "Hello, World!" | sudo tee /var/www/your_domain/index.html
```

clone github, masuk terlebih dahulu ke direktori yang sudah di buat pada langkah 4:
```bash
git clone https://github.com/okeyy07-rgb/cybershild.github.io.git
```

Buat konfigurasi virtual host di /etc/nginx/sites-available/server1:
```bash
sudo nano /etc/nginx/sites-available/your_domain
```
"your_domain" di ganti dengna nama sesuai yang anda inginkan"

Tambahkan konfigurasi berikut:
```bash
server {
    listen 80 default_server;
    server_name 192.168.100.232;
    root /var/www/server1/cybershild.github.io;
    index index.html;

    location / {
        auth_basic "Restricted Content";
        auth_basic_user_file /etc/nginx/.htpasswd;
        try_files $uri $uri/ =404;

        ## Rate Limiting ##
        limit_req zone=one burst=5 nodelay;

        ## Security Headers ##
        add_header X-Frame-Options "SAMEORIGIN";
        add_header X-XSS-Protection "1; mode=block";
        add_header X-Content-Type-Options "nosniff";
        add_header Strict-Transport-Security "max-age=31536000; includeSubdomains; preload";

        ## Cache Control ##
        expires 1d;
        add_header Cache-Control "public, must-revalidate, proxy-revalidate";
    }
}

server {
    listen 443 ssl http2;
    server_name 192.168.100.232;

    ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
    ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;

    root /var/www/server1/cybershild.github.io;
    index index.html;

    location / {
        auth_basic "Restricted Content";
        auth_basic_user_file /etc/nginx/.htpasswd;
        try_files $uri $uri/ =404;

        ## Security Headers ##
        add_header X-Frame-Options "SAMEORIGIN";
        add_header X-XSS-Protection "1; mode=block";
        add_header X-Content-Type-Options "nosniff";
        add_header Strict-Transport-Security "max-age=31536000; includeSubdomains; preload";

        ## Cache Control ##
        expires 1d;
        add_header Cache-Control "public, must-revalidate, proxy-revalidate";
    }
}

server {
    listen 80;
    server_name 192.168.100.232;

    location / {
        proxy_cache my_cache;
        proxy_cache_valid 200 302 10m;
        proxy_cache_valid 404 1m;
        proxy_cache_use_stale error timeout invalid_header updating http_500 http_502 http_503 http_504;
        proxy_cache_lock on;
        proxy_cache_revalidate on;
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        try_files $uri $uri/ =404;

        ## Security Headers ##
        add_header X-Frame-Options "SAMEORIGIN";
        add_header X-XSS-Protection "1; mode=block";
        add_header X-Content-Type-Options "nosniff";
        add_header Strict-Transport-Security "max-age=31536000; includeSubdomains; preload";

        ## Cache Control ##
        expires 1d;
        add_header Cache-Control "public, must-revalidate, proxy-revalidate";
    }
}
```
Konfigurasi pertama adalah blok server untuk HTTP yang mendengarkan permintaan pada port 80 dan bertindak sebagai server default. Konfigurasi ini mencakup otentikasi dasar untuk melindungi konten, dan pengaturan try_files untuk memastikan hanya file yang valid yang akan dilayani. Selain itu, ada fitur Rate Limiting yang membatasi permintaan dari klien dengan menggunakan zona one dengan burst sebesar 5 permintaan. Ditambahkan juga header keamanan untuk mencegah berbagai jenis serangan seperti XSS dan penyisipan konten tidak aman, serta pengaturan kontrol cache yang mengatur masa berlaku konten di sisi klien selama satu hari.

Selanjutnya, blok server kedua adalah untuk HTTPS yang mendengarkan pada port 443 dengan HTTP/2 dan menggunakan sertifikat self-signed untuk enkripsi. Konfigurasi ini sangat mirip dengan blok server HTTP, dengan tambahan pengaturan SSL menggunakan sertifikat self-signed.

Terakhir, blok server ketiga digunakan untuk reverse proxy dan caching. Blok ini mengaktifkan caching proxy menggunakan zona my_cache, dengan pengaturan masa berlaku cache untuk respons HTTP tertentu dan penggunaan cache lama jika terjadi kesalahan. Permintaan diteruskan ke backend yang didefinisikan di upstream dengan pengaturan header proxy yang sesuai. Blok ini juga memiliki pengaturan keamanan dan kontrol cache yang sama seperti blok server lainnya.

Dengan konfigurasi ini, server Anda akan memiliki fitur-fitur tambahan seperti reverse proxy, load balancing (jika ditambahkan lebih banyak server pada upstream), HTTP/2, rate limiting, gzip compression, cache control, dan security headers untuk meningkatkan keamanan, kinerja, dan fungsionalitas server web Anda.

### Konfigurasi global pada nginx
masukkan konfigurasi berikut pada /etc/nginx/nginx.conf:
```bash
events {
    worker_connections 1024; # Anda bisa menyesuaikan jumlah koneksi sesuai kebutuhan
}

http {
    ## Basic Settings ##
    sendfile on;
    tcp_nopush on;
    types_hash_max_size 2048;
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    ## Rate Limiting ##
    limit_req_zone $binary_remote_addr zone=one:10m rate=1r/s;

    ## Gzip Compression ##
    gzip on;
    gzip_comp_level 2;
    gzip_min_length 512;
    gzip_proxied any;
    gzip_vary on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

    ## Define Proxy Cache Path ##
    proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=my_cache:10m max_size=1g inactive=60m use_temp_path=off;

    ## Load Balancing Upstream ##
    upstream backend {
        server 127.0.0.1:8080;
    }

    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```
### Langkah 5: Aktifkan Virtual Host
Hubungkan konfigurasi virtual host ke /etc/nginx/sites-enabled/server1:
```bash
sudo ln -s /etc/nginx/sites-available/your_domain /etc/nginx/sites-enabled/
```

### Langkah 6: Restart Nginx
Restart Nginx untuk menerapkan perubahan:
```bash
sudo systemctl restart nginx
```

### Langkah 7: Tes Konfigurasi
Cek apakah situs Anda dapat diakses dengan mengetik 192.168.100.232 (sesuai nama domain yang sudah anda buat) di peramban web disini saya menggunakan local host.
![App Screenshot](loginnginx.png)
![App Screenshot](halamanutama.png)


# 2. LAYANAN APACHE2
Apache2 adalah layanan web server yang berfungsi untuk mengelola dan melayani situs web dan aplikasi web melalui internet:

Cara kerja
Apache menerima permintaan dari browser pengunjung, memprosesnya, dan mengirimkan respons berupa halaman web atau konten lainnya. 


### Konfigurasi
### Langkah 1: Hentikan terlebih dahulu layanan nginx
```bash
sudo systemctl stop nginx
```

### Langkah 2: Instal Apache2
Pertama, instal Apache2 dengan perintah berikut:
```bash
sudo apt update
sudo apt install apache2
```

### Langkah 3: Mengatur Izin dan Direktori
Buat direktori untuk situs web Anda jika belum ada:
```bash
sudo mkdir -p /var/www/server2
sudo chown -R $USER:$USER /var/www/server2
sudo chmod -R 755 /var/www
```

### Langkah 4: Clone Kembali WEb pada github
Opsional jika tidak mempunyai WEB, bisa membuat File HTML sederhana seperti:
```bash
echo "Hello, World!" | sudo tee /var/www/your_domain/index.html
```

clone github, masuk terlebih dahulu ke direktori server2:
```bash
git clone https://github.com/okeyy07-rgb/cybershild.github.io.git
```


### Langkah 5: Buat File Konfigurasi Virtual Host
Buat file konfigurasi virtual host untuk situs Anda di /etc/apache2/sites-available/server2.conf:
```bash
sudo nano /etc/apache2/sites-available/server2.conf
```

Tambahkan konfigurasi berikut:
```bash
<VirtualHost *:80>
    ServerAdmin webmaster@192.168.100.232
    ServerName 192.168.100.232
    DocumentRoot /var/www/server2/cybershild.github.io

    <Directory /var/www/server2/cybershild.github.io>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    # Rate Limiting
    <Location "/">
        SetOutputFilter RATE_LIMIT
        SetEnv rate-limit 400  # kecepatan transfer dalam bytes per detik
    </Location>

    # Gzip Compression
    <IfModule mod_deflate.c>
        AddOutputFilterByType DEFLATE text/html text/plain text/xml text/css application/javascript application/json
    </IfModule>

    # Security Headers
    Header always set X-Frame-Options "SAMEORIGIN"
    Header always set X-XSS-Protection "1; mode=block"
    Header always set X-Content-Type-Options "nosniff"
    Header always set Strict-Transport-Security "max-age=31536000; includeSubdomains; preload"

    # Cache Control
    <FilesMatch "\.(html|css|js|png|jpg|jpeg|gif|ico)$">
    Header set Cache-Control "max-age=2592000, public"
    </FilesMatch>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
Pada Konfigurasi Apache tersebut, telah ditambahkan dengan beberapa fitur untuk meningkatkan kinerja dan keamanan layanan web. Pada bagian VirtualHost *:80, pengaturan dasar seperti ServerAdmin, ServerName, dan DocumentRoot sudah ada, serta pengaturan untuk mengakses direktori dokumen dengan Options Indexes FollowSymLinks, AllowOverride All, dan Require all granted.

Untuk membatasi kecepatan transfer data, saya telah menambahkan konfigurasi Rate Limiting menggunakan modul mod_ratelimit dengan mengatur SetOutputFilter RATE_LIMIT dan SetEnv rate-limit 400, yang membatasi kecepatan transfer menjadi 400 byte per detik. Selain itu, saya menambahkan Gzip Compression untuk mengompresi konten sebelum mengirimkannya ke klien, menggunakan modul mod_deflate dengan pengaturan AddOutputFilterByType DEFLATE untuk berbagai tipe konten seperti HTML, CSS, dan JavaScript.

Untuk meningkatkan keamanan, saya menambahkan beberapa Security Headers. Header X-Frame-Options diatur ke "SAMEORIGIN" untuk mencegah klikjacking, X-XSS-Protection diatur ke "1; mode=block" untuk mencegah serangan XSS, X-Content-Type-Options diatur ke "nosniff" untuk mencegah MIME-type sniffing, dan Strict-Transport-Security diatur untuk memastikan komunikasi HTTPS. saya juga menambahkan pengaturan Cache Control dengan menggunakan FilesMatch untuk mencocokkan berbagai tipe file seperti HTML, CSS, JS, dan gambar, dan mengatur header Cache-Control dengan masa berlaku cache selama 30 hari.

Keseluruhan konfigurasi ini juga mencakup pengaturan logging untuk error dan akses, dengan ErrorLog dan CustomLog yang mengarah ke file log yang tepat. Dengan fitur-fitur ini, server Apache  akan lebih aman, efisien, dan memberikan pengalaman pengguna yang lebih baik.

Berikut adalah service yang saya tambahkan pada layanan apache, beserta konfigurasi yang terpisah:

### 1. Rate Limiting
Untuk melakukan rate limiting, kita bisa menggunakan modul mod_ratelimit di Apache.

Aktifkan Modul:
```bash
sudo a2enmod ratelimit
```
Tambahkan Konfigurasi Rate Limiting ke Virtual Host:
```bash
<Location "/">
    SetOutputFilter RATE_LIMIT
    SetEnv rate-limit 400  # kecepatan transfer dalam bytes per detik
</Location>
```

### 2. Gzip Compression:
Untuk mengaktifkan gzip compression, kita bisa menggunakan modul mod_deflate di Apache.

Aktifkan Modul:
```bash
sudo a2enmod deflate
```
Tambahkan Konfigurasi Gzip Compression ke Virtual Host:
```bash
<IfModule mod_deflate.c>
    AddOutputFilterByType DEFLATE text/html text/plain text/xml text/css application/javascript application/json
</IfModule>
```

### 3. Security Headers:
Tambahkan Konfigurasi Security Headers ke Virtual Host:
```bash
Header always set X-Frame-Options "SAMEORIGIN"
Header always set X-XSS-Protection "1; mode=block"
Header always set X-Content-Type-Options "nosniff"
Header always set Strict-Transport-Security "max-age=31536000; includeSubdomains; preload"
```
### 4. Cache Control
Tambahkan Konfigurasi Cache Control ke Virtual Host:
```bash
<FilesMatch "\.(html|css|js|png|jpg|jpeg|gif|ico)$">
    Header set Cache-Control "max-age=2592000, public"
</FilesMatch>
```


### Langkah 6: Aktifkan Virtual Host
Aktifkan file konfigurasi virtual host dan nonaktifkan konfigurasi default jika perlu:
```bash
sudo a2ensite server2.conf
sudo a2dissite 000-default.conf
```

### Langkah 7: Reload Apache2
Reload layanan Apache2 untuk menerapkan perubahan:
```bash
sudo systemctl reload apache2
```

### Langkah 6: Tes Konfigurasi
Coba akses server Anda dengan memasukkan alamat IP atau nama domain yang sesuai di peramban:
![App Screenshot](hasil2.png)


# 3. LAYANAN MYSQL
Layanan MySQL adalah sistem manajemen basis data relasional (RDBMS) yang digunakan 
untuk membuat tabel dan menyimpan data. MySQL merupakan perangkat lunak open 
source yang populer dan banyak digunakan di berbagai aplikasi web, situs web dinamis, 
dan sistem tertanam. 
Jangan Lupa untuk menghentikan layanan yang sedang aktif.
### Langkah 1: Instalasi MySQL
Pertama, instal MySQL Server dengan perintah berikut:
```bash
sudo apt update
sudo apt install mysql-server
```

### Langkah 2: Mengamankan Instalasi MySQL
Setelah instalasi selesai, amankan instalasi MySQL dengan menjalankan skrip keamanan:
```bash
sudo mysql_secure_installation
```
Ikuti petunjuk di layar untuk mengatur kata sandi root, menghapus pengguna anonim, menonaktifkan login root jarak jauh, dan menghapus database uji.

### Langkah 3: Masuk ke MySQL
Masuk ke shell MySQL sebagai pengguna root:
```bash
sudo mysql -u root -p
```
Masukkan kata sandi root yang Anda atur sebelumnya.

### Langkah 4: Membuat Database dan Pengguna
1. Buat Database Baru: Buat database baru bernama FinalProjectDB:
```sql
CREATE DATABASE FinalProjectDB;
```

2. Buat Pengguna Baru: Buat pengguna baru bernama finaluser dan atur kata sandinya menjadi password (pastikan Anda mengganti password dengan kata sandi yang lebih aman sesuai kebijakan Anda) pada localhost saya mengisi dengan IP saya:
```sql
CREATE USER 'finaluser'@'192.168.100.232' IDENTIFIED BY 'password';
```

3. Berikan Hak Istimewa: Berikan semua hak istimewa pada database FinalProjectDB untuk pengguna finaluser:
```sql
GRANT ALL PRIVILEGES ON FinalProjectDB.* TO 'finaluser'@'192.168.100.232';
```

4. Flush Privileges: Muat ulang tabel hak istimewa agar perubahan diterapkan:
```sql
FLUSH PRIVILEGES;
```

5. Keluar dari MySQL: Keluar dari shell MySQL:
```sql
EXIT;
```

Berikut adalah rangkuman perintah-perintah tersebut:
```sql
CREATE DATABASE FinalProjectDB;
CREATE USER 'finaluser'@'192.168.100.232' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON FinalProjectDB.* TO 'finaluser'@'192.168.100.232';
FLUSH PRIVILEGES;
EXIT;
```

### Langkah 5:Buat atau Edit File Konfigurasi PHP
Buat atau edit file konfigurasi PHP untuk menangani koneksi ke database MySQL. Anda bisa membuat file baru bernama db_connect.php di direktori root aplikasi web Anda.
1. Buat File db_connect.php:
```bash
sudo nano /var/www/server3/db_connect.php
```

2. Tambahkan Kode Koneksi MySQL: Masukkan kode berikut ke dalam file db_connect.php:
```php
<?php
$db_host = '192.168.100.232';
$db_name = 'FinalProjectDB';
$db_user = 'finaluser';
$db_pass = 'maulanaoki07';

$mysqli = new mysqli($db_host, $db_user, $db_pass, $db_name);

if ($mysqli->connect_error) {
    die("Connection failed: " . $mysqli->connect_error);
}
echo "Connected successfully";
?>
```

### Langkah 6: Buat Formulir HTML
Sebelumnya saya seperti biasa cloning terlebih dahulu file web saya dari github di directory server3:
```bash
git clone https://github.com/okeyy07-rgb/cybershild.github.io.git
```
setelah itu:
1. Buat atau Edit File index.html:
```bash
sudo nano /var/www/server3/cybershild.github.io/index.html
```

2. Tambahkan Formulir HTML: Masukkan kode berikut ke dalam file index.html:
```html
<!DOCTYPE html>
<html>
<head>
    <title>Formulir Kontak</title>
</head>
<body>
    <h1>Formulir Kontak</h1>
    <form action="submit_form.php" method="post">
        Nama: <input type="text" name="nama"><br>
        Email: <input type="email" name="email"><br>
        Pesan: <textarea name="pesan"></textarea><br>
        <input type="submit" value="Kirim">
    </form>
</body>
</html>
```

### Langkah 7: Buat File PHP untuk Memproses Formulir
Buat file PHP baru bernama submit_form.php yang akan memproses data dari formulir dan menyimpannya ke database MySQL.
1. Buat File submit_form.php:
```bash
sudo nano /var/www/server3/submit_form.php
```
2. Tambahkan Kode PHP untuk Memproses Data: Masukkan kode berikut ke dalam file submit_form.php:
```php
<?php
include 'db_connect.php';

$nama = $_POST['nama'];
$email = $_POST['email'];
$pesan = $_POST['pesan'];

$sql = "INSERT INTO kontak (nama, email, pesan) VALUES ('$nama', '$email', '$pesan')";

if ($mysqli->query($sql) === TRUE) {
    echo "Pesan Anda telah dikirim!";
} else {
    echo "Error: " . $sql . "<br>" . $mysqli->error;
}

$mysqli->close();
?>
```

### Langkah 8: Buat Tabel di Database
Pastikan tabel kontak sudah ada di database FinalProjectDB dengan struktur yang sesuai. Anda bisa membuat tabelnya seperti ini:
1. Masuk ke MySQL:
```bash
sudo mysql -u root -p
```
2. Pilih Database:
```sql
USE FinalProjectDB;
```
3. Buat Tabel kontak:
```sql
CREATE TABLE kontak (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nama VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL,
    pesan TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```


### Langkah 9: Uji Koneksi dan Formulir
Merestart MySQL:
```bash
sudo systemctl restart mysql
```

Akses halaman HTML Anda melalui peramban dan coba kirim formulir untuk memastikan data dikirim dan disimpan di database dengan benar:
```plaintext
http://192.168.100.232/index.html
```
Hasil pada browser dengna mengunjungi alamat "http://192.168.100.232/index.html":
![App Screenshot](hasil3.png)

## Menambah Layanan Service

1. Backup Otomatis 

   A. Instalasi mysqldump:
    
    mysqldump biasanya sudah tersedia di instalasi MySQL. Jika belum, Anda bisa menginstalnya dengan:
    ```bash
    sudo apt-get install mysql-client
    ```

   B. Buat Skrip Backup:

    Buat skrip backup otomatis yang akan menjalankan mysqldump dan menyimpan hasil backup ke direktori tertentu.
    ```bash
    sudo nano /usr/local/bin/backup_mysql.sh
    ```
    Tambahkan script berikut:
    ```bash
    #!/bin/bash
    TIMESTAMP=$(date +"%F")
    BACKUP_DIR="/path/to/backup/directory"
    MYSQL_USER="finaluser"
    MYSQL_PASSWORD="password"
    MYSQL_DATABASE="FinalProjectDB"

    mkdir -p ${BACKUP_DIR}/${TIMESTAMP}
    mysqldump --user=${MYSQL_USER} --password=${MYSQL_PASSWORD} ${MYSQL_DATABASE} > ${BACKUP_DIR}/${TIMESTAMP}/${MYSQL_DATABASE}.sql
    ```
    Ganti /path/to/backup/directory dengan direktori yang Anda inginkan untuk menyimpan backup.

    c. Berikan Izin Eksekusi pada Skrip: bash
    ```bash
    sudo chmod +x /usr/local/bin/backup_mysql.sh
    ```
    ![App Screenshot](crontab.png)
    
    Nanti akan muncul seperti gambar tersebut. Pilihlah editor yang Anda inginkan untuk mengedit crontab. Jika Anda tidak yakin, 
    opsi 1 (nano) adalah yang paling mudah dan user-friendly. Anda bisa memilihnya dengan mengetikkan angka 1 dan menekan Enter.

    Setelah memilih editor, Anda akan diarahkan ke file crontab untuk root. Tambahkan baris berikut untuk menjadwalkan backup MySQL 
    setiap hari pada pukul 2 pagi:
    ```bash
    0 2 * * * /usr/local/bin/backup_mysql.sh
    ```
#
2. Optimasi Database

    a. Masuk ke MySQL Shell
    Jalankan perintah berikut di terminal untuk masuk ke MySQL shell:
    ```bash
    sudo mysql -u root -p
    ```
    Kemudian, masukkan password root MySQL Anda.

    b. Pilih Database yang Ingin Dioptimalkan
    Pilih database yang ingin Anda optimalkan:
    ```sql
    USE FinalProjectDB;
    ```

    c. Jalankan Perintah OPTIMIZE TABLE
    Jalankan perintah OPTIMIZE TABLE pada tabel yang ingin dioptimalkan. Dalam kasus ini, tabel kontak:
    ```sql
    OPTIMIZE TABLE kontak;
    ```
#
3. Monitopring Database

    a. Instalasi MySQLTuner:
    MySQLTuner adalah skrip Perl yang membantu mengoptimalkan dan memberikan rekomendasi untuk pengaturan MySQL.
    ```bash
    sudo apt-get install mysqltuner
    ```

    Jalankan MySQLTuner:
    ```bash
    sudo mysqltuner
    ``` 

    b. Periksa Log File untuk Peringatan
    MySQLTuner menunjukkan bahwa ada 2 peringatan dalam file log MySQL. Anda perlu memeriksa detail peringatan tersebut untuk 
    mengambil tindakan yang sesuai.
    Lihat log MySQL untuk peringatan:
    ```bash
    sudo cat /var/log/mysql/error.log
    ```
    Periksa dan pahami peringatan yang ada, kemudian ambil langkah yang diperlukan untuk mengatasinya.

    c. Aktifkan skip-name-resolve
    Aktifkan pengaturan skip-name-resolve untuk menghindari resolusi nama balik yang dapat mengurangi kinerja.
     Ini mengharuskan Anda untuk menggunakan alamat IP daripada nama host dalam konfigurasi pengguna MySQL.
     Edit file konfigurasi MySQL:
     ```bash
     sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
    ```

    Tambahkan baris berikut di bagian [mysqld]:
    ```Ini
    [mysqld]
    skip-name-resolve
    innodb_redo_log_capacity = 32M
    ```

    Restart MySQL untuk menerapkan perubahan:
    ```bash
    sudo systemctl restart mysql
    ```

## 4. LAYANAN GRAFANA
Layanan Grafana adalah perangkat lunak open source yang digunakan untuk 
visualisasi data dan monitoring. Grafana dapat mengambil data dari berbagai
 sumber dan memungkinkan pengguna untuk membuat dashboard yang menampilkan data 
 dalam bentuk grafik, tabel, dan grafis lainnya. 

 ### Langkah 1: Install Dependencies
 Pastikan semua paket yang dibutuhkan sudah terinstal:
 ```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https software-properties-common wget
```

### Langkah 2: Tambahkan Repository Grafana
Tambahkan repository Grafana ke sistem Anda:
```bash
sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
sudo wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
```

### Langkah 3: Install Grafana
Instal Grafana dari repository yang telah ditambahkan:
```bash
sudo apt-get update
sudo apt-get install grafana
```

### Langkah 4: Jalankan dan Mulai Grafana
Setelah instalasi selesai, jalankan dan mulai server Grafana:
```bash
sudo systemctl start grafana-server
sudo systemctl enable grafana-server
```

### Langkah 5: Akses Grafana melalui Browser
Buka browser Anda dan kunjungi alamat IP server Anda dengan menambahkan port 3000:
```plaintext
http://192.168.100.232:3000
```
![App Screenshot](Awal_garfana.png)

Nanti akan muncul seperti ini.

Saat Anda pertama kali mengakses halaman login Grafana, Anda akan diminta untuk 
masuk dengan kredensial default. Kredensial default untuk Grafana biasanya adalah:

- Username: admin

- Password: admin

Setelah Anda masuk menggunakan kredensial ini, Anda akan diminta untuk mengubah 
password admin. Berikut adalah langkah-langkah yang perlu Anda ikuti:

1. Masukkan kredensial default:

    - Username: admin

    - Password: admin

2. Ganti Password: Setelah login pertama kali, Anda akan diminta untuk mengganti 
password admin. Masukkan password baru Anda dan konfirmasi password tersebut.

3. Masuk dengan Password Baru: Gunakan password baru yang Anda buat untuk login 
kembali ke Grafana.

    Lalu akan masuk ke halaman beranda grafana:

![App Screenshot](beranda.png)

### Langkah 6: Tambahkan Sumber Data (Data Source)

1. Klik pada "Connections" di sidebar kiri dan pilih "Data Sources".

2. Klik "Add data source".

3. Pilih jenis sumber data yang sesuai dengan situs web Anda. Misalnya, 
jika Anda ingin memantau data dari MySQL, pilih "MySQL".

### Langkah 7: Konfigurasi Sumber Data
1. Isi detail konfigurasi yang diperlukan:

    - Name: Berikan nama untuk sumber data ini.

    - Host: Masukkan alamat IP atau hostname dari server database Anda.

    - Database: Nama database Anda (misalnya, FinalProjectDB).

    - User: Nama pengguna database (misalnya, finaluser).

    - Password: Kata sandi untuk pengguna database.

Jika Anda memilih MySQL sebagai sumber data, konfigurasi ini akan terlihat 
seperti ini:
![App Screenshot](1.png)
![App Screenshot](2.png)

setelah selesai di setting, lalu klik "Save & Test". jik berhasil terhubung,
maka akan muncul pemberitahuan sebagai berikut:
![App Screenshot](3.png)


### Langkah 8: Buat Dashboard Baru
1. Akses Menu Dashboards:

    - Di sidebar kiri, klik "Dashboards".

    - Pilih "New Dashboard".

2. Tambah Panel Baru:

    - Klik "+ Add visualization".

    - Anda akan masuk ke mode editor panel.

### Langkah 9: Konfigurasi Panel
1. Pilih Sumber Data:

    - Di panel editor, pilih sumber data MySQL yang baru saja Anda konfigurasikan.

2. Buat Query untuk Mengambil Data:

    - Di bagian "Query", tulis query SQL untuk mengambil data yang ingin Anda 
        visualisasikan, tetapi query mengikuti jenis visualisasi yang dipakai. 
        Contohnya:
```sql
SELECT
  kategori AS label,
  COUNT(*) AS value
FROM
  kontak
GROUP BY
  kategori;
```

3. Pilih Jenis Visualisasi:

    - Di panel editor, pilih jenis visualisasi yang Anda inginkan, seperti grafik, 
    tabel, gauge, dsb. disini say menggunakan "PIE CHART"


### Langkah 10: Verifikasi Data di Database
Pastikan tabel kontak memiliki data yang sesuai untuk ditampilkan. 
Jalankan query berikut langsung di MySQL untuk memverifikasi data:
1. Masuk ke MySQL:
```bash
sudo mysql -u root -p
```

2. Pilih Database:
```sql
USE FinalProjectDB;
```

3. Periksa Struktur Tabel:
```sql
DESCRIBE kontak;
```
Ini akan menampilkan semua kolom dalam tabel kontak. Pastikan kolom kategori dan kolom lain yang 
diperlukan ada.

### Langkah 11: Tambahkan Kolom Jika Perlu
Jika kolom kategori tidak ada, Anda bisa menambahkannya. Misalnya, jika Anda ingin 
menambahkan kolom kategori dengan tipe data VARCHAR, Anda bisa menggunakan query berikut:
```sql
ALTER TABLE kontak ADD COLUMN kategori VARCHAR(255);
```

### Langkah 12: Tambah Data ke Tabel kontak
Berikut adalah query untuk menambahkan data ke tabel kontak menggunakan kolom yang ada 
(kategori, nama, email, pesan, created_at):
```sql
INSERT INTO kontak (kategori, nama, email, pesan, created_at)
VALUES 
('Kategori1', 'Nama1', 'email1@example.com', 'Pesan1', NOW()),
('Kategori2', 'Nama2', 'email2@example.com', 'Pesan2', NOW()),
('Kategori1', 'Nama3', 'email3@example.com', 'Pesan3', NOW());
```

### Langkah 13: Verifikasi Data di Tabel
Setelah menambahkan data, verifikasi isi tabel kontak:
```sql
SELECT * FROM kontak;
```

### Langkah 14: Jalankan Query Pie Chart di Grafana
![App Screenshot](4.png)
Jika berhasil, maka ketika query yang dibuat di jalankan akan mengeluarkan Pie Chart nya.

## Menambahkan Service Grafana

### 1. Langkah-langkah Menyetel Alert di grafana
Grafana dapat mengirimkan pemberitahuan ketika metrik tertentu mencapai ambang batas yang telah ditentukan. 
Anda dapat mengkonfigurasi alerting melalui panel Grafana untuk mendapatkan peringatan real-time.

Langkah-Langkah:
1. Buka dasbor Grafana dan pilih panel yang ingin Anda tambahkan alerting.
![App Screenshot](allert.png)

2. Masukkan Nama untuk Aturan Alert:
Di bagian "Enter alert rule name", masukkan nama yang deskriptif untuk aturan alert Anda. Misalnya, "Alert Kontak Kategori".

3. Definisikan Query dan Kondisi Alert:
   - Data Source: Pilih mysql sebagai sumber data.

   - Time Interval: Setel interval waktu, misalnya 10 minutes.

   - Database: Pilih database FinalProjectDB.

   - Table: Pilih tabel kontak.

   - Column: Pilih kolom yang ingin Anda pantau, misalnya kategori.

   - Aggregation: Pilih metode agregasi yang sesuai, misalnya COUNT.
   - Alias: Berikan nama alias untuk kolom hasil agregasi, misalnya jumlah_kategori
![App Screenshot](settingg1.png)
   - Klik Run Query untuk melihat hasil query dan memverifikasi apakah data yang diambil sudah sesuai.

4. Pilih Tipe Aturan:
Di bagian "Rule type", Anda bisa memilih apakah aturan alert akan dikelola oleh Grafana (Grafana-managed) atau oleh sumber data (data source-managed). Untuk kemudahan, pilih Grafana-managed.
![App Screenshot](settingg2.png)

    Di bagian "Expressions", Anda bisa menambahkan ekspresi matematika atau operasi lainnya untuk memanipulasi data yang dikembalikan dari query. Misalnya, Anda bisa menambahkan ekspresi untuk memeriksa apakah jumlah_kategori melebihi nilai tertentu.

    Expression 1: Reduce
    - Type: Reduce

    - Input: A (hasil dari query SQL di atas)

    - Function: Last (mengambil nilai terakhir dari hasil query)

    - Mode: Strict

    Setting Reduce:
    - Pilih Reduce di bagian Expressions.

    - Tetapkan Input ke A.

    - Pilih Function Last.

    - Pilih Mode Strict.

    Expression 2: Threshold
    - Pilih Threshold di bagian Expressions.

    - Tetapkan Input ke B (hasil dari ekspresi Reduce).

    - Pilih Condition IS ABOVE dan tetapkan nilai 0.

5. Set evaluation behavior
![App Screenshot](settingg3.png)
    1. Folder

        Choose Folder: Pilih folder tempat aturan alert akan disimpan. Anda bisa menggunakan folder yang sudah ada atau membuat folder baru dengan mengeklik tombol "+ New folder".

    2. Evaluation group and interval
        -  Evaluation Group: Pilih grup evaluasi yang sesuai untuk alert ini. Jika belum ada grup yang cocok, Anda bisa membuat grup baru dengan mengeklik tombol "+ New evaluation group".

        - Interval: Tentukan interval evaluasi, misalnya 1m atau 5m, tergantung seberapa sering Anda ingin Grafana memeriksa kondisi alert.
    
    3. Pending period

        Pending Period: Tentukan periode pending yang diperlukan agar kondisi threshold terpenuhi sebelum alert dikirim. Anda bisa memasukkan nilai seperti 1m, 2m, atau 5m. Misalnya, jika Anda menetapkan 1m, Grafana akan menunggu selama 1 menit setelah kondisi threshold terpenuhi sebelum mengirimkan alert.

6. Configure labels and notifications
![App Screenshot](settingg4.png)
    1. Labels

        Label digunakan untuk menambahkan metadata ke alert Anda yang dapat membantu mengkategorikan dan mengelola alert. Anda bisa menambahkan label seperti severity, team, atau environment.

        Langkah-langkah Menambahkan Label:
        - Klik tombol "Add labels".

        - Tambahkan label sesuai dengan kebutuhan Anda. Misalnya:

        - Key: severity

        - Value: high

        - Anda bisa menambahkan lebih banyak label jika diperlukan.

    2. Notification Channels (Contact Points)

        Di bagian ini, Anda bisa mengatur bagaimana dan kepada siapa notifikasi alert akan dikirim. Grafana mendukung berbagai saluran notifikasi seperti email, Slack, dan lainnya.

    3. Optional: Muting, Grouping, and Timings
        Di bagian ini, Anda bisa mengatur pengaturan tambahan untuk muting, pengelompokan, dan penjadwalan notifikasi.


7. Configure Notification Massage
![App Screenshot](settingg5.png)
    1. Summary (Optional)

        Field ini digunakan untuk memberikan ringkasan singkat tentang apa yang terjadi dan mengapa alert ini dipicu.

        Contoh:
        - Summary: "Alert Kategori Kontak"

        - Penjelasan: "Jumlah kategori kontak melebihi ambang batas yang telah ditentukan."

    2. Description (Optional)

        Field ini digunakan untuk memberikan deskripsi lebih rinci tentang aturan alert. Di sini Anda dapat menjelaskan apa yang dilakukan oleh aturan alert dan konteks tambahan yang mungkin diperlukan.

        Contoh:

        Description: "Aturan alert ini memantau jumlah entri dalam kategori 'Kategori1' di tabel 'kontak' pada database 'FinalProjectDB'. Alert akan dipicu jika jumlah entri melebihi 5."

#### setelah selesai melakukan konfigurasi, klik "Save". ketika sudah save, maka akan muncul halaman seperti berikut:
![App Screenshot](setting6.png)



## 5. LAYANAN DOCKER
Docker adalah platform perangkat lunak open-source yang memungkinkan pengguna untuk:
- Membuat, menguji, dan menjalankan aplikasi dalam kontainer
- Mengelola siklus hidup aplikasi dalam kontainer, dari pengembangan hingga pengujian dan penerapan
- Memastikan aplikasi yang dibuat di lingkungan pengembangan akan berfungsi dengan cara yang 
sama di lingkungan produksi 

### Langkah Instalasi dan Konfigurasi:
### Langkah 1: Persiapkan Prasyarat
Pastikan Anda menggunakan versi 64-bit dari Ubuntu yang kompatibel dengan Docker, seperti 
Ubuntu 20.04 (LTS) atau versi yang lebih baru.

### Langkah 2: Hapus Paket Konflik
Sebelum menginstal Docker, pastikan untuk menghapus paket yang mungkin akan berkonflik 
dengan Docker. Jalankan perintah berikut:
```bash
sudo apt-get remove docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc
```

### Langkah 3: Tambahkan Repository Docker
Tambahkan repository Docker ke sistem Anda dengan menjalankan perintah berikut:
```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```

### Langkah 4: Instal Docker
1. Update package database:

```bash
sudo apt-get update
```

2. Install Docker
```bash
sudo apt-get install -y docker-ce docker-ce-cli containerd.io
```

3. Mulai dan aktifkan Docker:
```bash
sudo systemctl start docker
sudo systemctl enable
```

### Langkah 5: Verifikasi Instalasi
```bash
sudo docker run hello-world
```
Jika Anda melihat pesan "Hello from Docker!", maka instalasi berhasil.

### Langkah 6: Cloning File WEB yang berada pada github
```bash
git clone https://github.com/okeyy07-rgb/cybershild.github.io.git
```

### Langkah 6: Buat Dockerfile
Di dalam direktori project Anda, buat file bernama Dockerfile dengan konten berikut:
```Dockerfile
# Gunakan image resmi Nginx
FROM nginx:alpine

# Salin konten situs web dari repository ke direktori Nginx
COPY ./cybershild.github.io /usr/share/nginx/html

# Ekspose port 80 untuk HTTP
EXPOSE 80

# Perintah untuk menjalankan Nginx
CMD ["nginx", "-g", "daemon off;"]
```

### Langkah 7: Bangun Ulang Image Docker
Bangun ulang image Docker menggunakan Dockerfile yang sudah diperbarui:
```bash
docker build -t my-html-website .
```

### Langkah 7: Jalankan Container Docker
Setelah image berhasil dibangun, jalankan container Docker:
```bash
docker run -d -p 80:80 --name my-html-container my-html-website
```


### Langkah 8: Verifikasi
Buka browser dan akses http://localhost atau IP server Anda untuk memastikan 
situs web HTML Anda berjalan di dalam container Docker.
![App Screenshot](Hasil5.png)


