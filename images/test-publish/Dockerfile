FROM nginx:latest@sha256:f547e3d0d5d02f7009737b284abc87d808e4252b42dceea361811e9fc606287f
RUN rm /etc/nginx/conf.d/default.conf
COPY data/index.html /usr/share/nginx/html/index.html
COPY data/acme-reverse-proxy.conf /etc/nginx/conf.d/
VOLUME /usr/share/nginx/html
VOLUME /etc/nginx
EXPOSE 8080