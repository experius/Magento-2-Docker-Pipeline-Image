# Dockerfile.ci: Optimized for Bitbucket Pipelines

# ==============================================================================
# STAGE 1: CI RUNNER IMAGE
# ==============================================================================
FROM php:8.4-cli AS ci_runner

LABEL org.opencontainers.image.authors="dylanmaurits@happyhorizon.com" \
      org.opencontainers.image.title="php8.4-node16-ci" \
      org.opencontainers.image.description="CI runner image for PHP 8.4 with Node 16, Composer 2, and Magento Coding Standard" \
      org.opencontainers.image.source="https://github.com/happyhorizon/Magento-2-Docker-Pipeline-Image" \
      org.opencontainers.image.licenses="MIT" \
      org.opencontainers.image.created="2025-11-14"

# Set non-interactive mode and essential paths
ENV DEBIAN_FRONTEND=noninteractive
ENV GOPATH=/go
ENV PATH="/usr/local/bin:$GOPATH/bin:${PATH}"

# 1. Install PHP Extension Installer (via curl/pipe for minimal layer size)
# 2. Install Dependencies, PHP Extensions, Node.js, Go, and Cleanup in ONE Layer
RUN set -eux; \
    # Fetch and install the PHP extension installer
    curl -fsSL https://github.com/mlocati/docker-php-extension-installer/releases/latest/download/install-php-extensions -o /usr/local/bin/install-php-extensions; \
    chmod +x /usr/local/bin/install-php-extensions; \
    \
    # Install Node.js 16 and necessary system dependencies (Combined APT steps)
    curl -fsSL https://deb.nodesource.com/setup_16.x | bash -; \
    apt-get update; \
    apt-get install -y --no-install-recommends \
        # Essential/Common
        zip apt-utils gnupg wget curl psmisc unzip tar cron bash-completion rsync procps \
        # PHP related libs
        libzip-dev openssh-client redis-tools default-mysql-client \
        # Tools
        git lynx python3 python3-setuptools golang-go nodejs; \
    \
    # Install PHP extensions
    install-php-extensions \
        gd opcache bcmath intl mbstring pdo_mysql soap xsl zip sockets calendar ftp oauth openssl; \
    \
    curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer; \
    \
    # Install yarn globally and Mhsendmail (Go binary)
    npm install -g yarn; \
    go install github.com/mailhog/mhsendmail@latest; \
    \
    # Cleanup: Critical step for image size reduction
    apt-get purge -y --auto-remove golang-go; \
    apt-get clean; \
    rm -rf /var/lib/apt/lists/*; \
    # Remove the installer to save space since it's no longer needed
    rm -f /usr/local/bin/install-php-extensions

# --- Non-root user setup and Permissions (Combined into one layer) ---
# Create non-root user 'appuser' and set necessary permissions for CI
RUN set -eux; \
    useradd -m -d /home/appuser -s /bin/bash appuser; \
    mkdir -p /opt/atlassian/pipelines/agent/build; \
    # Optimize chown by being specific and using globbing where possible
    chown -R appuser:appuser \
        /opt/atlassian/pipelines/agent/build \
        /var/www \
        /usr/local/etc/php \
        /root \
        /home/appuser; \
    # Chown only the specific mhsendmail binary location
    chown -R appuser:appuser /go

# Configuration and binaries (Remaining ADD/COPY steps)
ADD .docker/config/php.ini /usr/local/etc/php/php.ini
COPY .docker/bin/* /usr/local/bin/
COPY .docker/users/* /var/www/
# Permissions for custom binaries
RUN chmod +x /usr/local/bin/*

# --- Final setup ---
# Set the user and working directory for the pipeline execution
USER appuser
WORKDIR /opt/atlassian/pipelines/agent/build
# Since Composer is installed in /usr/local/bin, this ENV is now optional but kept for vendor bin access
ENV PATH="/root/.composer/vendor/bin/:${PATH}"