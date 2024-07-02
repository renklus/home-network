FROM nginx:latest@sha256:1aaa8180df68200fd41f9066cf62155e3b71183c04b2895a7388d5fd84ef3c8b
RUN rm /etc/nginx/conf.d/default.conf
COPY data/index.html /usr/share/nginx/html/index.html
COPY data/acme-reverse-proxy.conf /etc/nginx/conf.d/
VOLUME /usr/share/nginx/html
VOLUME /etc/nginx
EXPOSE 8080