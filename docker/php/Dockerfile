FROM php:8.3-fpm

RUN apt-get update && apt-get install -y \
    git unzip \
    libicu-dev libxml2-dev libzip-dev \
    libjpeg-dev libpng-dev libwebp-dev libfreetype6-dev \
    libsodium-dev libonig-dev \
    && pecl install redis \
    && docker-php-ext-enable redis \
    && docker-php-ext-configure gd \
       --with-freetype --with-jpeg --with-webp \
    && docker-php-ext-install \
       intl mysqli pdo_mysql zip gd sodium xml mbstring opcache \
    && rm -rf /var/lib/apt/lists/*

# OPcache (important)
RUN echo "opcache.enable=1" \
 && echo "opcache.memory_consumption=256" \
 && echo "opcache.max_accelerated_files=10000" \
 > /usr/local/etc/php/conf.d/opcache.ini

COPY docker/php/moodle.ini /usr/local/etc/php/conf.d/moodle.ini

WORKDIR /var/www/html