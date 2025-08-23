## Panduan Setup Proyek Menggunakan Laravel new

pull image docker image
``
dockerdami/laravel-installer-image
``

atau

buat ``Dockerfile`` sebagai berikut.

```yaml
# Menggunakan image dasar PHP dengan PHP-FPM
FROM php:8.3-fpm-alpine

# Menginstal ekstensi PHP yang dibutuhkan Laravel
RUN apk add --no-cache \
    git \
    curl \
    libzip-dev \
    libpng-dev \
    libjpeg-turbo-dev \
    libxml2-dev

# Menginstal Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Menginstal Laravel Installer
RUN composer global require laravel/installer

# Menambahkan direktori vendor global ke PATH
ENV PATH="/root/.composer/vendor/bin:${PATH}"

# Mengatur working directory
WORKDIR /var/www/html

# Perintah default saat kontainer berjalan
CMD ["php-fpm"]
```

build image 

```bash
docker build -t laravel-installer-image .
```

terus buat proyek dengan command berikut

``docker run --rm -it -v $(pwd):/var/www/html laravel-installer-image laravel new``