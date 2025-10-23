FROM nginx:latest@sha256:029d4461bd98f124e531380505ceea2072418fdf28752aa73b7b273ba3048903
RUN rm /etc/nginx/conf.d/default.conf
COPY data/index.html /usr/share/nginx/html/index.html
COPY data/acme-reverse-proxy.conf /etc/nginx/conf.d/
VOLUME /usr/share/nginx/html
VOLUME /etc/nginx
EXPOSE 8080