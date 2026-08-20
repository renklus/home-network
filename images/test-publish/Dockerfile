FROM nginx:latest@sha256:8f029c543423e3eac6b08254718bc31eb75633b1e448026b6616927baa7d4bfe
RUN rm /etc/nginx/conf.d/default.conf
COPY data/index.html /usr/share/nginx/html/index.html
COPY data/acme-reverse-proxy.conf /etc/nginx/conf.d/
VOLUME /usr/share/nginx/html
VOLUME /etc/nginx
EXPOSE 8080