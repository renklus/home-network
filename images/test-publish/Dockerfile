FROM nginx:latest@sha256:341bf0f3ce6c5277d6002cf6e1fb0319fa4252add24ab6a0e262e0056d313208
RUN rm /etc/nginx/conf.d/default.conf
COPY data/index.html /usr/share/nginx/html/index.html
COPY data/acme-reverse-proxy.conf /etc/nginx/conf.d/
VOLUME /usr/share/nginx/html
VOLUME /etc/nginx
EXPOSE 8080