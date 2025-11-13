# ==============================================================================
# STAGE 1: BUILDER - Used for building Go binaries and installing PHP extensions
# ==============================================================================
FROM php:8.4-apache AS base_builder

LABEL org.opencontainers.image.authors="matthijsbreed@happyhorizon.com"

# Set non-interactive mode and temporary Go environment
ENV DEBIAN_FRONTEND=noninteractive
ENV GOPATH=/go
ENV PATH="/usr/local/bin:$GOPATH/bin:${PATH}"

# --- Install PHP Extension Installer ---
ADD https://github.com/mlocati/docker-php-extension-installer/releases/latest/download/install-php-extensions /usr/local/bin/

# --- Install Base System Dependencies, Build Tools, and PHP Extensions (Optimized Layer) ---
# Combine apt-get operations and cleanup to reduce layers and image size.
RUN set -eux; \
    \
    # 1. Install system tools and common utilities
    apt-get update; \
    apt-get install -y --no-install-recommends \
        zip apt-utils gnupg redis-tools default-mysql-client git vim wget curl lynx psmisc unzip tar cron bash-completion rsync python3 python3-setuptools; \
    \
    # 2. Make PHP extension installer executable
    chmod +x /usr/local/bin/install-php-extensions; \
    \
    # 3. Install PHP Extensions (Extensions and their build dependencies are handled here)
    install-php-extensions \
        gd opcache bcmath intl mbstring pdo_mysql soap xsl zip sockets calendar ftp oauth openssl; \
    \
    # 4. Final APT cleanup
    apt-get purge -y --auto-remove; \
    rm -rf /var/lib/apt/lists/*

# --- Install Node.js (Version 16.x as requested) ---
RUN curl -sL https://deb.nodesource.com/setup_16.x | bash - \
    && apt-get install -y nodejs \
    && npm install -g yarn

# --- Install Composer ---
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin/ --filename=composer --version=2.8.11

# --- Install Code Sniffer (Optimized to clean up git metadata and composer cache) ---
RUN git clone https://github.com/magento/magento-coding-standard.git ~/.composer/vendor/magento/magento-coding-standard \
    && cd ~/.composer/vendor/magento/magento-coding-standard \
    && composer install --no-dev --no-interaction \
    && ln -s ~/.composer/vendor/magento/magento-coding-standard/vendor/bin/phpcs /usr/local/bin/phpcs \
    # Cleanup files that aren't needed after installation
    && rm -rf ~/.composer/vendor/magento/magento-coding-standard/.git \
    && rm -rf ~/.composer/cache \

# --- Install Mhsendmail (Go binary compilation in builder stage) ---
RUN apt-get update && apt-get install -y --no-install-recommends golang-go \
    && go install github.com/mailhog/mhsendmail@latest \
    && apt-get purge -y --auto-remove golang-go \
    && rm -rf /var/lib/apt/lists/* \

# ==============================================================================
# STAGE 2: FINAL IMAGE - Based on the original base image for a cleaner result
# ==============================================================================
FROM php:8.4-apache

# Set non-interactive mode
ENV DEBIAN_FRONTEND=noninteractive

# --- Copy compiled artifacts and configuration from the builder stage ---
# Copy binaries created in the first stage
COPY --from=base_builder /usr/local/bin/composer /usr/local/bin/composer
COPY --from=base_builder /usr/local/bin/phpcs /usr/local/bin/phpcs
COPY --from=base_builder /go/bin/mhsendmail /usr/local/bin/mhsendmail

# Copy Node.js and NPM, including global packages like yarn
COPY --from=base_builder /usr/local/bin/node /usr/local/bin/node
COPY --from=base_builder /usr/local/bin/npm /usr/local/bin/npm
COPY --from=base_builder /usr/local/bin/npx /usr/local/bin/npx
COPY --from=base_builder /usr/local/bin/yarn /usr/local/bin/yarn
COPY --from=base_builder /usr/local/lib/node_modules /usr/local/lib/node_modules

# Ensure install-php-extensions is copied if needed for later runtime use
COPY --from=base_builder /usr/local/bin/install-php-extensions /usr/local/bin/install-php-extensions

# Copy remaining configuration and utility files
ADD .docker/config/preferences /etc/apt/preferences
ADD .docker/config/php.ini /usr/local/etc/php/php.ini
ADD .docker/config/magento.conf /etc/apache2/sites-available/magento.conf
ADD .docker/config/custom-xdebug.ini /usr/local/etc/php/conf.d/custom-xdebug.ini
COPY .docker/bin/* /usr/local/bin/
COPY .docker/users/* /var/www/

# --- Configure system ---
# Combine file permission changes and Apache mods into fewer layers
RUN set -eux; \
    chmod +x /usr/local/bin/*; \
    ln -s /etc/apache2/sites-available/magento.conf /etc/apache2/sites-enabled/magento.conf; \
    rm -f /etc/apache2/sites-enabled/000-default.conf; \
    rm -f /etc/apache2/sites-available/default-ssl.conf; \
    echo "ServerName localhost" >> /etc/apache2/apache2.conf; \
    \
    # User and permissions setup
    a2enmod rewrite headers; \
    \
    chmod 777 -Rf /var/www /var/www/.*; \
    chown -Rf www-data:www-data /var/www /var/www/.*; \
    usermod -u 1000 www-data; \
    chsh -s /bin/bash www-data

# --- Final image metadata ---
ENV PATH="/var/www/.composer/vendor/bin/:${PATH}"
VOLUME /var/www/html
WORKDIR /var/www/html