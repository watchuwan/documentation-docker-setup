## Panduan Setup Proyek CRUD Laravel-Vue dengan Docker
Panduan ini akan memandu Anda dalam menyiapkan proyek Laravel & Vue menggunakan Docker Compose untuk lingkungan pengembangan dan produksi.

Langkah Awal: Membuat Proyek Laravel
Jika Anda belum memiliki proyek Laravel, Anda dapat membuatnya menggunakan Docker tanpa harus menginstal Composer di sistem lokal Anda.

Buka terminal di direktori tempat Anda ingin membuat proyek baru dan jalankan perintah berikut:

```bash
docker run --rm -v "$(pwd)":/app composer create-project laravel/laravel crud-vue
```

Perintah ini akan membuat folder crud-vue yang berisi instalasi Laravel baru. Setelah itu, masuk ke direktori crud-vue untuk melanjutkan ke langkah berikutnya.

Perintah ini akan membuat folder crud-vue yang berisi instalasi Laravel baru. Setelah itu, masuk ke direktori crud-vue untuk melanjutkan ke langkah berikutnya.

1. Struktur File Proyek
Setelah proyek dibuat, pastikan Anda membuat struktur direktori dan file berikut di root proyek Anda (crud-vue):

```.
├── docker-compose.yml          # Konfigurasi dasar untuk semua service
├── Dockerfile.dev              # Dockerfile khusus untuk pengembangan
├── Dockerfile.prod             # Dockerfile khusus untuk produksi
└── docker/
    └── nginx/
        └── default.conf        # Konfigurasi Nginx
```


2. File Konfigurasi Dasar
Berikut adalah isi dari file-file konfigurasi utama Anda. Salin dan letakkan di lokasi yang benar.

```docker-compose.yml```
File ini adalah konfigurasi dasar untuk semua layanan. Ia menggunakan ``Dockerfile.dev`` secara default.


```yaml

services:
  # PHP-FPM Service (Aplikasi Laravel Anda)
  app:
    build:
      context: .
      dockerfile: Dockerfile.dev
    image: crud-vue-app
    volumes:
      - .:/var/www/html
    ports:
      - "5173:5173"
    environment:
      APP_NAME: crud-vue-app
      APP_ENV: local
      APP_DEBUG: "true"
      APP_URL: http://localhost

      DB_CONNECTION: pgsql
      DB_HOST: pgsql_db
      DB_PORT: 5432
      DB_DATABASE: ${DB_DATABASE:-crud_vue_db}
      DB_USERNAME: ${DB_USERNAME:-postgres}
      DB_PASSWORD: ${DB_PASSWORD:-postgres}

      REDIS_HOST: redis
      REDIS_PORT: 6379

    depends_on:
      - pgsql_db
      - redis

  # Nginx Web Server Service
  nginx:
    image: nginx:stable-alpine
    ports:
      - "80:80"
    volumes:
      - .:/var/www/html
      - ./docker/nginx/default.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - app

  # PostgreSQL Database Service
  pgsql_db:
    image: postgres:17.5-alpine
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: ${DB_DATABASE:-crud_vue_db}
      POSTGRES_USER: ${DB_USERNAME:-postgres}
      POSTGRES_PASSWORD: ${DB_PASSWORD:-postgres}
    volumes:
      - crud-vue_pgsql_data:/var/lib/postgresql/data

  # Redis Cache/Queue Service
  redis:
    image: redis:alpine
    ports:
      - "6379:6379"
    volumes:
      - crud-vue_redis_data:/data

# Definisi Docker Volumes untuk data persisten
volumes:
  crud-vue_pgsql_data:
  crud-vue_redis_data:
  ```

``Dockerfile.dev``
Gunakan file ini untuk lingkungan pengembangan. Image ini berisi Node.js dan NPM sehingga Anda bisa menjalankan ```npm run dev``` di dalamnya.

# Dockerfile.dev (Gunakan untuk pengembangan)

```bash
FROM php:8.3-fpm-alpine

# Instal dependensi sistem, termasuk nodejs dan npm
RUN apk add --no-cache \
    git \
    curl \
    libzip-dev \
    libpng-dev \
    libjpeg-turbo-dev \
    freetype-dev \
    icu-dev \
    postgresql-dev \
    unzip \
    nodejs \
    npm

# Instal ekstensi PHP yang dibutuhkan
RUN docker-php-ext-install pdo_pgsql
RUN docker-php-ext-install pdo_mysql zip exif pcntl gd intl opcache
RUN docker-php-ext-configure gd --with-freetype --with-jpeg

# Bersihkan cache
RUN rm -rf /var/cache/apk/*

# Instal Composer secara global
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

WORKDIR /var/www/html

# Salin semua kode aplikasi
COPY . .

# Beri hak akses yang benar pada direktori
RUN chown -R www-data:www-data /var/www/html/storage \
    /var/www/html/bootstrap/cache

EXPOSE 9000

CMD ["php-fpm"]
```


``Dockerfile.prod``
Gunakan file ini untuk lingkungan produksi. Ini menggunakan multi-stage build untuk menghasilkan image yang lebih kecil dan lebih aman karena tidak menyertakan alat pengembangan.

# Dockerfile.prod (Gunakan untuk produksi)
```dockerfile
# --- Stage 1: Build Frontend dan Backend ---
FROM node:22-alpine AS node_builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm install
COPY . .
RUN npm run build

FROM php:8.3-fpm-alpine AS composer_builder
WORKDIR /app
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer
COPY composer.json composer.lock ./
RUN composer install --no-dev --optimize-autoloader

# --- Stage 2: Final Runtime Image ---
FROM php:8.3-fpm-alpine AS final

# Instal dependensi sistem yang diperlukan (versi minimal)
RUN apk add --no-cache \
    libzip-dev \
    libpng-dev \
    libjpeg-turbo-dev \
    freetype-dev \
    icu-dev \
    postgresql-dev \
    nzip \
    nodejs \
    npm       

# Instal ekstensi PHP yang dibutuhkan (versi minimal)
RUN docker-php-ext-install pdo_pgsql
RUN docker-php-ext-install pdo_mysql zip exif pcntl gd intl opcache
RUN docker-php-ext-configure gd --with-freetype --with-jpeg

# Bersihkan cache
RUN rm -rf /var/cache/apk/*

WORKDIR /var/www/html

# Salin dependensi Composer dari stage `composer_builder`
COPY --from=composer_builder /app/vendor /var/www/html/vendor

# Salin hasil build frontend dari stage `node_builder`
COPY --from=node_builder /app/public/build /var/www/html/public/build

# Salin sisa kode aplikasi Laravel (file-file lain)
COPY --chown=www-data:www-data . .

# Hapus file yang tidak diperlukan di produksi
RUN rm -f /var/www/html/composer.json \
    /var/www/html/composer.lock \
    /var/www/html/package.json \
    /var/www/html/package-lock.json \
    /var/www/html/README.md

# Set hak akses yang benar
RUN chown -R www-data:www-data /var/www/html/storage \
    /var/www/html/bootstrap/cache

EXPOSE 9000

CMD ["php-fpm"]
```
docker/nginx/default.conf
Konfigurasi Nginx ini sama untuk kedua lingkungan, karena ia menangani permintaan ke PHP-FPM dan juga proxy ke Vite.
```bash
server {
    listen 80;
    server_name localhost;
    root /var/www/html/public;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-XSS-Protection "1; mode=block";
    add_header X-Content-Type-Options "nosniff";

    index index.html index.php;

    charset utf-8;

    location /@vite/ {
        proxy_pass http://app:5173;
    }

    location /resources/ {
        proxy_pass http://app:5173;
    }

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    error_page 404 /index.php;

    location ~ \.php$ {
        fastcgi_pass app:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.ht {
        deny all;
    }
}
```
3. Alur Kerja Pengembangan (Development Workflow)
Lingkungan pengembangan dirancang untuk fleksibilitas dan live-reloading.

Memulai Lingkungan Pengembangan
Jalankan perintah ini di root proyek Anda untuk membangun dan menjalankan semua service:

```bash 
docker-compose up -d --build
```

Bekerja dengan Proyek
Setelah kontainer berjalan, Anda harus menjalankan perintah-perintah setup awal. Gunakan terminal baru untuk masuk ke dalam kontainer app.

```bash
docker-compose exec app sh
```
Di dalam kontainer, jalankan perintah berikut:

Instal Dependensi dan Laravel Starter Kit Vue:
Langkah pertama adalah menginstal semua dependensi PHP dan Node.js, termasuk menginstal Laravel Breeze untuk Vue.
```bash
composer install
composer require laravel/breeze --dev
php artisan breeze:install vue
npm install
```
composer require laravel/breeze --dev: Menginstal paket Laravel Breeze.

php artisan breeze:install vue: Menjalankan perintah Breeze untuk membuat scaffolding (struktur dasar) untuk Vue, termasuk file-file otentikasi.

npm install: Menginstal dependensi baru yang ditambahkan oleh Breeze.

Buat Kunci Aplikasi (Application Key):
Jalankan perintah ini untuk membuat kunci unik yang penting untuk keamanan Laravel.
```bash
php artisan key:generate
```
Jalankan Migrasi Database:
```bash
php artisan migrate
```
Menjalankan Vite Development Server:
Di terminal lain, masuk ke container app lagi dan jalankan:
```bash
npm run dev
```
Perubahan pada kode Vue Anda akan otomatis ter-reload di http://localhost.

4. Alur Kerja Produksi (Production Workflow)
Lingkungan produksi dirancang untuk efisiensi dan keamanan. Image akan jauh lebih kecil karena tidak menyertakan alat-alat pengembangan.

Membangun dan Menjalankan untuk Produksi
Anda harus mengedit file docker-compose.yml untuk menggunakan Dockerfile.prod sebelum menjalankan perintah.

Langkah 1: Di file docker-compose.yml, ubah baris dockerfile: Dockerfile.dev menjadi dockerfile: Dockerfile.prod.
```yaml
# ...
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.prod # <-- Ganti di sini
# ...
```
Langkah 2: Bangun dan jalankan container. Proses ini akan menjalankan npm run build dan composer install --no-dev di dalam Dockerfile, jadi tidak ada langkah manual yang diperlukan.
```bash
docker-compose up -d --build
```
Setelah proses selesai, aplikasi Anda akan siap diakses di http://localhost dengan semua aset frontend yang sudah di-build.