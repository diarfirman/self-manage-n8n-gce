# **Dokumentasi Setup n8n dengan HTTPS, Domain, dan Docker Compose di GCE**

Dokumen ini merangkum langkah-langkah untuk men-setup instance n8n self-hosted menggunakan Docker Compose di Google Compute Engine (GCE) Ubuntu, dengan konfigurasi HTTPS menggunakan domain gratis dari DuckDNS dan sertifikat SSL dari Let's Encrypt.

**Tujuan Akhir:**

* n8n berjalan dan dapat diakses melalui domain kustom dengan HTTPS yang valid.  
* Konfigurasi n8n (workflows, credentials) tersimpan secara persisten.  
* Webhook n8n berfungsi dengan layanan eksternal seperti Telegram.

**Lingkungan yang Digunakan:**

* Server: Google Compute Engine (GCE) instance dengan OS Ubuntu.  
* Deployment: Docker dan Docker Compose.  
* Domain: Subdomain gratis dari DuckDNS (misalnya, `<NAMA_DOMAIN_ANDA>.duckdns.org`).  
* Alamat IP Publik GCE: `<ALAMAT_IP_PUBLIK_ANDA>` (sebagai contoh).  
* Sertifikat SSL: Let's Encrypt.  
* Direktori Kerja di Server: Diasumsikan `/home/<NAMA_PENGGUNA_ANDA>/n8n/` untuk file `docker-compose.yml` dan data terkait.

## **Daftar Isi**

1. Persiapan Awal & Instalasi Docker  
2. Mendapatkan Nama Domain Gratis (DuckDNS)  
3. Konfigurasi Firewall GCE  
4. Mendapatkan Sertifikat SSL dari Let's Encrypt  
5. Persiapan Direktori untuk n8n di Server  
6. Konfigurasi `docker-compose.yml` untuk n8n  
7. Menjalankan n8n  
8. Mengelola n8n  
9. Mengupgrade n8n  
10. Pembaruan Sertifikat SSL Otomatis & Penerapan ke n8n  
11. Troubleshooting Umum (Ringkasan)

## **1\. Persiapan Awal & Instalasi Docker**

Pastikan server GCE Ubuntu Anda sudah terupdate dan Docker serta Docker Compose (v2 direkomendasikan) terinstal.

**Update Server:**

```
sudo apt update
sudo apt upgrade -y

```

**Install Docker:**

```
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker

```

**Install** Docker Compose **v2 (sebagai plugin Docker CLI):**

```
# Hapus versi lama jika ada (misal, docker-compose v1)
sudo apt remove docker-compose -y
# Coba hapus juga jika diinstal via pip (mungkin perlu pip3)
sudo pip uninstall docker-compose -y || sudo pip3 uninstall docker-compose -y

# Install plugin
sudo apt update
sudo apt install docker-compose-plugin -y

# Verifikasi instalasi
docker compose version

```

## **2\. Mendapatkan Nama Domain Gratis (DuckDNS)**

Untuk contoh ini, kita menggunakan DuckDNS untuk mendapatkan subdomain gratis.

1. Kunjungi [https://www.duckdns.org/](https://www.duckdns.org/).  
2. Login menggunakan salah satu metode yang disediakan.  
3. Buat subdomain (misalnya, `<NAMA_SUBDOMAIN_ANDA>`) sehingga domain lengkap Anda menjadi `<NAMA_SUBDOMAIN_ANDA>.duckdns.org`.  
4. Arahkan subdomain tersebut ke alamat IP publik statis GCE Anda (misalnya, `<ALAMAT_IP_PUBLIK_ANDA>`).  
5. Pastikan DNS sudah terpropagasi dan domain Anda mengarah ke IP yang benar. Anda bisa mengeceknya dengan `ping <NAMA_SUBDOMAIN_ANDA>.duckdns.org` dari komputer lain.

## **3\. Konfigurasi Firewall GCE**

Pastikan port yang diperlukan terbuka di firewall Google Cloud Platform untuk instance GCE Anda:

* **Port 80 (TCP):** Diperlukan untuk validasi domain oleh Let's Encrypt (HTTP-01 challenge).  
* **Port 443 (TCP):** Diperlukan untuk akses HTTPS ke n8n.

Cara membuat aturan firewall di GCP Console:

1. Navigasi ke **VPC network** \> **Firewall**.  
2. Klik **CREATE FIREWALL RULE**.  
3. Konfigurasi aturan untuk port 80 dan 443:  
   * **Name:** Deskriptif (misalnya, `allow-http-80`, `allow-https-443`).  
   * **Direction of traffic:** Ingress.  
   * **Action on match:** Allow.  
   * **Targets:** `All instances in the network` atau target spesifik menggunakan tag (misalnya, `http-server`, `https-server`).  
   * **Source filter:** IP ranges.  
   * **Source IP ranges:** `0.0.0.0/0` (untuk mengizinkan dari mana saja).  
   * **Protocols and ports:** `tcp:80` untuk HTTP, dan `tcp:443` untuk HTTPS.

Jika Anda menggunakan firewall internal di Ubuntu (seperti `ufw`), pastikan port tersebut juga diizinkan di sana.

```
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable # Jika belum aktif
sudo ufw status

```

## **4\. Mendapatkan Sertifikat SSL dari Let's Encrypt**

Gunakan `certbot` untuk mendapatkan sertifikat SSL gratis untuk domain Anda.

**Install Certbot:**

```
sudo apt install certbot -y

```

**Minta Sertifikat:** Pastikan tidak ada layanan lain yang berjalan di port 80 sebelum menjalankan perintah ini (hentikan n8n jika sedang berjalan di port 80).

```
sudo certbot certonly --standalone -d <NAMA_SUBDOMAIN_ANDA>.duckdns.org --agree-tos -m emailanda@example.com --preferred-challenges http

```

Ganti `<NAMA_SUBDOMAIN_ANDA>.duckdns.org` dengan domain Anda dan `emailanda@example.com` dengan email Anda.

Jika berhasil, sertifikat akan disimpan di `/etc/letsencrypt/live/<NAMA_SUBDOMAIN_ANDA>.duckdns.org/`. File penting:

* `fullchain.pem` (sertifikat publik \+ chain)  
* `privkey.pem` (kunci privat)

Certbot biasanya mengatur pembaruan otomatis. Anda bisa menguji dengan `sudo certbot renew --dry-run`.

## **5\. Persiapan Direktori untuk n8n di Server**

Buat direktori di host server Anda untuk menyimpan data persisten n8n dan salinan sertifikat (jika menggunakan metode salin).

**Lokasi `docker-compose.yml`:** Diasumsikan berada di `/home/<NAMA_PENGGUNA_ANDA>/n8n/docker-compose.yml`.

**Direktori Data n8n:**

```
mkdir -p /home/<NAMA_PENGGUNA_ANDA>/n8n/n8n_data
# Atur kepemilikan agar pengguna 'node' (UID 1000) di dalam container bisa menulis
sudo chown -R 1000:1000 /home/<NAMA_PENGGUNA_ANDA>/n8n/n8n_data
```

**Direktori untuk Salinan Sertifikat (Metode yang Digunakan dalam Konfigurasi Akhir):** Ini adalah direktori tempat Anda akan menyalin sertifikat dari `/etc/letsencrypt/` agar n8n dapat mengaksesnya dengan izin yang benar.

```
sudo mkdir -p /home/<NAMA_PENGGUNA_ANDA>/n8n/n8n_certs
# Salin sertifikat
sudo cp /etc/letsencrypt/live/<NAMA_SUBDOMAIN_ANDA>.duckdns.org/fullchain.pem /home/<NAMA_PENGGUNA_ANDA>/n8n/n8n_certs/fullchain.pem
sudo cp /etc/letsencrypt/live/<NAMA_SUBDOMAIN_ANDA>.duckdns.org/privkey.pem /home/<NAMA_PENGGUNA_ANDA>/n8n/n8n_certs/privkey.pem
# Atur izin
sudo chown -R 1000:1000 /home/<NAMA_PENGGUNA_ANDA>/n8n/n8n_certs # Agar user node bisa baca
sudo chmod 644 /home/<NAMA_PENGGUNA_ANDA>/n8n/n8n_certs/fullchain.pem
sudo chmod 600 /home/<NAMA_PENGGUNA_ANDA>/n8n/n8n_certs/privkey.pem # Kunci privat lebih ketat

```

Ganti `<NAMA_PENGGUNA_ANDA>` dan `<NAMA_SUBDOMAIN_ANDA>.duckdns.org` dengan nilai yang sesuai.

## **6\. Konfigurasi `docker-compose.yml` untuk n8n**

Buat file `docker-compose.yml` di `/home/<NAMA_PENGGUNA_ANDA>/n8n/docker-compose.yml` dengan konten berikut:

```
services:
  n8n:
    image: n8nio/n8n # Untuk versi stabil terbaru, atau n8nio/n8n:<versi_spesifik>
    restart: always
    ports:
      # Petakan port 443 di host ke port 443 di container untuk HTTPS standar
      - "443:443"
    environment:
      - N8N_HOST=<NAMA_SUBDOMAIN_ANDA>.duckdns.org
      - N8N_PORT=443 # n8n akan mendengarkan di port 443 di dalam container
      - N8N_PROTOCOL=https
      # Path di dalam kontainer setelah di-mount dari direktori kustom
      - N8N_SSL_CERT=/sertifikat_n8n/fullchain.pem
      - N8N_SSL_KEY=/sertifikat_n8n/privkey.pem
      - WEBHOOK_URL=https://<NAMA_SUBDOMAIN_ANDA>.duckdns.org/
      - GENERIC_TIMEZONE=Asia/Jakarta # Zona waktu Anda
      - N8N_RUNNERS_ENABLED=true # Direkomendasikan
      # Opsional: untuk memperbaiki peringatan izin file config
      # - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
    volumes:
      # Volume untuk data n8n (workflow, credentials, dll.)
      - /home/<NAMA_PENGGUNA_ANDA>/n8n/n8n_data:/home/node/.n8n
      # Mount direktori sertifikat kustom Anda dari host ke /sertifikat_n8n di kontainer
      - /home/<NAMA_PENGGUNA_ANDA>/n8n/n8n_certs:/sertifikat_n8n:ro
      # Opsional: Mount seluruh direktori Let's Encrypt jika Anda ingin merujuk langsung ke sana
      # dan menangani izin dengan cara lain. Namun, konfigurasi di atas menggunakan salinan.
      # - /etc/letsencrypt:/etc/letsencrypt:ro

```

Ganti `<NAMA_PENGGUNA_ANDA>` dan `<NAMA_SUBDOMAIN_ANDA>.duckdns.org` dengan nilai yang sesuai. **Catatan:** Baris `version: '3.7'` di awal file `docker-compose.yml` sudah usang (obsolete) dan dapat dihapus jika menggunakan Docker Compose v2+.

## **7\. Menjalankan n8n**

1. Pastikan Anda berada di direktori yang berisi file `docker-compose.yml` Anda:

```
cd /home/<NAMA_PENGGUNA_ANDA>/n8n
```

2. Jalankan n8n:

```
docker compose up -d
```

3. **Verifikasi:**  
   * Akses UI n8n di `https://<NAMA_SUBDOMAIN_ANDA>.duckdns.org`. Seharusnya tidak ada peringatan SSL.  
   * Periksa status kontainer: `docker compose ps`  
   * Periksa log jika ada masalah: `docker compose logs -f n8n`

## **8\. Mengelola n8n**

Semua perintah `docker compose` dijalankan dari direktori yang berisi file `docker-compose.yml`.

* **Melihat Log:**

```
docker compose logs n8n
docker compose logs -f n8n # Untuk mengikuti log secara real-time
```

* **Menghentikan n8n:**

```
docker compose down
```

* (Ini akan menghentikan dan menghapus kontainer, tetapi volume data tetap aman).  
* **Memulai n8n:**

```
docker compose up -d
```

* **Merestart n8n:**

```
docker compose restart n8n
```

## **9\. Jalankan n8n di Browser**

1. Buka alamat yang dimasukkan pada konfigurasi docker-compose.yml  
2. Setup n8n untuk pertama kali dengan memasukkan beberapa detail

## **10\. Melakukan upgrade n8n**

1. **Backup Data (Sangat Direkomendasikan):**

```
sudo cp -Rp /home/<NAMA_PENGGUNA_ANDA>/n8n/n8n_data /home/<NAMA_PENGGUNA_ANDA>/n8n/n8n_data_backup_$(date +%Y%m%d%H%M%S)
```

2. **Navigasi ke Direktori `docker-compose.yml`:**

```
cd /home/<NAMA_PENGGUNA_ANDA>/n8n
```

3. **Tarik Image Terbaru (jika menggunakan tag `latest` atau versi baru):** Jika Anda ingin menggunakan versi spesifik, ubah tag `image` di `docker-compose.yml` (misalnya, `image: n8nio/n8n:1.95.3`).

```
docker compose pull n8n # Menarik tag yang didefinisikan di docker-compose.yml
```

4. **Hentikan** Kontainer **Saat Ini:**

```
docker compose down
```

5. **Mulai n8n dengan Image Baru:**

```
docker compose up -d
```

6. Docker Compose akan menggunakan image yang baru ditarik/didefinisikan dan melampirkan kembali volume data Anda.

## **10\. Pembaruan Sertifikat SSL Otomatis & Penerapan ke n8n**

Sertifikat Let's Encrypt berlaku 90 hari. `certbot` biasanya mengatur pembaruan otomatis.

**Karena Anda Menyalin Sertifikat ke Direktori Kustom (seperti dalam konfigurasi terakhir):** Pembaruan otomatis `certbot` akan memperbarui sertifikat di `/etc/letsencrypt/live/`, tetapi **tidak** pada salinan Anda di `/home/<NAMA_PENGGUNA_ANDA>/n8n/n8n_certs/`. Anda perlu mengatur ini secara manual atau dengan skrip.

**Solusi:** Gunakan *deploy hook* `certbot`. Buat skrip (misalnya, `/opt/scripts/renew_n8n_certs.sh`) yang akan:

1. Menyalin sertifikat yang baru diperbarui.  
2. Merestart kontainer n8n.

Contoh isi `/opt/scripts/renew_n8n_certs.sh`:

```
#!/bin/bash

# Path ke direktori docker-compose.yml n8n Anda
N8N_COMPOSE_DIR="/home/<NAMA_PENGGUNA_ANDA>/n8n"
# Domain Anda
DOMAIN="<NAMA_SUBDOMAIN_ANDA>.duckdns.org"
# Direktori tujuan salinan sertifikat Anda
CERT_COPY_DIR="/home/<NAMA_PENGGUNA_ANDA>/n8n/n8n_certs"

echo "Memulai proses pembaruan sertifikat n8n untuk ${DOMAIN}..."

# Salin file sertifikat
echo "Menyalin sertifikat baru..."
sudo cp "/etc/letsencrypt/live/${DOMAIN}/fullchain.pem" "${CERT_COPY_DIR}/fullchain.pem"
sudo cp "/etc/letsencrypt/live/${DOMAIN}/privkey.pem" "${CERT_COPY_DIR}/privkey.pem"

# Sesuaikan izin lagi
echo "Menyesuaikan izin sertifikat..."
sudo chown 1000:1000 "${CERT_COPY_DIR}/fullchain.pem" "${CERT_COPY_DIR}/privkey.pem"
sudo chmod 644 "${CERT_COPY_DIR}/fullchain.pem"
sudo chmod 600 "${CERT_COPY_DIR}/privkey.pem"

# Restart n8n menggunakan docker compose
echo "Merestart layanan n8n..."
if cd "${N8N_COMPOSE_DIR}" && docker compose restart n8n; then
  echo "Sertifikat n8n untuk ${DOMAIN} telah berhasil diperbarui dan layanan n8n direstart."
else
  echo "GAGAL merestart layanan n8n. Periksa log docker compose."
fi

exit 0
```

Ganti `<NAMA_PENGGUNA_ANDA>` dan `<NAMA_SUBDOMAIN_ANDA>.duckdns.org` dengan nilai yang sesuai. Buat skrip ini executable: `sudo chmod +x /opt/scripts/renew_n8n_certs.sh`.

Kemudian, Anda bisa mengedit file konfigurasi pembaruan Certbot untuk domain Anda. File ini biasanya ada di `/etc/letsencrypt/renewal/<NAMA_SUBDOMAIN_ANDA>.duckdns.org.conf`. Tambahkan atau modifikasi baris `renew_hook` di bawah bagian `[renewalparams]`:

```
# Tambahkan atau modifikasi baris ini di dalam file .conf yang sesuai
renew_hook = /opt/scripts/renew_n8n_certs.sh
```

Setelah itu, setiap kali `certbot renew` berhasil memperbarui sertifikat untuk domain tersebut, skrip `renew_n8n_certs.sh` akan dijalankan secara otomatis.

Anda bisa menguji pembaruan dan hook dengan:

```
sudo certbot renew --force-renewal
```

Perintah `--force-renewal` akan memaksa pembaruan meskipun sertifikat belum akan kedaluwarsa, berguna untuk pengujian.

## **11\. Troubleshooting Umum (Ringkasan)**

* **Error `ENOENT: no such file or directory` untuk file sertifikat:**  
  * Pastikan path di `N8N_SSL_CERT` dan `N8N_SSL_KEY` benar merujuk ke lokasi *di dalam kontainer*.  
  * Pastikan volume mount sudah benar memetakan direktori host (yang berisi sertifikat) ke direktori di dalam kontainer yang dirujuk oleh variabel environment.  
  * Pastikan file sertifikat benar-benar ada di direktori host yang di-mount.  
* **Error `EACCES: permission denied` untuk file sertifikat atau direktori data:**  
  * Untuk direktori data (`/home/<NAMA_PENGGUNA_ANDA>/n8n/n8n_data` di host): Pastikan dimiliki oleh UID `1000` (`sudo chown -R 1000:1000 /home/<NAMA_PENGGUNA_ANDA>/n8n/n8n_data`).  
  * Untuk direktori sertifikat yang di-mount (misalnya `/home/<NAMA_PENGGUNA_ANDA>/n8n/n8n_certs` di host): Pastikan pengguna `node` (UID 1000\) di dalam kontainer dapat membacanya. Jika di-mount read-only (`:ro`), izin baca di host untuk `others` atau kepemilikan oleh UID 1000 sudah cukup.  
* **Tidak bisa akses n8n di browser:**  
  * Periksa log n8n (`docker compose logs n8n`).  
  * Pastikan kontainer n8n berjalan (`docker compose ps`).  
  * Pastikan firewall GCE dan `ufw` (jika aktif) mengizinkan port 443 (atau port kustom yang Anda gunakan).  
  * Pastikan DNS untuk domain Anda mengarah ke IP yang benar.  
* **Webhook tidak berfungsi:**  
  * Pastikan `WEBHOOK_URL` di `docker-compose.yml` sudah benar dan menggunakan HTTPS dengan domain dan port yang benar (misalnya, `https://<NAMA_SUBDOMAIN_ANDA>.duckdns.org/`).  
  * Jika menggunakan sertifikat Let's Encrypt, layanan eksternal (seperti Telegram) seharusnya sudah mempercayainya. Tidak perlu lagi mengirim parameter `certificate` saat mendaftarkan webhook ke Telegram.  
  * Pastikan path webhook yang Anda daftarkan ke layanan eksternal (misalnya, `/rest/webhook/XYZ`) sudah benar dan workflow n8n yang sesuai aktif.  
  * Periksa log n8n saat webhook seharusnya diterima untuk melihat apakah ada error atau aktivitas.  
  * Pastikan tidak ada masalah jaringan atau firewall yang memblokir koneksi masuk dari server layanan eksternal ke port n8n Anda (biasanya 443).

