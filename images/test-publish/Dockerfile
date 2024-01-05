FROM nginx:latest
RUN rm /etc/nginx/conf.d/default.conf
COPY data/index.html /usr/share/nginx/html/index.html
COPY data/acme-reverse-proxy.conf /etc/nginx/conf.d/
VOLUME /usr/share/nginx/html
VOLUME /etc/nginx
EXPOSE 8080