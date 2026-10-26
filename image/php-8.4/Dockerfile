# syntax=docker/dockerfile:1
#
# ghcr.io/glitchr-studio/php:8.4 - the PHP every CI job of the omni families
# runs in, built once by .github/workflows/image.yml instead of apt-get and
# docker-php-ext-install in each job.
#
# The extensions are the union of what the suites ask for: omnisense/omnibase
# (intl, zip, gd, bcmath, ftp, xsl, exif, imagick, igbinary; git2 for the
# forge's repository viewer), omnibridge/* (intl, bcmath, imagick), omniauth/*
# (openssl, sodium), omnisearch/* and glitchr/typesense-bundle (curl, zip).
# pdo_sqlite, curl, openssl, sodium and mbstring come with the official image.
#
# Composer and PHPStan are pinned: a job resolves the package's dependencies,
# never its tools.

ARG PHP_IMAGE=php:8.4.27-cli-bookworm@sha256:496b5e02fe9fe44fcfb715023b931048111dc4e0bc719789d83c906be9f04b05
FROM ${PHP_IMAGE}

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        git openssh-client unzip libicu-dev libzip-dev libpng-dev libjpeg-dev libwebp-dev libfreetype-dev \
        libxslt1-dev libmagickwand-dev libgit2-dev \
    && docker-php-ext-configure gd --with-jpeg --with-webp --with-freetype \
    && docker-php-ext-install -j"$(nproc)" intl zip gd bcmath ftp xsl exif \
    && pecl install imagick-3.8.0 igbinary-3.2.16 \
    && docker-php-ext-enable imagick igbinary \
    && rm -rf /var/lib/apt/lists/* /tmp/pear

# php-git2 (libgit2 bindings), at the commit the core's harness has been built with.
ARG PHP_GIT2_COMMIT=9eac6fb33e4fbc41a29274455265f365d3181952
RUN git clone https://github.com/RogerGee/php-git2.git /usr/src/php-git2 \
    && cd /usr/src/php-git2 \
    && git checkout -q "$PHP_GIT2_COMMIT" \
    && phpize \
    && ./configure --with-libgit2=/usr \
    && make -j"$(nproc)" \
    && make install \
    && docker-php-ext-enable git2 \
    && cd / && rm -rf /usr/src/php-git2

COPY --from=composer:2.10.3 /usr/bin/composer /usr/local/bin/composer

ARG PHPSTAN_VERSION=2.3.1
RUN curl -fsSL -o /usr/local/bin/phpstan "https://github.com/phpstan/phpstan/releases/download/${PHPSTAN_VERSION}/phpstan.phar" \
    && chmod +x /usr/local/bin/phpstan \
    && phpstan --version

# A checkout mounted by the runner belongs to another user. Upload progress is
# off: CVE-2026-109619's mitigation (php.net, 2026-10-21), and no suite needs it.
RUN git config --system --add safe.directory '*' \
    && printf 'memory_limit = 1G\nsession.upload_progress.enabled = 0\n' > /usr/local/etc/php/conf.d/zz-ci.ini

ENV COMPOSER_ALLOW_SUPERUSER=1 \
    COMPOSER_NO_INTERACTION=1 \
    COMPOSER_CACHE_DIR=/tmp/composer-cache

LABEL org.opencontainers.image.source="https://github.com/glitchr-studio/ci" \
      org.opencontainers.image.description="PHP 8.4 with Composer, PHPStan and the extensions the omni families' suites need" \
      org.opencontainers.image.licenses="MIT"

RUN php -m && composer --version \
    && php -r 'exit(PHP_VERSION_ID >= 80427 && !ini_get("session.upload_progress.enabled") ? 0 : 1);'
