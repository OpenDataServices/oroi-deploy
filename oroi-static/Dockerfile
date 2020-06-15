# Do this instead of using the httpd image,
# so we get the debian style conf.d we're used to.
FROM ubuntu:20.04
ENV DEBIAN_FRONTEND noninteractive
RUN apt-get update && \
    apt-get -y upgrade
RUN apt-get install -y apache2
RUN echo "ErrorLog /dev/stderr" >> /etc/apache2/apache2.conf
RUN a2enmod proxy proxy_http
COPY oroi-static.conf /etc/apache2/sites-available/000-default.conf
EXPOSE 80
CMD /usr/sbin/apache2ctl -D FOREGROUND
