FROM nginx:latest@sha256:bc45d248c4e1d1709321de61566eb2b64d4f0e32765239d66573666be7f13349
RUN rm /etc/nginx/conf.d/default.conf
COPY data/index.html /usr/share/nginx/html/index.html
COPY data/acme-reverse-proxy.conf /etc/nginx/conf.d/
VOLUME /usr/share/nginx/html
VOLUME /etc/nginx
EXPOSE 8080