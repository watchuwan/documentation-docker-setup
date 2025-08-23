## Panduan Setup Proyek Laravel Filament 4 di Docker (macOS)

Panduan ini akan menjelaskan cara step by step

Langkah Awal: Membuat Proyek Laravel
Jika Anda belum memiliki proyek Laravel, Anda dapat membuatnya menggunakan Docker tanpa harus menginstal Composer di sistem lokal Anda.

Buka terminal di direktori tempat Anda ingin membuat proyek baru dan jalankan perintah berikut:

```bash
docker run --rm -v "$(pwd)":/app composer create-project laravel/laravel nama-proyek
```

Perintah ini akan membuat folder proyek-laravel-filament yang berisi instalasi Laravel baru. Setelah itu, masuk ke direktori proyek-laravel-filament untuk melanjutkan ke langkah berikutnya.

1. Struktur File Proyek
Setelah proyek dibuat, pastikan Anda membuat struktur direktori dan file berikut di root proyek Anda:

```.
├── docker-compose.yml          # Konfigurasi dasar untuk semua service
├── Dockerfile              # Dockerfile khusus untuk pengembangan
└── docker/
    └── nginx/
        └── default.conf        # Konfigurasi Nginx
```

2. File Konfigurasi Dasar
Berikut adalah isi dari file-file konfigurasi utama Anda. Salin dan letakkan di lokasi yang benar.

```docker-compose.yml```
File ini adalah konfigurasi dasar untuk semua layanan.

```yaml
services:
  # PHP-FPM Service (Aplikasi Laravel Anda)
  app:
    build:
      context: .
      dockerfile: Dockerfile
    image: crudfilament-app # Nama image unik untuk proyek ini
    volumes:
      - .:/var/www/html
    ports:
       - "5173:5173"
    environment:
      # Variabel lingkungan Laravel
      APP_NAME: crudfilament-app
      APP_ENV: local
      APP_DEBUG: "true"
      APP_URL: http://localhost

      # Konfigurasi Database PostgreSQL (PASTIKAN INI SAMA DENGAN pgsql_db)
      DB_CONNECTION: pgsql
      DB_HOST: pgsql_db
      DB_PORT: 5432
      DB_DATABASE: ${DB_DATABASE:-cms_db} # Nama database Anda
      DB_USERNAME: ${DB_USERNAME:-postgres} # User database Anda
      DB_PASSWORD: ${DB_PASSWORD:-postgres} # Password database Anda

      # Konfigurasi Redis
      REDIS_HOST: redis
      REDIS_PORT: 6379

    depends_on:
      - pgsql_db
      - redis

  # Nginx Web Server Service
  nginx:
    image: nginx:stable-alpine
    ports:
      - "80:80" # Map host port 80 ke container port 80 (akses di http://localhost)
    volumes:
      - .:/var/www/html
      - ./docker/nginx/default.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - app

  # PostgreSQL Database Service
  pgsql_db:
    image: postgres:17.5-alpine # PostgreSQL 17.5 stabil
    ports:
      - "5432:5432" # Map host port 5432 ke container port 5432 (untuk DBeaver, dll.)
    environment:
      # PASTIKAN INI SAMA DENGAN YANG DI SERVICE 'app'
      POSTGRES_DB: ${DB_DATABASE:-cms_db}
      POSTGRES_USER: ${DB_USERNAME:-postgres}
      POSTGRES_PASSWORD: ${DB_PASSWORD:-postgres}
    volumes:
      - crudfilament_pgsql_data:/var/lib/postgresql/data # Volume data persisten unik

  # Redis Cache/Queue Service
  redis:
    image: redis:alpine
    ports:
      - "6379:6379" # Map host port 6379 ke container port 6379 (untuk aplikasi)
    volumes:
      - crudfilament_redis_data:/data # Volume data persisten unik

# Definisi Docker Volumes untuk data persisten
volumes:
  crudfilament_pgsql_data: # Volume unik untuk PostgreSQL
  crudfilament_redis_data: # Volume unik untuk Redis
```
``Dockerfile``
Gunakan file ini untuk lingkungan pengembangan. Image ini berisi Node.js dan NPM sehingga Anda bisa menjalankan ```npm run dev``` di dalamnya.


```bash
FROM php:8.3-fpm-alpine AS base

# --- Stage 1: Build PHP-FPM application (including Node.js for frontend) ---
# Menggunakan base PHP image yang sama untuk kesederhanaan dan memastikan semua tooling ada di satu tempat
FROM base

# Instal dependensi sistem yang diperlukan oleh Laravel dan PHP
# Tambahkan nodejs dan npm di sini secara eksplisit
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

# Bersihkan cache APK untuk mengurangi ukuran image akhir
RUN rm -rf /var/cache/apk/*

# Instal Composer secara global di dalam container
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Atur direktori kerja ke root aplikasi Laravel
WORKDIR /var/www/html

# Salin file composer.json dan composer.lock (untuk instal dependensi PHP)
COPY composer.json composer.lock ./

# Instal dependensi Composer
RUN composer install --no-scripts --prefer-dist --optimize-autoloader

# Salin package.json dan package-lock.json jika ada
COPY package*.json ./

# **PENTING**: Instal dependensi Node.js di dalam container.
# Ini akan membuat folder node_modules dengan Tailwind, Vite, dll.
RUN npm install

# Salin sisa kode aplikasi Laravel Anda ke dalam container
COPY . .

# Jalankan composer dump-autoload (memastikan autoloader diperbarui setelah menyalin kode)
RUN composer dump-autoload --optimize

# Expose port 9000 untuk PHP-FPM
EXPOSE 9000

# Perintah default untuk menjalankan PHP-FPM saat container dimulai
CMD ["php-fpm"]
```

``docker/nginx/default.conf``
Konfigurasi Nginx ini sama untuk kedua lingkungan, karena ia menangani permintaan ke PHP-FPM dan juga proxy ke Vite.


```sh
server {
    listen 80;
    server_name localhost;
    root /var/www/html/public; # Arahkan ke direktori public Laravel

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-XSS-Protection "1; mode=block";
    add_header X-Content-Type-Options "nosniff";

    index index.html index.htm index.php;

    charset utf-8;

    location /@vite/ {
        proxy_pass http://app:5173;
    }

    # Aturan proxy untuk aset Vite.
    # Misalnya, http://localhost/resources/js/app.js
    location /resources/ {
        proxy_pass http://app:5173;
    }

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass app:9000; # Meneruskan permintaan PHP ke service 'app' (PHP-FPM)
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

``vite.config.js``
Konfigurasi vite ini untuk memastikan vite dapat diakses

```bash
import { defineConfig } from "vite";
import laravel from "laravel-vite-plugin";
import tailwindcss from "@tailwindcss/vite";

export default defineConfig({
    plugins: [
        laravel({
            input: ["resources/css/app.css", "resources/js/app.js","resources/css/filament/admin/theme.css"],
            refresh: true,
        }),
        tailwindcss(),
    ],

    server: {
        host: "0.0.0.0", // <--- Sangat penting ini
        port: 5173,
        strictPort: true,
        hmr: {
            host: "localhost", // <--- Pastikan ini 'localhost'
            clientPort: 5173,
        },
        watch: {
            usePolling: true, // <--- Sangat penting untuk Docker Desktop di Windows/macOS
        },
    },
});
```

``package.json``
ini package json yang dipakai dimana semua versi tidak ada conflict

```bash

{
    "$schema": "https://json.schemastore.org/package.json",
    "private": true,
    "type": "module",
    "scripts": {
        "build": "vite build",
        "dev": "vite"
    },
    "devDependencies": {
        "@tailwindcss/vite": "^4.1.11",
        "autoprefixer": "^10.4.21",
        "axios": "^1.8.2",
        "concurrently": "^9.0.1",
        "laravel-vite-plugin": "^2.0.0",
        "tailwindcss": "^4.1.11",
        "vite": "^7.0.4"
    }
}
```