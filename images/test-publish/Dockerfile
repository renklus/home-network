FROM nginx:latest@sha256:f5c017fb33c6db484545793ffb67db51cdd7daebee472104612f73a85063f889
RUN rm /etc/nginx/conf.d/default.conf
COPY data/index.html /usr/share/nginx/html/index.html
COPY data/acme-reverse-proxy.conf /etc/nginx/conf.d/
VOLUME /usr/share/nginx/html
VOLUME /etc/nginx
EXPOSE 8080