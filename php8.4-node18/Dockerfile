# Dockerfile.ci: Optimized for Bitbucket Pipelines (Multi-Stage Refactor)

# ==============================================================================
# STAGE 1: MHSEMAIL BUILDER
# Builds the Go binary in a minimal Go image and then discards the toolchain.
# ==============================================================================
FROM golang:1.21-alpine AS mhsendmail_builder

# Install the mailhog sendmail binary
RUN go install github.com/mailhog/mhsendmail@latest

# ==============================================================================
# STAGE 2: CI RUNNER IMAGE
# ==============================================================================
FROM php:8.4-cli AS ci_runner

LABEL org.opencontainers.image.authors="dylanmaurits@happyhorizon.com" \
      org.opencontainers.image.title="php8.4-node18-ci" \
      org.opencontainers.image.description="CI runner image for PHP 8.4 with Node 18, Composer 2, and Magento Coding Standard" \
      org.opencontainers.image.source="https://github.com/happyhorizon/Magento-2-Docker-Pipeline-Image" \
      org.opencontainers.image.licenses="MIT" \
      org.opencontainers.image.created="2025-11-14"

# Set non-interactive mode and essential paths (GOPATH is no longer needed)
ENV DEBIAN_FRONTEND=noninteractive
ENV PATH="/usr/local/bin:${PATH}"

# 1. Install Dependencies, PHP Extensions, Node.js, and Cleanup in ONE Layer
RUN set -eux; \
    # Fetch and install the PHP extension installer
    curl -fsSL https://github.com/mlocati/docker-php-extension-installer/releases/latest/download/install-php-extensions -o /usr/local/bin/install-php-extensions; \
    chmod +x /usr/local/bin/install-php-extensions; \
    \
    # Install Node.js 18 and necessary system dependencies (Combined APT steps)
    curl -fsSL https://deb.nodesource.com/setup_18.x | bash -; \
    apt-get update; \
    apt-get install -y --no-install-recommends \
        # Essential/Common
        zip apt-utils gnupg wget curl unzip bash-completion rsync procps \
        # PHP related libs
        libzip-dev openssh-client redis-tools default-mysql-client \
        git nodejs; \
    \
    # Install PHP extensions
    install-php-extensions \
        gd opcache bcmath intl mbstring pdo_mysql soap xsl zip sockets calendar ftp oauth openssl; \
    \
    curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer; \
    \
    # Install yarn globally
    npm install -g yarn; \
    \
    # Cleanup: Critical step for image size reduction
    apt-get clean; \
    rm -rf /var/lib/apt/lists/*; \
    # Remove the installer to save space since it's no longer needed
    rm -f /usr/local/bin/install-php-extensions

# --- Copy the securely compiled Go binary from Stage 1 ---
COPY --from=mhsendmail_builder /go/bin/mhsendmail /usr/local/bin/mhsendmail

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
    # Chown the specific mhsendmail binary location
    chown appuser:appuser /usr/local/bin/mhsendmail

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