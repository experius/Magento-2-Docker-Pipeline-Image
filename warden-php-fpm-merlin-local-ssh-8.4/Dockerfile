FROM "wardenenv/php-fpm:8.4-magento2"

USER root

EXPOSE 22
RUN /bin/sh -c dnf install sshd && dnf clean all && rm -rf /var/cache/dnf

USER www-data