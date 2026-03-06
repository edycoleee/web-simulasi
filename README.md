# Absensi Digital — absen.sulfat.site

Frontend absensi berbasis web statis yang di-serve lewat Nginx container, dilindungi reverse proxy, dan siap diakses oleh klien Android maupun browser desktop.

---

## Daftar Isi

1. [Gambaran Sistem](#gambaran-sistem)
2. [Struktur Proyek](#struktur-proyek)
3. [Halaman HTML — Simulasi Absensi](#halaman-html--simulasi-absensi)
4. [Pengamanan Nginx](#pengamanan-nginx)
5. [Menjalankan Proyek](#menjalankan-proyek)
6. [Konfigurasi Reverse Proxy (192.10.10.15)](#konfigurasi-reverse-proxy-192101015)
7. [Checklist Keamanan & Performa](#checklist-keamanan--performa)

---

## Gambaran Sistem

```
Android / Browser
       │  HTTPS
       ▼
┌─────────────────────┐
│  Nginx Reverse Proxy │  192.10.10.15  (SSL Termination, HTTPS)
│  absen.sulfat.site  │
└────────┬────────────┘
         │  HTTP (LAN Internal)
         ▼
┌─────────────────────┐
│  Container Nginx    │  host:8080 → container:80
│  (absen_sulfat)     │  Static HTML + Security Headers
└─────────────────────┘
```

- **SSL/TLS** selesai di reverse proxy. Container backend cukup HTTP biasa.
- **IP asli klien** diteruskan lewat header `X-Real-IP` dari reverse proxy ke container.
- **CORS** dikelola di sisi aplikasi backend, bukan di Nginx container, agar tidak ada double-header yang menyebabkan error pada Android.

---

## Struktur Proyek

```
coba-html/
├── docker-compose.yml        # Definisi service container
├── nginx/
│   └── default.conf          # Konfigurasi Nginx container (keamanan + performa)
├── html/
│   └── index.html            # Halaman utama simulasi absensi
├── logs/
│   ├── access.log            # Log akses (IP asli klien, bukan IP proxy)
│   └── error.log             # Log error Nginx
└── README.md
```

---

## Halaman HTML — Simulasi Absensi

### Fitur

| Fitur | Keterangan |
|---|---|
| Form Absensi | Input nama, NIP/NIM, divisi/kelas |
| Status Kehadiran | Tombol toggle: **Hadir**, **Sakit**, **Izin** dengan warna berbeda |
| Validasi Client-side | Cek field wajib sebelum submit, notifikasi jika tidak lengkap |
| Toast Feedback | Konfirmasi visual setelah absensi berhasil dikirim |
| Desain Responsif | Glassmorphism dark-mode, nyaman di mobile maupun desktop |
| Tanpa Dependensi Eksternal | Murni HTML + CSS + Vanilla JS, tidak butuh koneksi CDN |

### Tampilan

Halaman menggunakan desain **dark glassmorphism** dengan palet warna:
- Biru terang `#53c0f0` — aksen utama, input aktif, tombol submit
- Hijau `#2ed573` — status Hadir
- Oranye — status Sakit
- Abu gelap `rgba(255,255,255,0.05)` — background card

### Alur Form

```
Isi Nama + NIP + Divisi
        │
        ▼
Pilih Status (Hadir / Sakit / Izin)
        │
        ▼
[Kirim Absensi]
        │
        ├── Data tidak lengkap → alert()
        └── Data lengkap → tampilkan toast sukses, reset form
```

> **Catatan:** Halaman ini adalah **simulasi frontend** — data belum dikirim ke API/database. Untuk integrasi backend, ganti event handler `submit` di `index.html` dengan `fetch()` ke endpoint API.

---

## Pengamanan Nginx

Konfigurasi lengkap ada di [`nginx/default.conf`](nginx/default.conf).

### 1. Trusted Proxy — Real IP dari Header

```nginx
set_real_ip_from  192.10.10.15;
real_ip_header    X-Real-IP;
real_ip_recursive on;
```

**Masalah yang diselesaikan:**
Tanpa konfigurasi ini, `$remote_addr` selalu berisi `192.10.10.15` (IP proxy), bukan IP asli klien Android. Log menjadi tidak informatif dan fitur rate-limit berbasis IP tidak berfungsi.

**Cara kerjanya:**
Modul `ngx_http_realip_module` (sudah built-in di nginx:alpine) membaca nilai header `X-Real-IP` yang dikirim reverse proxy, lalu mengganti nilai `$remote_addr` dengan IP asli klien. Seluruh log, kondisi `if`, dan `limit_req_zone` otomatis menggunakan IP yang benar.

**`real_ip_recursive on`** — diperlukan jika ada chain proxy lebih dari satu lapis (proxy → proxy → container). Nginx akan menelusuri seluruh chain dan mengambil IP terluar yang bukan alamat tepercaya.

---

### 2. CORS — Tidak di Nginx Container

```nginx
# CORS TIDAK diset di sini — ditangani di aplikasi backend
# agar tidak terjadi double-header yang menyebabkan error di Android.
```

**Mengapa penting?**
Jika CORS diset *baik* di Nginx *maupun* di kode aplikasi backend, response akan mengandung dua header `Access-Control-Allow-Origin`. Browser dan klien Android (OkHttp, Retrofit) akan menolak response tersebut sesuai spesifikasi Fetch/CORS.

**Aturan:**
- Aplikasi backend (Node.js, Laravel, FastAPI, dll.) yang bertanggung jawab penuh atas header CORS.
- Nginx container **tidak** boleh menambahkan `add_header Access-Control-*` apapun.

Contoh CORS yang benar di sisi aplikasi (Node.js/Express):

```js
app.use(cors({
  origin: 'https://absen.sulfat.site',
  methods: ['GET', 'POST', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'Authorization'],
}));
```

---

### 3. Tidak Ada SSL di Backend

```nginx
# Container hanya listen 80 (HTTP)
listen 80;
```

**Arsitektur SSL Termination:**

```
Klien  ──HTTPS──►  Nginx 192.10.10.15  ──HTTP──►  Container :80
                   (SSL terminate di sini)         (LAN internal)
```

Traffic antara reverse proxy dan container berjalan di jaringan LAN internal. Mengenkripsi ulang traffic ini (end-to-end internal) hanya menambah overhead CPU tanpa manfaat keamanan yang signifikan selama:
1. Jaringan LAN internal dikontrol dan terisolasi
2. Container tidak dapat diakses langsung dari publik

Untuk meningkatkan isolasi lebih lanjut, gunakan Docker network internal:

```yaml
# docker-compose.yml
networks:
  internal:
    internal: true  # tidak bisa diakses dari luar host
```

---

### 4. Security Headers

```nginx
add_header X-Frame-Options        "SAMEORIGIN"    always;
add_header X-Content-Type-Options "nosniff"       always;
add_header X-XSS-Protection       "1; mode=block" always;
```

| Header | Fungsi |
|---|---|
| `X-Frame-Options: SAMEORIGIN` | Mencegah halaman di-embed di `<iframe>` dari domain lain (anti-clickjacking) |
| `X-Content-Type-Options: nosniff` | Mencegah browser menebak tipe konten file, menghindari MIME-sniffing attack |
| `X-XSS-Protection: 1; mode=block` | Mengaktifkan filter XSS bawaan browser lama; browser modern sudah menggunakan CSP |

**Rekomendasi tambahan** untuk produksi:

```nginx
add_header Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline';" always;
add_header Referrer-Policy         "strict-origin-when-cross-origin" always;
add_header Permissions-Policy      "geolocation=(), microphone=(), camera=()" always;
```

---

### 5. Cache Static Assets

```nginx
location ~* \.(css|js|png|jpg|jpeg|gif|ico|svg|woff2?)$ {
    expires 30d;
    add_header Cache-Control "public, no-transform";
}
```

File statis di-cache browser selama **30 hari**. Permintaan berulang tidak menyentuh server, loading halaman menjadi instan dari kunjungan kedua.

---

### 6. Custom Log Format

```nginx
log_format main_real '$remote_addr [$time_local] "$request" '
                     '$status $body_bytes_sent "$http_referer" '
                     '"$http_user_agent" real_ip=$http_x_real_ip';
```

Log mencatat dua nilai:
- `$remote_addr` — IP asli klien (sudah disubstitusi oleh `real_ip` module)
- `real_ip=$http_x_real_ip` — nilai mentah dari header `X-Real-IP` untuk keperluan audit/debugging

---

## Menjalankan Proyek

### Prasyarat

- Docker Engine ≥ 24
- Docker Compose V2

### Jalankan

```bash
docker compose up -d
```

Container akan berjalan di `http://localhost:8080`.

### Perintah Berguna

```bash
# Cek status container
docker compose ps

# Test konfigurasi nginx (tanpa restart)
docker compose exec absen-web nginx -t

# Reload konfigurasi nginx (tanpa downtime)
docker compose exec absen-web nginx -s reload

# Lihat log akses real-time
docker compose logs -f absen-web

# Atau langsung dari file log
tail -f logs/access.log

# Stop
docker compose down
```

---

## Konfigurasi Reverse Proxy (192.10.10.15)

Tambahkan server block berikut di Nginx reverse proxy:

```nginx
server {
    listen 80;
    server_name absen.sulfat.site;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name absen.sulfat.site;

    ssl_certificate     /etc/letsencrypt/live/absen.sulfat.site/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/absen.sulfat.site/privkey.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    location / {
        proxy_pass         http://<IP_SERVER_BACKEND>:8080;
        proxy_http_version 1.1;
        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;       # ← IP asli klien
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_set_header   Connection        "";
    }
}
```

> Ganti `<IP_SERVER_BACKEND>` dengan IP server tempat Docker berjalan. Jika reverse proxy dan Docker berada di server yang sama, gunakan `127.0.0.1`.

---

## Checklist Keamanan & Performa

### Keamanan

- [x] Trusted proxy: `set_real_ip_from` + `real_ip_header`
- [x] Real IP tercatat di log, bukan IP proxy
- [x] CORS dikelola di aplikasi backend (tidak double-header)
- [x] SSL terminate di reverse proxy, backend HTTP-only di LAN internal
- [x] `X-Frame-Options` — anti-clickjacking
- [x] `X-Content-Type-Options` — anti MIME sniffing
- [x] `X-XSS-Protection` — XSS filter browser lama
- [ ] Content Security Policy (CSP) — *disarankan untuk produksi*
- [ ] Rate limiting (`limit_req_zone`) — *disarankan untuk API endpoint*
- [ ] Firewall: pastikan port 8080 hanya bisa diakses dari `192.10.10.15`

### Performa

- [x] Nginx alpine image (ringan, ~20MB)
- [x] Static file serving (tidak ada PHP/Node overhead)
- [x] Cache-Control 30 hari untuk asset statis
- [x] `http2` di reverse proxy (perlu diaktifkan)
- [ ] Gzip/Brotli compression — *disarankan untuk produksi*

```nginx
# Tambahkan di nginx/default.conf untuk kompresi
gzip on;
gzip_types text/html text/css application/javascript application/json image/svg+xml;
gzip_min_length 1024;
gzip_vary on;
```
