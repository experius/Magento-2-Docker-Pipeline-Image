FROM php:8.1-apache

MAINTAINER Experius <derrick@experius.nl>

# Install System Dependencies

RUN apt-get update \
	&& DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
	software-properties-common \
	&& apt-get update \
	&& DEBIAN_FRONTEND=noninteractive apt-get install -y \
	libfreetype6-dev \
	libicu-dev \
	libssl-dev \
	libjpeg62-turbo-dev \
	libmcrypt-dev \
	libedit-dev \
	libedit2 \
	libxslt1-dev \
	libzip-dev \
	libonig-dev \
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
	libjpeg-dev \
	&& apt-get clean

# Install Magento Dependencies

RUN docker-php-ext-configure \
    gd --with-jpeg --with-freetype; \
    docker-php-ext-install \
    opcache \
    gd \
    bcmath \
    intl \
    mbstring \
    pdo_mysql \
    soap \
    xsl \
    zip \
    sockets

# Install oAuth

RUN apt-get update \
    && apt-get install -y \
    libpcre3 \
    libpcre3-dev \
    # php-pear \
    && pecl install oauth \
    && echo "extension=oauth.so" > /usr/local/etc/php/conf.d/docker-php-ext-oauth.ini

# Install ext-sockets

RUN docker-php-ext-install sockets

# Install Node and NPM

RUN curl -sL https://deb.nodesource.com/setup_12.x | bash - \
    && apt-get install -y nodejs

# Install Composer

RUN	curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin/ --filename=composer --version=2.3.3

# Install Code Sniffer

RUN git clone https://github.com/magento/magento-coding-standard.git ~/.composer/vendor/magento/magento-coding-standard
RUN cd ~/.composer/vendor/magento/magento-coding-standard && composer install
RUN ln -s ~/.composer/vendor/magento/magento-coding-standard/vendor/bin/phpcs /usr/local/bin;

ENV PATH="/var/www/.composer/vendor/bin/:${PATH}"

# Install Mhsendmail

RUN DEBIAN_FRONTEND=noninteractive apt-get -y install golang-go \
   && mkdir /opt/go \
   && export GOPATH=/opt/go \
   && go get github.com/mailhog/mhsendmail


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
