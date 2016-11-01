FROM phusion/baseimage:0.9.19
MAINTAINER Miloš Kozák <milos.kozak@lejmr.com>

# Suporting software versions
ARG IREDMAIL_VERSION=0.9.5-1
ARG ADMINER_VERSION=4.2.5

# Default values changable at startup
ARG DOMAIN=DOMAIN
ARG HOSTNAME=HOSTNAME
ARG TIMEZONE=Europe/Prague
ARG SOGO_WORKERS=2

### Installation
# Prerequisites
ENV DEBIAN_FRONTEND noninteractive
RUN echo "APT::Install-Recommends 0;" >> /etc/apt/apt.conf.d/01-no-recommends \
    && echo "APT::Install-Suggests 0;" >> /etc/apt/apt.conf.d/01-no-recommends \
    && echo $TIMEZONE > /etc/timezone

RUN apt-get -q update \
    && apt-get install -y -q \
      apt-utils \
    && apt-get install -y -q \
      wget \
      bzip2 \
      iptables \
      openssl \
      mysql-server

RUN echo $DOMAIN > /etc/mailname \
    && echo $HOSTNAME > /opt/hostname \
    && mv /bin/uname /bin/uname_ \
    && mv /bin/hostname /bin/hostname_
    
COPY ./uname /bin/uname
COPY ./hostname /bin/hostname

# Install of iRedMail from sources
WORKDIR /opt/iredmail

RUN wget -O - https://bitbucket.org/zhb/iredmail/downloads/iRedMail-"${IREDMAIL_VERSION}".tar.bz2 | \
    tar xvj --strip-components=1

# Generate configuration file
COPY ./config-gen /opt/iredmail/config-gen
RUN sh ./config-gen $HOSTNAME $DOMAIN > ./config

# Initiate automatic installation process 
RUN sed s/$(hostname_)/$(cat /opt/hostname | xargs echo -n).$(cat /etc/mailname | xargs echo -n)/ /etc/hosts > /tmp/hosts_ \
    && cat /tmp/hosts_ > /etc/hosts \
    && rm /tmp/hosts_ \
    && echo $HOSTNAME > /etc/hostname \
    && sleep 5;service mysql start \
    && IREDMAIL_DEBUG='NO' \
      CHECK_NEW_IREDMAIL='NO' \
      AUTO_USE_EXISTING_CONFIG_FILE=y \
      AUTO_INSTALL_WITHOUT_CONFIRM=y \
      AUTO_CLEANUP_REMOVE_SENDMAIL=y \
      AUTO_CLEANUP_REMOVE_MOD_PYTHON=y \
      AUTO_CLEANUP_REPLACE_FIREWALL_RULES=n \
      AUTO_CLEANUP_RESTART_IPTABLES=n \
      AUTO_CLEANUP_REPLACE_MYSQL_CONFIG=y \
      AUTO_CLEANUP_RESTART_POSTFIX=n \
      bash iRedMail.sh

### Final configuration
RUN . ./config \
    && sed -i 's/PREFORK=.*$'/PREFORK=$SOGO_WORKERS/ /etc/default/sogo \
    && sed -i 's/WOWorkersCount.*$'/WOWorkersCount=$SOGO_WORKERS\;/ /etc/sogo/sogo.conf \
    && echo "[client]\nuser=root\npassword=${MYSQL_ROOT_PASSWD}" > /root/.my.cnf \
    && chmod 600 /root/.my.cnf \
    && sed -i '/^Foreground /c Foreground true' /etc/clamav/clamd.conf \
    && sed -i '/init.d/c pkill -sighup clamd' /etc/logrotate.d/clamav-daemon \
    && sed -i '/^Foreground /c Foreground true' /etc/clamav/freshclam.conf \
    && sed -i 's/^bind-address/#bind-address/' /etc/mysql/mysql.conf.d/mysqld.cnf

# Prepare for the first run
RUN tar jcf /root/mysql.tar.bz2 /var/lib/mysql && rm -rf /var/lib/mysql \
    && tar jcf /root/vmail.tar.bz2 /var/vmail && rm -rf /var/vmail \
    && tar jcf /root/clamav.tar.bz2 /var/lib/clamav && rm -rf /var/lib/clamav

# Install Adminer
RUN wget https://www.adminer.org/static/download/${ADMINER_VERSION}/adminer-${ADMINER_VERSION}.php -O /opt/www/roundcubemail/adminer.php

### Startup services
# Core Services
ADD services/mysql.sh /etc/service/1_mysql/run
ADD services/postfix.sh /etc/service/2_postfix/run
ADD services/amavis.sh /etc/service/3_amavis/run
ADD services/iredapd.sh /etc/service/2_iredapd/run
ADD services/dovecot.sh /etc/service/5_dovecot/run

# Frontend
ADD services/sogo.sh /etc/service/10_sogo/run
ADD services/iredadmin.sh /etc/service/10_iredadmin/run
ADD services/php7-fpm.sh /etc/service/10_php7-fpm/run
ADD services/nginx.sh /etc/service/10_httpd/run

# Enhancement
ADD services/fail2ban.sh /etc/service/30_fail2ban/run
ADD services/clamav-daemon.sh /etc/service/30_clamav-daemon/run
ADD services/clamav-freshclam.sh /etc/service/31_clamav-freshclam/run


### Purge some packets and save disk space
RUN apt-get purge -y -q dialog apt-utils augeas-tools \
    && apt-get autoremove -y -q \
    && apt-get clean -y -q \
    && rm -rf /var/lib/apt/lists/*

# Open Ports:
# Apache: 80/tcp, 443/tcp Postfix: 25/tcp, 587/tcp
# Dovecot: 110/tcp, 143/tcp, 993/tcp, 995/tcp
EXPOSE 80 443 25 587 110 143 993 995