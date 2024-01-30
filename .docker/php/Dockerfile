FROM composer:2 as composer

FROM php:8.2-cli-bullseye
SHELL ["/bin/bash", "-oeux", "pipefail", "-c"]

ARG UID=1000
ARG GID=1000
ENV COMPOSER_HOME=/composer
RUN groupmod -o -g ${GID} www-data && \
    usermod -o -u ${UID} -g www-data www-data

COPY --from=composer /usr/bin/composer /usr/bin/composer

WORKDIR /usr/src/laravel

RUN apt-get update && \
    apt-get -y --no-install-recommends install git \
    libpq-dev && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* && \
    docker-php-ext-install pdo_pgsql && \
    mkdir ${COMPOSER_HOME} && \
    chown -R www-data: ${COMPOSER_HOME}

USER www-data