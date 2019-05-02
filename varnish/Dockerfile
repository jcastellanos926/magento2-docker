FROM ubuntu:18.04

MAINTAINER jcastellanos926

# Update server, install supervisor and setup supervisord.conf file
RUN apt-get update && apt-get install -y supervisor
RUN mkdir -p /var/log/supervisor

# Install dependencies of varnish
RUN apt-get install -y \
    curl \
    gnupg \
    vim \
    apt-transport-https

# Install varnish
RUN curl -s https://packagecloud.io/install/repositories/varnishcache/varnish41/script.deb.sh | bash \
    && apt-get install -y varnish \
    && rm -rf /var/lib/apt/lists/* \
    && rm /etc/varnish/default.vcl \
    && rm /etc/default/varnish

EXPOSE 6082 80
CMD ["/usr/bin/supervisord"]