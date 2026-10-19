FROM nginx:latest@sha256:bc4cad49969aaf95eb4eecec5572846f86e59f1e8332aba36ed64b91aff8b724
RUN rm /etc/nginx/conf.d/default.conf
COPY data/index.html /usr/share/nginx/html/index.html
COPY data/acme-reverse-proxy.conf /etc/nginx/conf.d/
VOLUME /usr/share/nginx/html
VOLUME /etc/nginx
EXPOSE 8080