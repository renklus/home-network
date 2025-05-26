FROM nginx:latest@sha256:fb39280b7b9eba5727c884a3c7810002e69e8f961cc373b89c92f14961d903a0
RUN rm /etc/nginx/conf.d/default.conf
COPY data/index.html /usr/share/nginx/html/index.html
COPY data/acme-reverse-proxy.conf /etc/nginx/conf.d/
VOLUME /usr/share/nginx/html
VOLUME /etc/nginx
EXPOSE 8080