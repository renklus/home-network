FROM nginx:latest@sha256:8632b3314d74c61f0e27f59055bab13f7793b988c94c090154d867bc29e0f39b
RUN rm /etc/nginx/conf.d/default.conf
COPY data/index.html /usr/share/nginx/html/index.html
COPY data/acme-reverse-proxy.conf /etc/nginx/conf.d/
VOLUME /usr/share/nginx/html
VOLUME /etc/nginx
EXPOSE 8080