FROM php:8.2-apache

RUN apt-get update && apt-get install -y --no-install-recommends \
        libcurl4-openssl-dev \
        libfreetype6-dev \
        libicu-dev \
        libjpeg-dev \
        libldap2-dev \
        libmcrypt-dev \
        libmemcached-dev \
        libpng-dev \
        libpq-dev \
        libxml2-dev \
        libzip-dev \
        libmagickwand-dev \
        libmagickcore-6.q16-3-extra \
        libsmbclient-dev \
        libgmp-dev \
        && rm -rf /var/lib/apt/lists/*

RUN debMultiarch="$(dpkg-architecture --query DEB_BUILD_MULTIARCH)"; \
    docker-php-ext-configure gd --with-freetype --with-jpeg; \
    docker-php-ext-configure ldap --with-libdir="lib/$debMultiarch"; \
    docker-php-ext-install \
        exif \
        gd \
        intl \
        ldap \
        opcache \
        pcntl \
        pdo_mysql \
        pdo_pgsql \
        zip \
        sysvsem \
        gmp

RUN pecl install APCu; \
    pecl install memcached; \
    pecl install redis; \
    pecl install xdebug; \
    pecl install imagick; \
    pecl install smbclient; \
    pecl install mcrypt; \
    \
    docker-php-ext-enable \
        apcu \
        memcached \
        redis \
        xdebug \
        imagick \
        smbclient; \
	docker-php-source delete && \
		rm -r /tmp/* /var/cache/*

# dev tools separate install so we quickly change without rebuilding all php extensions
RUN apt update && apt-get install -y --no-install-recommends \
    git curl vim sudo cron smbclient iproute2 lnav wget iputils-ping gnupg2 jq ripgrep rsync \
        && rm -rf /var/lib/apt/lists/*

# Install PHPUnit
RUN wget -O /usr/local/bin/phpunit8 https://phar.phpunit.de/phpunit-8.phar && \
    chmod +x /usr/local/bin/phpunit8 && \
    wget -O /usr/local/bin/phpunit9 https://phar.phpunit.de/phpunit-9.phar && \
    chmod +x /usr/local/bin/phpunit9

# Install NVM (Fermium = v14, Gallium = v16)
RUN curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/master/install.sh | bash \
    && export NVM_DIR="/root/.nvm" \
    && . "$NVM_DIR/nvm.sh" \
    && nvm install lts/gallium \
    && nvm install lts/fermium \
    && npm install -g npm@7
RUN wget https://gist.githubusercontent.com/nickvergessen/e21ee0a09ee3b3f7fd1b04c83dd3e114/raw/83142be1e50c23e8de1bd7aae88a95e5d6ae1ce2/nextcloud_log.json && lnav -i nextcloud_log.json && rm nextcloud_log.json

RUN { \
        echo '[global]'; \
        echo 'client min protocol = SMB2'; \
        echo 'client max protocol = SMB3'; \
        echo 'hide dot files = no'; \
} > /etc/samba/smb.conf

RUN mkdir --parent /var/log/cron
ADD configs/cron.conf /etc/nc-cron.conf
RUN crontab /etc/nc-cron.conf

ADD configs/php/nextcloud.ini /usr/local/etc/php/conf.d/nextcloud.ini
ADD configs/php/xdebug.ini /usr/local/etc/php/conf.d/xdebug.ini

# Setup blackfire probe
RUN wget -q -O - https://packages.blackfire.io/gpg.key | sudo apt-key add - \
    && echo "deb http://packages.blackfire.io/debian any main" | sudo tee /etc/apt/sources.list.d/blackfire.list \
    && apt-get update  \
    && (apt-get install -y --no-install-recommends blackfire-php blackfire \
		&& printf "blackfire.agent_socket=tcp://blackfire:8307\n" >> $PHP_INI_DIR/conf.d/zz-blackfire.ini.disabled) \
		|| echo "Skipped blackfire as the installation failed" \
    && rm -rf /var/lib/apt/lists/*

RUN echo "ServerName localhost" >> /etc/apache2/apache2.conf

RUN a2enmod rewrite

COPY --from=composer:latest /usr/bin/composer /usr/local/bin/composer

VOLUME /var/www/html
VOLUME /var/www/html/apps-writable
VOLUME /var/www/html/config
VOLUME /var/www/html/data

ENV SQL sqlite
ENV NEXTCLOUD_AUTOINSTALL YES
ENV WITH_REDIS NO

ENV WEBROOT /var/www/html
WORKDIR /var/www/html

ENTRYPOINT  ["/usr/local/bin/bootstrap.sh"]
CMD ["apache2-foreground"]

ADD data/installing.html /root/installing.html
ADD configs/autoconfig_mysql.php configs/autoconfig_pgsql.php configs/autoconfig_oci.php configs/s3.php configs/config.php configs/redis.config.php configs/apcu.config.php /root/
ADD bin/bootstrap.sh bin/occ /usr/local/bin/
