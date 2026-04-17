FROM "wardenenv/php-fpm:8.4-magento2"

USER root

EXPOSE 22
RUN /bin/sh -c dnf install sshd && dnf clean all && rm -rf /var/cache/dnf
CMD service ssh start && while true; do sleep 3000; done

USER www-data