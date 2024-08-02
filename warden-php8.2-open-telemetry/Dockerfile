FROM docker.io/wardenenv/php-fpm:8.2-magento2

USER root

RUN dnf install -y php-pecl-opentelemetry && dnf clean all && rm -rf /var/cache/dnf

USER www-data