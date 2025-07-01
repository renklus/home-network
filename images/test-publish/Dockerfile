FROM nginx:latest@sha256:93230cd54060f497430c7a120e2347894846a81b6a5dd2110f7362c5423b4abc
RUN rm /etc/nginx/conf.d/default.conf
COPY data/index.html /usr/share/nginx/html/index.html
COPY data/acme-reverse-proxy.conf /etc/nginx/conf.d/
VOLUME /usr/share/nginx/html
VOLUME /etc/nginx
EXPOSE 8080