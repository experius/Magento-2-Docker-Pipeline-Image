# Dockerfile Refactored for Nginx

# ==============================================================================
# STAGE 1: BUILDER (ASSET GENERATION)
# ==============================================================================
# Changed base image to php:8.4-fpm (or just a non-webserver variant like cli/fpm)
FROM php:8.4-fpm AS base_builder

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
# Changed base image to php:8.4-fpm
FROM php:8.4-fpm

# Install Nginx in the final stage
RUN set -eux; \
    apt-get update; \
    apt-get install -y --no-install-recommends nginx; \
    rm -rf /var/lib/apt/lists/*;

# Set non-interactive mode
ENV DEBIAN_FRONTEND=noninteractive

# check node files
RUN ls -l /usr/lib && ls -l /usr/local/bin

# --- Copy all binaries from builder stage ---
COPY --from=base_builder /usr/local/bin/composer /usr/local/bin/composer
COPY --from=base_builder /usr/local/bin/phpcs /usr/local/bin/phpcs
COPY --from=base_builder /go/bin/mhsendmail /usr/local/bin/mhsendmail

# Copy Node.js and NPM binaries/modules
COPY --from=base_builder /usr/bin/node /usr/bin/node
COPY --from=base_builder /usr/bin/npm /usr/bin/npm
COPY --from=base_builder /usr/bin/npx /usr/bin/npx
COPY --from=base_builder /usr/bin/yarn /usr/bin/yarn
COPY --from=base_builder /usr/lib/node_modules /usr/lib/node_modules
COPY --from=base_builder /usr/local/bin/install-php-extensions /usr/local/bin/install-php-extensions

# --- Add config files from local context ---
ADD .docker/config/preferences /etc/apt/preferences
ADD .docker/config/php.ini /usr/local/etc/php/php.ini
# Changed config file copy to Nginx site configuration
ADD .docker/config/magento.conf /etc/nginx/sites-available/default
ADD .docker/config/custom-xdebug.ini /usr/local/etc/php/conf.d/custom-xdebug.ini
COPY .docker/bin/* /usr/local/bin/
COPY .docker/users/* /var/www/

# --- Configure system ---
RUN set -eux; \
    chmod +x /usr/local/bin/*; \
    # Link Nginx config and remove default
    ln -sf /etc/nginx/sites-available/default /etc/nginx/sites-enabled/default; \
    # The following commands for apache are removed/changed
    # rm -f /etc/apache2/sites-enabled/000-default.conf; \
    # rm -f /etc/apache2/sites-available/default-ssl.conf; \
    # echo "ServerName localhost" >> /etc/apache2/apache2.conf; \
    # a2enmod rewrite headers; \
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

# Added Nginx as the primary execution command, starting PHP-FPM and Nginx
CMD ["/bin/bash", "-c", "service php8.4-fpm start && nginx -g 'daemon off;'"]