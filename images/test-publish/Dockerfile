FROM nginx:latest@sha256:ca871a86d45a3ec6864dc45f014b11fe626145569ef0e74deaffc95a3b15b430
RUN rm /etc/nginx/conf.d/default.conf
COPY data/index.html /usr/share/nginx/html/index.html
COPY data/acme-reverse-proxy.conf /etc/nginx/conf.d/
VOLUME /usr/share/nginx/html
VOLUME /etc/nginx
EXPOSE 8080