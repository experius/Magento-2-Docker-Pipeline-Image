FROM php:8.4-apache

# Install php extension dependencies
ADD --chmod=0755 https://github.com/mlocati/docker-php-extension-installer/releases/latest/download/install-php-extensions /usr/local/bin/

#MAINTAINER Experius <derrick@experius.nl>
LABEL org.opencontainers.image.authors="matthijsbreed@happyhorizon.com"

RUN apt-get update \
	&& DEBIAN_FRONTEND=noninteractive apt-get install -y \
	zip \
	apt-utils \
	gnupg \
	redis-tools \
	default-mysql-client \
	git \
	vim \
	wget \
	curl \
	lynx \
	psmisc \
	unzip \
	tar \
	cron \
	bash-completion \
	rsync \
	&& apt-get clean

RUN install-php-extensions \
    gd --with-jpeg --with-freetype; \
    install-php-extensions \
    opcache \
    gd \
    bcmath \
    intl \
    mbstring \
    pdo_mysql \
    soap \
    xsl \
    zip \
    sockets \
    calendar \
    ftp \
    oauth \
    sockets \
    openssl

# Install Node and NPM

ADD .docker/config/preferences /etc/apt/preferences
RUN curl -sL https://deb.nodesource.com/setup_18.x | bash - \
    && apt-get install -y nodejs

# Install Composer

RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin/ --filename=composer --version=2.8.11

# Install Code Sniffer

RUN git clone https://github.com/magento/magento-coding-standard.git ~/.composer/vendor/magento/magento-coding-standard
RUN cd ~/.composer/vendor/magento/magento-coding-standard && composer install
RUN ln -s ~/.composer/vendor/magento/magento-coding-standard/vendor/bin/phpcs /usr/local/bin;

ENV PATH="/var/www/.composer/vendor/bin/:${PATH}"

# Install Mhsendmail

RUN DEBIAN_FRONTEND=noninteractive apt-get -y install golang-go \
   && mkdir /opt/go \
   && export GOPATH=/opt/go \
   && go install github.com/mailhog/mhsendmail@latest


# Configuring system

ADD .docker/config/php.ini /usr/local/etc/php/php.ini
ADD .docker/config/magento.conf /etc/apache2/sites-available/magento.conf
ADD .docker/config/custom-xdebug.ini /usr/local/etc/php/conf.d/custom-xdebug.ini
COPY .docker/bin/* /usr/local/bin/
COPY .docker/users/* /var/www/
RUN chmod +x /usr/local/bin/*
RUN ln -s /etc/apache2/sites-available/magento.conf /etc/apache2/sites-enabled/magento.conf
RUN rm /etc/apache2/sites-enabled/000-default.conf
RUN rm /etc/apache2/sites-available/default-ssl.conf
RUN service apache2 restart

RUN chmod 777 -Rf /var/www /var/www/.* \
    && chown -Rf www-data:www-data /var/www /var/www/.* \
    && usermod -u 1000 www-data \
    && chsh -s /bin/bash www-data\
    && a2enmod rewrite \
    && a2enmod headers

VOLUME /var/www/html
WORKDIR /var/www/html
