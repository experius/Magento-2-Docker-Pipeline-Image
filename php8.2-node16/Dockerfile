# Dockerfile.ci: Optimized for Bitbucket Pipelines

# ==============================================================================
# STAGE 1: CI RUNNER IMAGE
# ==============================================================================
# Use a slim, non-webserver base image for maximum efficiency
FROM php:8.2-cli AS ci_runner

LABEL org.opencontainers.image.authors="dylanmaurits@happyhorizon.com"

# Set non-interactive mode
ENV DEBIAN_FRONTEND=noninteractive
ENV GOPATH=/go
ENV PATH="/usr/local/bin:$GOPATH/bin:${PATH}"

# 1. Install PHP Extension Installer
ADD https://github.com/mlocati/docker-php-extension-installer/releases/latest/download/install-php-extensions /usr/local/bin/

# 2. Install Base System Dependencies & PHP Extensions (Optimized Layer)
RUN set -eux; \
    apt-get update; \
    apt-get install -y --no-install-recommends \
        zip apt-utils gnupg redis-tools default-mysql-client git wget curl lynx psmisc unzip tar cron bash-completion rsync python3 python3-setuptools libzip-dev procps; \
    chmod +x /usr/local/bin/install-php-extensions; \
    install-php-extensions \
        gd opcache bcmath intl mbstring pdo_mysql soap xsl zip sockets calendar ftp oauth openssl; \
    apt-get purge -y --auto-remove; \
    rm -rf /var/lib/apt/lists/*

# 3. Install Node.js 16.x (for asset compilation)
RUN curl -sL https://deb.nodesource.com/setup_16.x | bash - \
    && apt-get install -y nodejs \
    && npm install -g yarn

# 4. Install Composer & Code Sniffer
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin/ --filename=composer --version=2.8.11 \
    && git clone https://github.com/magento/magento-coding-standard.git ~/.composer/vendor/magento/magento-coding-standard \
    && cd ~/.composer/vendor/magento/magento-coding-standard \
    && composer install --no-dev --no-interaction \
    && ln -s ~/.composer/vendor/magento/magento-coding-standard/vendor/bin/phpcs /usr/local/bin/phpcs \
    && rm -rf ~/.composer/vendor/magento/magento-coding-standard/.git \
    && rm -rf ~/.composer/cache

# 5. Install Mhsendmail (Go binary compilation)
RUN apt-get update && apt-get install -y --no-install-recommends golang-go \
    && go install github.com/mailhog/mhsendmail@latest \
    && apt-get purge -y --auto-remove golang-go \
    && rm -rf /var/lib/apt/lists/*

ADD .docker/config/php.ini /usr/local/etc/php/php.ini
COPY .docker/bin/* /usr/local/bin/
COPY .docker/users/* /var/www/
RUN chmod +x /usr/local/bin/*

# --- Final setup ---
WORKDIR /opt/atlassian/pipelines/agent/build
ENV PATH="/root/.composer/vendor/bin/:${PATH}"