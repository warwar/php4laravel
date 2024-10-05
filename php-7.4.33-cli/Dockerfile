FROM alpine:3.20 AS stage0

# dependencies required for running "phpize"
# these get automatically installed and removed by "docker-php-ext-*" (unless they're already installed)
ENV PHPIZE_DEPS="autoconf dpkg-dev dpkg file g++ gcc libc-dev make pkgconf re2c"
ENV PHP_INI_DIR /usr/local/etc/php

# persistent / runtime deps
RUN apk add --update --no-cache ca-certificates curl openssl tar xz bash unzip zip libbz2 libpng libjpeg libzip freetype icu-libs icu-data-full 7zip gettext libmemcached expat


# ensure www-data user exists
RUN set -eux \
	&& adduser -u 82 -D -S -G www-data www-data

RUN set -eux \
	&& mkdir -p "$PHP_INI_DIR/conf.d" \
	&& [ ! -d /var/www/html ]; mkdir -p /var/www/html; chown www-data:www-data /var/www/html; chmod 1777 /var/www/html

ENV PHP_CFLAGS="-fstack-protector-strong -fpic -fpie -O2 -D_LARGEFILE_SOURCE -D_FILE_OFFSET_BITS=64"
ENV PHP_CPPFLAGS="$PHP_CFLAGS"
ENV PHP_LDFLAGS="-Wl,-O1 -pie"

#7.4
ENV GPG_KEYS 5A52880781F755608BF815FC910DEB46F53EA312 42670A7FE4D0441C8E4632349E4FDC074A4EF02D

ENV PHP_VERSION 7.4.33
ENV PHP_URL="https://www.php.net/distributions/php-7.4.33.tar.xz" PHP_ASC_URL="https://www.php.net/distributions/php-7.4.33.tar.xz.asc"
ENV PHP_SHA256="924846abf93bc613815c55dd3f5809377813ac62a9ec4eb3778675b82a27b927"

RUN set -eux; \
	\
	apk add --no-cache --virtual .fetch-deps gnupg; \
	\
	mkdir -p /usr/src; \
	cd /usr/src; \
	\
	curl -fsSL -o php.tar.xz "$PHP_URL"; \
	\
	if [ -n "$PHP_SHA256" ]; then \
		echo "$PHP_SHA256 *php.tar.xz" | sha256sum -c -; \
	fi; \
	\
	if [ -n "$PHP_ASC_URL" ]; then \
		curl -fsSL -o php.tar.xz.asc "$PHP_ASC_URL"; \
		export GNUPGHOME="$(mktemp -d)"; \
		for key in $GPG_KEYS; do \
			gpg --batch --keyserver keyserver.ubuntu.com --recv-keys "$key"; \
		done; \
		gpg --batch --verify php.tar.xz.asc php.tar.xz; \
		gpgconf --kill all; \
		rm -rf "$GNUPGHOME"; \
	fi; \
	\
	apk del --no-network .fetch-deps

COPY docker-php-source /usr/local/bin/

RUN set -eux; \
	apk add --no-cache --virtual .build-deps \
		$PHPIZE_DEPS \
		argon2-dev \
		coreutils \
		curl-dev \
		gnu-libiconv-dev \
		libsodium-dev \
		libxml2-dev \
		linux-headers \
		oniguruma-dev \
		openssl-dev \
		readline-dev \
		sqlite-dev \
        libffi-dev \
        freetype-dev  \
        libjpeg-turbo-dev  \
        libzip-dev \
        libmcrypt-dev  \
        libwebp-dev  \
        libpng-dev  \
        libxpm-dev  \
        icu-dev \
        gettext-dev \
        musl-dev \
        mariadb-dev \
        libmemcached-dev \
        expat-dev \
	;

#ENV LD_PRELOAD /usr/lib/preloadable_libiconv.so php
RUN rm -rf /usr/include/iconv.h \
    && rm -rf /usr/include/libcharset.h \
    && rm -rf /usr/include/localcharset.h \
    && ln -sv /usr/include/gnu-libiconv/*.h /usr/include/

RUN export \
		CFLAGS="$PHP_CFLAGS" \
		CPPFLAGS="$PHP_CPPFLAGS" \
		LDFLAGS="$PHP_LDFLAGS" \
		PHP_BUILD_PROVIDER='https://github.com/docker-library/php' \
		PHP_UNAME='Linux - Docker' \
	;
RUN docker-php-source extract




WORKDIR /usr/src/php

RUN apk add patch
COPY php-7.4.26-openssl3.patch /usr/src
RUN patch -p1 < ../php-7.4.26-openssl3.patch

#	cd /usr/src/php; \
#	gnuArch="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)"; \
RUN	echo 'gnuArch="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)"; \
      ./configure \
		--build="$gnuArch" \
        --disable-debug \
		--with-config-file-path="$PHP_INI_DIR" \
		--with-config-file-scan-dir="$PHP_INI_DIR/conf.d" \
		--enable-option-checking=fatal \
		--with-mhash \
		--enable-ftp \
#        --enable-bcmath \
        --enable-calendar \
        --enable-ctype \
		--with-curl \
		--with-curl \
        --enable-exif \
        --with-ffi \
        --enable-fileinfo \
        --enable-filter \
        --enable-ftp \
        --enable-gd \
        --with-freetype=/usr/ \
        --with-gettext \
        --enable-intl \
        --enable-json \
        --with-mhash \
        --with-iconv=/usr \
		--enable-mbstring \
        --enable-pcntl \
		--enable-mysqlnd \
        --with-pdo-mysql=/usr \
        --with-expat \
		--with-password-argon2 \
		--with-sodium=shared \
		--with-pdo-sqlite=/usr \
		--with-sqlite3=/usr \
		--with-openssl \
		--with-readline \
		--with-zlib \
#        --with-zlib-dir=/usr \
		--enable-cli \
        --enable-sockets \
        --disable-opcache ' >> ../cfg-with-param.sh ;
RUN cd /usr/src/php; sh ../cfg-with-param.sh;
#		--with-pic \
#		--enable-phpdbg \
#		--enable-phpdbg-readline \

RUN cd /usr/src/php; make -j "$(nproc)"
RUN cd /usr/src/php; find -type f -name '*.a' -delete
RUN cd /usr/src/php; make install
RUN cd /usr/src/php; find /usr/local -type f -perm '/0111' -exec sh -euxc 'strip --strip-all "$@" || : ' -- '{}' +
RUN cd /usr/src/php; make clean

RUN cd /usr/src/php; cp -v php.ini-* "$PHP_INI_DIR/"; \
	cd /; \
	docker-php-source delete;

RUN cd /usr/src/php; runDeps="$( \
		scanelf --needed --nobanner --format '%n#p' --recursive /usr/local \
			| tr ',' '\n' \
			| sort -u \
			| awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
	)"; \
apk add --no-cache $runDeps;

COPY docker-php-ext-* docker-php-entrypoint install-php-extensions /usr/local/bin/

#
#RUN	apk del --no-network .build-deps;
##	pecl update-channels; \
##	rm -rf /tmp/pear ~/.pearrc; \
RUN php --version


# sodium was built as a shared module (so that it can be replaced later if so desired), so let's enable it too (https://github.com/docker-library/php/issues/598)
#RUN docker-php-ext-enable sodium

#RUN pecl install --force igbinary
RUN install-php-extensions igbinary
RUN docker-php-ext-enable igbinary

#RUN pecl install --force msgpack
RUN install-php-extensions msgpack
RUN docker-php-ext-enable msgpack

#RUN pecl install --force --enable-redis-igbinary redis
RUN install-php-extensions redis
RUN docker-php-ext-enable redis

RUN install-php-extensions memcached
RUN docker-php-ext-enable memcached

RUN install-php-extensions zip
RUN install-php-extensions shmop
RUN install-php-extensions sodium
RUN install-php-extensions sysvmsg
RUN install-php-extensions sysvsem
RUN install-php-extensions sysvshm
RUN install-php-extensions opcache
RUN install-php-extensions bcmath



FROM scratch
ENV PHPIZE_DEPS="autoconf dpkg-dev dpkg file g++ gcc libc-dev make pkgconf re2c"
ENV PHP_INI_DIR="/usr/local/etc/php"
ENV PHP_CFLAGS="-fstack-protector-strong -fpic -fpie -O2 -D_LARGEFILE_SOURCE -D_FILE_OFFSET_BITS=64"
ENV PHP_CPPFLAGS="$PHP_CFLAGS"
ENV PHP_LDFLAGS="-Wl,-O1 -pie"
ENV BUILD_DEPS="$PHPIZE_DEPS argon2-dev coreutils curl-dev gnu-libiconv-dev libsodium-dev libxml2-dev linux-headers oniguruma-dev openssl-dev readline-dev sqlite-dev"

COPY --from=stage0 / /
STOPSIGNAL SIGQUIT

ENTRYPOINT ["docker-php-entrypoint"]
CMD ["php", "-a"]
