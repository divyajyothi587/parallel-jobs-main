FROM ubuntu:20.04
ARG DEBIAN_FRONTEND=noninteractive
RUN apt-get update 
RUN apt-get install apache2 apache2-utils -y
ADD index.html /var/www/html/index.html
RUN apt-get clean
EXPOSE 80
CMD ["apache2ctl", "-D", "FOREGROUND"]