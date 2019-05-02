FROM php:7.2-apache

MAINTAINER jcastellanos926

RUN usermod -a -G root www-data
RUN usermod -u 1000 www-data \
	&& chsh -s /bin/bash www-data

# Composer executable
RUN curl -sS https://getcomposer.org/installer | php

# Made composer cli global
RUN mv composer.phar /usr/local/bin/composer

# User & password of magento 2 api account (https://devdocs.magento.com/guides/v2.0/install-gde/prereq/connect-auth.html)
COPY ./auth.json  /root/.composer/

# Install the required extensions of php and dependencies
RUN requirements="libpng-dev libmcrypt-dev libmcrypt4 libcurl3-dev libfreetype6 libjpeg62-turbo-dev libjpeg62-turbo libfreetype6-dev libicu-dev libxslt1-dev" \
    && apt-get update && apt-get install -y $requirements \
    && docker-php-ext-install pdo_mysql \
    && docker-php-ext-configure gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/ \
    && docker-php-ext-install gd \
    && docker-php-ext-install mbstring \
    && docker-php-ext-install zip \
    && docker-php-ext-install intl \
    && docker-php-ext-install xsl \
    && docker-php-ext-install soap \
    && docker-php-ext-install bcmath \
    && docker-php-ext-configure opcache --enable-opcache \
    && docker-php-ext-install opcache

# Dev utils
RUN apt-get update && apt-get install -y \
    apt-utils \
    sudo \
    wget \
    vim \
    unzip \
    telnet \
    msmtp-mta \
    rsyslog --reinstall

# Install mcrypt from pecl
RUN cd /tmp && wget -c "http://pecl.php.net/get/mcrypt-1.0.1.tgz" --no-check-certificate \
    && tar -xf mcrypt-1.0.1.tgz \
    && cd mcrypt-1.0.1 \
    && phpize \
    && bash ./configure \
    && make \
    && make install
	
# Writting logs service
RUN service rsyslog restart

# Msmtp mail configuration
COPY ./msmtprc  /etc/
RUN chmod 600 /etc/msmtprc \
    && chown www-data:www-data /etc/msmtprc

# Download cert file
RUN wget https://curl.haxx.se/ca/cacert.pem --no-check-certificate \
    && mv cacert.pem /etc/ssl/certs/

# xdebug for PHPStorm
RUN cd /tmp && wget -c "https://xdebug.org/files/xdebug-2.6.1.tgz" --no-check-certificate \
    && tar -xf xdebug-2.6.1.tgz \
    && cd xdebug-2.6.1 \
    && phpize \
    && bash ./configure \
    && make \
    && make install \
    && echo "xdebug.remote_enable=1" >> /usr/local/etc/php/conf.d/xdebug.ini \
    && echo "xdebug.remote_port=9000" >> /usr/local/etc/php/conf.d/xdebug.ini \
    && echo "xdebug.remote_connect_back=0" >> /usr/local/etc/php/conf.d/xdebug.ini \
    && echo "host.docker.internal=$REMOTE_IP" >> /usr/local/etc/php/conf.d/xdebug.ini \
    && echo "xdebug.idekey=PHPSTORM" >> /usr/local/etc/php/conf.d/xdebug.ini \
    && echo "xdebug.max_nesting_level=1000" >> /usr/local/etc/php/conf.d/xdebug.ini \
    && echo "zend_extension=xdebug.so" >> /usr/local/etc/php/conf.d/xdebug.ini

# Disable default xdebug configuration
RUN mv /usr/local/etc/php/conf.d/xdebug.ini /usr/local/etc/php/conf.d/xdebug.ini.copy

COPY apache-conf/smtp.ini /usr/local/etc/php/conf.d/
COPY apache-conf/opcache.ini /usr/local/etc/php/conf.d/
COPY apache-conf/local-m2_php.ini /usr/local/etc/php/conf.d/
COPY apache-conf/local-m2.conf /etc/apache2/sites-available/

# Listen port 80
RUN sed -i "s/None/all/g" /etc/apache2/apache2.conf \
    && sed -i "s/80/80/g" /etc/apache2/ports.conf

# Add .bash_aliases file support
RUN echo "" >> /root/.bashrc \
    && echo "# Alias definitions." >> /root/.bashrc \
    && echo "# You may want to put all your additions into a separate file like" >> /root/.bashrc \
    && echo "# ~/.bash_aliases, instead of adding them here directly." >> /root/.bashrc \
    && echo "# See /usr/share/doc/bash-doc/examples in the bash-doc package." >> /root/.bashrc \
    && echo "if [ -f ~/.bash_aliases ]; then" >> /root/.bashrc \
    && echo "    . ~/.bash_aliases" >> /root/.bashrc \
    && echo "fi" >> /root/.bashrc

COPY .bash_aliases /root/

# Copy executables to container
COPY ./bin /usr/local/bin/magento/
RUN ln -s /usr/local/bin/magento/* /usr/local/bin/

# Modify apache properties
RUN a2enmod rewrite headers proxy_http
RUN a2dissite 000-default
RUN a2ensite local-m2

WORKDIR /var/www/html/src

EXPOSE 80 9000 8080
