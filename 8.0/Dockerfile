FROM debian:buster-slim AS base
# hadolint ignore=DL3009,DL3008
RUN apt-get update \
  && apt-get install --yes --no-install-recommends \
    apt-transport-https \
    ca-certificates \
  && rm -rf /var/lib/apt/lists/* \
  && { \
    echo 'deb https://deb.debian.org/debian buster main'; \
    echo 'deb https://deb.debian.org/debian buster-updates main'; \
    echo 'deb https://deb.debian.org/debian-security buster/updates main'; \
    } > /etc/apt/sources.list

FROM bcinteractive/nginx:1.19.10 AS nginx

FROM base AS downloader
# hadolint ignore=DL3009,DL3008
RUN apt-get update \
  && apt-get install --yes --no-install-recommends \
   wget \
   gnupg \
   dirmngr \
   git \
   unzip \
   xz-utils

FROM downloader AS upx
RUN wget -nv --compression=gzip -O "upx-3.96-amd64_linux.tar.xz" "https://github.com/upx/upx/releases/download/v3.96/upx-3.96-amd64_linux.tar.xz" \
  && tar xf "upx-3.96-amd64_linux.tar.xz" --strip-components=1 \
  && chmod +x /upx

# @TODO: check signature
FROM downloader AS s6-overlay
ARG S6_OVERLAY_VERSION=2.2.0.3
RUN wget -nv --compression=gzip -O "s6-overlay-${S6_OVERLAY_VERSION}.tar.gz" "https://github.com/just-containers/s6-overlay/releases/download/v${S6_OVERLAY_VERSION}/s6-overlay-amd64.tar.gz" \
  && tar xzf "s6-overlay-${S6_OVERLAY_VERSION}.tar.gz" -C /opt/
#  $ curl https://keybase.io/justcontainers/key.asc | gpg --import
#
#Then verify the downloaded files:
#
#$ gpg --verify s6-overlay-amd64.tar.gz.sig s6-overlay-amd64.tar.gz
#gpg: Signature made Sun 22 Nov 2015 09:11:29 AM CST using RSA key ID BD7BF0DC
#gpg: Good signature from "Just Containers <just-containers@jrjrtech.com>"

FROM downloader AS php
SHELL ["/bin/bash", "-o", "pipefail", "-c"]
ARG PHP_VERSION="8.0.5"
ARG PHP_SHA256="5dd358b35ecd5890a4f09fb68035a72fe6b45d3ead6999ea95981a107fd1f2ab"
ARG GPG_KEYS="1729F83938DA44E27BA0F4D3DBDB397470D12172 BFDDD28642824F8118EF77909B67A5C12229118F"
RUN wget -nv -O php-${PHP_VERSION}.tar.xz https://www.php.net/distributions/php-${PHP_VERSION}.tar.xz \
  && wget -nv -O php-${PHP_VERSION}.tar.xz.asc https://www.php.net/distributions/php-${PHP_VERSION}.tar.xz.asc \
  && echo "${PHP_SHA256} *php-${PHP_VERSION}.tar.xz" | sha256sum -c \
  && GNUPGHOME="$(mktemp -d)" \
  && export GNUPGHOME \
  && for key in $GPG_KEYS; do gpg --batch --keyserver ha.pool.sks-keyservers.net --recv-keys "${key}"; done \
  && gpg --batch --verify php-${PHP_VERSION}.tar.xz.asc php-${PHP_VERSION}.tar.xz \
  && tar xvf php-${PHP_VERSION}.tar.xz

FROM downloader AS php-memcached
ARG PHP_MEMCACHED_VERSION="v3.1.5"
RUN git clone https://github.com/php-memcached-dev/php-memcached
WORKDIR /php-memcached
RUN git checkout "${PHP_MEMCACHED_VERSION}" \
  && git submodule update --init

FROM base AS base-php
ARG PHP_VERSION="8.0.5"
# Prevent Debian's PHP packages from being installed
# hadolint ignore=DL3009,DL3008
RUN { \
    echo 'Package: php*'; \
    echo 'Pin: release *'; \
    echo 'Pin-Priority: -1'; \
  } > /etc/apt/preferences.d/no-debian-php \
  && apt-get update \
  && apt-get install --yes --no-install-recommends \
    libedit-dev \
    libcurl4-openssl-dev \
    libxml2-dev \
    libssl-dev \
    zlib1g-dev \
    libbz2-dev \
    libpng-dev \
    libwebp-dev \
    libjpeg-dev \
    libxpm-dev \
    libfreetype6-dev \
    libonig-dev \
    libsqlite3-dev \
    libsodium-dev \
    libargon2-dev \
    libxslt1-dev \
    libzip-dev \
    libpq-dev \
    libgmp-dev \
    libpspell-dev \
    libmemcached-dev \
  && rm -rf /var/lib/apt/lists/*

FROM base-php AS builder-php
ARG PHP_VERSION="8.0.5"
# -D_LARGEFILE_SOURCE and -D_FILE_OFFSET_BITS=64 (https://www.php.net/manual/en/intro.filesystem.php)
ARG PHP_CFLAGS="-fstack-protector-strong -fpic -fpie -O2 -D_LARGEFILE_SOURCE -D_FILE_OFFSET_BITS=64"
ARG PHP_CPPFLAGS="$PHP_CFLAGS"
ARG PHP_LDFLAGS="-Wl,-O1 -pie"
# hadolint ignore=DL3009,DL3008
RUN apt-get update \
  && apt-get install --yes --no-install-recommends \
    wget \
    xz-utils \
    gnupg \
    dirmngr \
    file \
    make \
    gcc \
    g++ \
    libc-dev \
    autoconf \
    pkg-config

## PHP BUILD
COPY --from=php /php-${PHP_VERSION} /php-${PHP_VERSION}
WORKDIR /php-${PHP_VERSION}
RUN ./configure \
    # make sure invalid --configure-flags are fatal errors instead of just warnings
    --enable-option-checking=fatal \
    --disable-all \
    --enable-fpm \
    --with-fpm-user=www-data \
    --with-fpm-group=www-data \
    --disable-cgi \
    --with-config-file-path=/usr/local/etc/php/php.ini \
    --with-config-file-scan-dir=/usr/local/etc/php/conf.d \
    --disable-short-tags \
    --enable-phar \
    --with-curl \
    --with-zlib \
    --with-libxml \
    --with-openssl \
    --enable-bcmath \
    --with-bz2 \
    --enable-calendar \
    --enable-exif \
    --enable-ftp \
    --enable-gd \
    --with-webp \
    --with-jpeg \
    --with-xpm \
    --with-freetype \
    --with-gettext \
    --with-mhash \
    --with-iconv \
    --enable-intl \
    --enable-mbstring \
    --with-sqlite3 \
    --with-mysqli=mysqlnd \
    --with-pgsql \
    --enable-pdo \
    --with-pdo-mysql=mysqlnd \
    --with-pdo-sqlite \
    --with-pdo-pgsql \
    --enable-soap \
    --with-sodium \
    --enable-sockets \
    --with-password-argon2 \
    --enable-dom \
    --with-xsl \
    --with-zip \
    --with-libedit \
    --enable-opcache \
    --enable-posix \
    --enable-fileinfo \
    --enable-session \
    --enable-xmlreader \
    --enable-xmlwriter \
    --enable-simplexml \
		#--enable-json \
    --enable-ctype \
    #filter ?
    #gmp ?
    #imagick
    #imap
    #memcached
    #mongodb
    #-readline ?
    #pspell
    #redis
    #sysvmsg
    #sysvsem
    #sysvshm
    #tokenizer
    #wddx
    #xml
    #Zend OPcache
    --enable-filter \
    --with-gnu-ld \
    --enable-shared=yes \
    --enable-static=yes \
    --enable-mbregex \
    --with-pspell \
    --with-gmp \
    --disable-rpath \
    --enable-libgcc \
    --enable-opcache \
  && make -j "$(nproc)" \
  && make install \
  && mkdir -p /usr/local/etc/php \
  && cp php.ini-production /usr/local/etc/php/php.ini

## PHP-MEMCACHED BUILD
COPY --from=php-memcached /php-memcached /php-memcached
WORKDIR /php-memcached
RUN phpize \
  && ./configure \
  && make \
  && make install

## OTHERS
RUN mkdir -p /usr/local/etc/php \
  && mkdir -p /etc/php/conf.d /etc/php/php-fpm.d /var/www/html \
  && mkdir -p /usr/local/etc/php/conf.d /usr/local/etc/php-fpm.d \
  && chown -R www-data:www-data /var/www/html \
  && chmod 755 /var/www/html \
	# @TODO: manage memcached ip
  && { \
    echo "extension=memcached.so"; \
    } > /usr/local/etc/php/conf.d/memcached.ini \
  && { \
    echo "zend_extension=opcache.so"; \
    echo "opcache.enable=1"; \
    echo "opcache.enable_cli=0"; \
    echo "opcache.memory_consumption=256"; \
    echo "opcache.interned_strings_buffer=16"; \
    echo "opcache.max_accelerated_files=65407"; \
    echo "opcache.max_wasted_percentage=5"; \
    echo "opcache.validate_timestamps=0"; \
    echo "opcache.revalidate_freq=0"; \
    echo "opcache.max_file_size=0"; \
    } > /usr/local/etc/php/conf.d/opcache.ini \
  && { \
    echo "include=etc/php-fpm.d/*.conf"; \
    echo "[global]"; \
    echo "error_log = /proc/self/fd/2"; \
    echo "daemonize = no"; \
  } > /usr/local/etc/php/php-fpm.conf \
  && { \
    echo "[www]"; \
    echo "listen = /tmp/php-fpm.sock"; \
    echo 'listen.owner = www-data'; \
		echo 'listen.group = www-data'; \
		echo 'listen.mode = 0660'; \
    echo "user = www-data"; \
    echo "group = www-data"; \
    echo "pm = dynamic"; \
    echo "pm.max_children = 5"; \
    echo "pm.min_spare_servers = 1"; \
    echo "pm.max_spare_servers = 3"; \
    echo "pm.start_servers = 2"; \
    echo 'catch_workers_output = 1'; \
    echo 'pm.status_path = /status'; \
    echo 'ping.path = /ping'; \
    echo "php_admin_flag[log_errors] = on"; \
    echo "php_admin_value[error_log] = /proc/self/fd/2"; \
    echo "php_flag[display_errors] = off"; \
    echo "php_value[open_basedir] = /var/www/html:/dev/urandom:/tmp"; \
    echo "php_flag[file_uploads] = on"; \
    echo "php_value[upload_tmp_dir] = /tmp"; \
    echo "php_flag[allow_url_fopen] = on"; \
    echo "php_value[memory_limit] = 128M"; \
    echo "php_value[post_max_size] = 1024M"; \
    echo "php_value[upload_max_filesize] = 1024M"; \
  } > /usr/local/etc/php-fpm.d/www.conf \
  # smoke tests
  && php --version

FROM base AS code
COPY --from=upx /upx /usr/local/bin
COPY --from=s6-overlay /opt/ /opt/
COPY --from=builder-php /usr/local/bin /opt/usr/local/bin
COPY --from=builder-php /usr/local/etc /opt/usr/local/etc
COPY --from=builder-php /usr/local/include /opt/usr/local/include
COPY --from=builder-php /usr/local/lib /opt/usr/local/lib
COPY --from=builder-php /usr/local/php /opt/usr/local/php
COPY --from=builder-php /usr/local/sbin /opt/usr/local/sbin
RUN cp --archive --parents /etc/passwd /opt \
  && cp --archive --parents /etc/group /opt \
  && cp --archive --parents /etc/shadow /opt \
  # hardening: remove unnecessary accounts \
  && sed --in-place --regexp-extended '/^(root|nobody|www-data)/!d' /opt/etc/group \
  && sed --in-place --regexp-extended '/^(root|nobody|www-data)/!d' /opt/etc/passwd \
  && sed --in-place --regexp-extended '/^(root|nobody|www-data)/!d' /opt/etc/shadow \
  # hardening: remove interactive shell \
  && sed --in-place --regexp-extended 's#^([^:]+):([^:]+):([^:]+):([^:]+):([^:]+):([^:]+):(.+)$#\1:\2:\3:\4:\5:\6:/bin/nologin#' /opt/etc/passwd
  # @TODO: hardening: lock all accounts
RUN mkdir -p /opt/var/www/html
RUN mkdir -p /opt/usr/local/etc/php/conf.d /opt/usr/local/etc/php-fpm.d
RUN mkdir -p /opt/etc/services.d/nginx /opt/etc/services.d/php-fpm
RUN mkdir -p /opt/tmp/client_body_temp /opt/tmp/proxy_temp /opt/tmp/fastcgi_temp
RUN mkdir -p /opt/etc/fix-attrs.d
RUN mkdir -p /opt/usr/local/NONE/etc/php-fpm.d/
RUN cp /opt/usr/local/etc/php-fpm.d/* /opt/usr/local/NONE/etc/php-fpm.d/
RUN { \
  echo '#!/usr/bin/execlineb -P'; \
  echo '#s6-setuidgid www-data'; \
  echo '/usr/local/sbin/php-fpm -c /usr/local/etc/php/php.ini -y /usr/local/etc/php/php-fpm.conf'; \
} > /opt/etc/services.d/php-fpm/run
RUN { \
  echo '#!/usr/bin/execlineb -P'; \
  echo '#s6-setuidgid www-data'; \
  echo '/usr/sbin/nginx -c /etc/nginx/nginx.conf'; \
} > /opt/etc/services.d/nginx/run
RUN { \
  echo '/var/www/html true www-data:www-data 0600 0700'; \
} > /opt/etc/fix-attrs.d/data
RUN find /opt/usr/bin /opt/usr/local/bin /opt/usr/local/sbin -executable -type f -exec strip --strip-all '{}' \;
RUN find /opt/usr/bin /opt/usr/local/bin /opt/usr/local/sbin -executable -type f -exec upx '{}' \;
COPY --from=nginx /usr/sbin/nginx /opt/usr/sbin/nginx
COPY --from=nginx /lib /opt/lib/
COPY --from=nginx /etc/nginx /opt/etc/nginx

FROM base-php
ARG PHP_VERSION="8.0.5"
ARG BUILD_DATE
ARG VCS_REF
LABEL org.opencontainers.image.created="${BUILD_DATE}"
LABEL org.opencontainers.image.url="https://github.com/bc-interactive/docker-php-fpm"
LABEL org.opencontainers.image.source="https://github.com/bc-interactive/docker-php-fpm"
LABEL org.opencontainers.image.version="${PHP_VERSION}"
LABEL org.opencontainers.image.revision="${VCS_REF}"
LABEL org.opencontainers.image.vendor="bcinteractive"
LABEL org.opencontainers.image.title="php-fpm"
LABEL org.opencontainers.image.authors="BC INTERACTIVE <contact@bc-interactive.fr>"
COPY --from=code /opt/ /
WORKDIR /var/www/html
EXPOSE 80 443
ENTRYPOINT ["/init"]
