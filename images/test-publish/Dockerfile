FROM nginx:latest@sha256:7272239bd21472f311aa3e86a85fdca0f1ad648995f983ab6e5e7dea665cd233
RUN rm /etc/nginx/conf.d/default.conf
COPY data/index.html /usr/share/nginx/html/index.html
COPY data/acme-reverse-proxy.conf /etc/nginx/conf.d/
VOLUME /usr/share/nginx/html
VOLUME /etc/nginx
EXPOSE 8080