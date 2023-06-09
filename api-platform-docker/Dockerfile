ARG PHP_VERSION=8.0
ARG NGINX_VERSION=1.22.0


FROM php:$PHP_VERSION-fpm-alpine as api_php
# persistent / runtime deps
RUN apk add --no-cache \
        acl \
        fcgi \
        file \
        gettext \
        git \
    ;

ARG APCU_VERSION=5.1.19
RUN set -eux; \
    apk add --no-cache --virtual .build-deps \
        $PHPIZE_DEPS \
        icu-dev \
        libzip-dev \
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
    ; \
    \
    runDeps="$( \
        scanelf --needed --nobanner --format '%n#p' --recursive /usr/local/lib/php/extensions \
            | tr ',' '\n' \
            | sort -u \
            | awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
    )"; \
    apk add --no-cache --virtual .api-phpexts-rundeps $runDeps; \
    \
    apk del .build-deps

COPY --from=composer /usr/bin/composer /usr/bin/composer
RUN ln -s $PHP_INI_DIR/php.ini-production $PHP_INI_DIR/php.ini
COPY docker/php/conf.d/prod.ini $PHP_INI_DIR/conf.d/api.ini

ENV COMPOSER_ALLOW_SUPERUSER=1

RUN set -eux; \
    composer global config --no-plugins allow-plugins.symfony/flex true; \
    composer global require "symfony/flex" --prefer-dist --no-progress --classmap-authoritative; \
    composer clear-cache

ENV PATH="${PATH}:/root/.composer/vendor/bin"

WORKDIR /srv/api

ARG APP_ENV=prod

COPY composer.json .
COPY composer.lock .
COPY symfony.lock .

RUN set -eux; \
    composer install --prefer-dist --no-dev --no-scripts --no-progress; \
    composer clear-cache

COPY .env .

RUN composer dump-env prod

COPY bin bin
COPY config config
COPY migrations migrations
COPY public public
COPY src src
COPY templates templates

RUN set -eux; \
    mkdir -p var/cache var/log; \
    composer dump-autoload --classmap-authoritative --no-dev; \
    composer run-script --no-dev post-install-cmd; \
    chmod +x bin/console; sync

VOLUME /srv/api/var

COPY docker/php/docker-entrypoint.sh /usr/local/bin/docker-entrypoint

RUN chmod +x /usr/local/bin/docker-entrypoint

ENTRYPOINT ["docker-entrypoint"]

CMD ["php-fpm"]

FROM nginx:$NGINX_VERSION-alpine as api_nginx

COPY docker/nginx/conf.d/default.conf /etc/nginx/conf.d/default.conf

WORKDIR /srv/api/public

COPY --from=api_php /srv/api/public .