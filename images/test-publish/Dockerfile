FROM nginx:latest@sha256:5473310a448fa874f3c944e0f98a79cc6c20ed39723141268bf91cc261dc48c4
RUN rm /etc/nginx/conf.d/default.conf
COPY data/index.html /usr/share/nginx/html/index.html
COPY data/acme-reverse-proxy.conf /etc/nginx/conf.d/
VOLUME /usr/share/nginx/html
VOLUME /etc/nginx
EXPOSE 8080