ARG PHP_VERSION=8.0
ARG NGINX_VERSION=1.17
ARG ALPINE_VERSION=3.13

FROM php:${PHP_VERSION}-fpm-alpine${ALPINE_VERSION} AS php

#timezone
ENV TZ=Europe/Warsaw
RUN apk add --no-cache tzdata
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone
RUN printf '[PHP]\ndate.timezone = "Europe/Warsaw"\n' > /$PHP_INI_DIR/conf.d/tzone.ini

RUN apk add --no-cache \
		acl \
		fcgi \
		file \
		gettext \
		git \
		gnu-libiconv \
	;

ENV LD_PRELOAD /usr/lib/preloadable_libiconv.so

ARG APCU_VERSION=5.1.20
RUN set -eux; \
	apk add --no-cache --virtual .build-deps \
		$PHPIZE_DEPS \
		icu-dev \
		libzip-dev \
		zlib-dev \
		postgresql-dev \
	; \
	\
	docker-php-ext-configure zip; \
	docker-php-ext-install -j$(nproc) \
		intl \
		pdo_pgsql \
		zip \
	; \
	pecl install \
		apcu-${APCU_VERSION} \
	; \
	pecl clear-cache; \
	docker-php-ext-enable \
		apcu \
		opcache \
	;

COPY docker/php/docker-healthcheck.sh /usr/local/bin/docker-healthcheck
RUN chmod +x /usr/local/bin/docker-healthcheck

HEALTHCHECK --interval=10s --timeout=3s --retries=3 CMD ["docker-healthcheck"]

COPY docker/php/docker-entrypoint.sh /usr/local/bin/docker-entrypoint
RUN chmod +x /usr/local/bin/docker-entrypoint

WORKDIR /srv

ARG APP_ENV=prod

COPY composer.json composer.lock symfony.lock ./

ARG USER_ID=1000
RUN apk --no-cache add shadow && usermod -u ${USER_ID} www-data

COPY .env ./
COPY bin bin/
COPY config config/
COPY public public/
COPY src src/
RUN chmod -R +x bin

RUN mkdir -p var vendor
RUN chown -R www-data:www-data .
RUN chown -R www-data:www-data /usr/local/etc/php
VOLUME /srv/var

USER www-data

COPY --from=composer:2.1 /usr/bin/composer /usr/bin/composer

ENTRYPOINT ["docker-entrypoint"]
CMD ["php-fpm"]

FROM php AS php_prod

RUN composer install --no-dev --no-scripts --no-interaction --prefer-dist --optimize-autoloader

FROM nginx:${NGINX_VERSION}-alpine AS nginx

COPY docker/nginx/conf.d/default.conf /etc/nginx/conf.d/default.conf

WORKDIR /srv/public

COPY --from=php /srv/public ./
