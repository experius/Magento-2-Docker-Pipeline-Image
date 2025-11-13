# Dockerfile Refactored to eliminate circular dependency

# ==============================================================================
# STAGE 1: BUILDER (ASSET GENERATION)
# ==============================================================================
FROM php:8.4-apache AS base_builder

LABEL org.opencontainers.image.authors="matthijsbreed@happyhorizon.com"

# Set non-interactive mode and temporary Go environment
ENV DEBIAN_FRONTEND=noninteractive
ENV GOPATH=/go
ENV PATH="/usr/local/bin:$GOPATH/bin:${PATH}"

# 1. Install PHP Extension Installer
ADD https://github.com/mlocati/docker-php-extension-installer/releases/latest/download/install-php-extensions /usr/local/bin/

# 2. Install Base System Dependencies & PHP Extensions (Optimized Layer)
RUN set -eux; \
    apt-get update; \
    apt-get install -y --no-install-recommends \
        zip apt-utils gnupg redis-tools default-mysql-client git vim wget curl lynx psmisc unzip tar cron bash-completion rsync python3 python3-setuptools; \
    chmod +x /usr/local/bin/install-php-extensions; \
    install-php-extensions \
        gd opcache bcmath intl mbstring pdo_mysql soap xsl zip sockets calendar ftp oauth openssl; \
    apt-get purge -y --auto-remove; \
    rm -rf /var/lib/apt/lists/*

# 3. Install Node.js 16.x (as requested)
RUN curl -sL https://deb.nodesource.com/setup_16.x | bash - \
    && apt-get install -y nodejs \
    && npm install -g yarn

# 4. Install Composer
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin/ --filename=composer --version=2.8.11

# 5. Install Code Sniffer & Cleanup
RUN git clone https://github.com/magento/magento-coding-standard.git ~/.composer/vendor/magento/magento-coding-standard \
    && cd ~/.composer/vendor/magento/magento-coding-standard \
    && composer install --no-dev --no-interaction \
    && ln -s ~/.composer/vendor/magento/magento-coding-standard/vendor/bin/phpcs /usr/local/bin/phpcs \
    && rm -rf ~/.composer/vendor/magento/magento-coding-standard/.git \
    && rm -rf ~/.composer/cache

# 6. Install Mhsendmail (Go binary compilation)
RUN apt-get update && apt-get install -y --no-install-recommends golang-go \
    && go install github.com/mailhog/mhsendmail@latest \
    && apt-get purge -y --auto-remove golang-go \
    && rm -rf /var/lib/apt/lists/*


# ==============================================================================
# STAGE 2: FINAL IMAGE (RUNTIME)
# ==============================================================================
FROM php:8.4-apache

# Set non-interactive mode
ENV DEBIAN_FRONTEND=noninteractive

# --- Copy all binaries from builder stage ---
COPY --from=base_builder /usr/local/bin/composer /usr/local/bin/composer
COPY --from=base_builder /usr/local/bin/phpcs /usr/local/bin/phpcs
COPY --from=base_builder /go/bin/mhsendmail /usr/local/bin/mhsendmail

# Copy Node.js and NPM binaries/modules
COPY --from=base_builder /usr/bin/node /usr/bin/node
COPY --from=base_builder /usr/lib/bin/npm /usr/lib/bin/npm
COPY --from=base_builder /usr/lib/bin/npx /usr/lib/bin/npx
COPY --from=base_builder /usr/lib/bin/yarn /usr/lib/bin/yarn
COPY --from=base_builder /usr/lib/lib/node_modules /usr/lib/lib/node_modules
COPY --from=base_builder /usr/local/bin/install-php-extensions /usr/local/bin/install-php-extensions

# --- Add config files from local context ---
ADD .docker/config/preferences /etc/apt/preferences
ADD .docker/config/php.ini /usr/local/etc/php/php.ini
ADD .docker/config/magento.conf /etc/apache2/sites-available/magento.conf
ADD .docker/config/custom-xdebug.ini /usr/local/etc/php/conf.d/custom-xdebug.ini
COPY .docker/bin/* /usr/local/bin/
COPY .docker/users/* /var/www/

# --- Configure system ---
RUN set -eux; \
    chmod +x /usr/local/bin/*; \
    ln -s /etc/apache2/sites-available/magento.conf /etc/apache2/sites-enabled/magento.conf; \
    rm -f /etc/apache2/sites-enabled/000-default.conf; \
    rm -f /etc/apache2/sites-available/default-ssl.conf; \
    echo "ServerName localhost" >> /etc/apache2/apache2.conf; \
    \
    a2enmod rewrite headers; \
    \
    # Permission changes
    chmod 777 -Rf /var/www /var/www/.*; \
    chown -Rf www-data:www-data /var/www /var/www/.*; \
    usermod -u 1000 www-data; \
    chsh -s /bin/bash www-data

# --- Final image metadata ---
ENV PATH="/var/www/.composer/vendor/bin/:${PATH}"
VOLUME /var/www/html
WORKDIR /var/www/html