FROM nginx:latest@sha256:6e23479198b998e5e25921dff8455837c7636a67111a04a635cf1bb363d199dc
RUN rm /etc/nginx/conf.d/default.conf
COPY data/index.html /usr/share/nginx/html/index.html
COPY data/acme-reverse-proxy.conf /etc/nginx/conf.d/
VOLUME /usr/share/nginx/html
VOLUME /etc/nginx
EXPOSE 8080